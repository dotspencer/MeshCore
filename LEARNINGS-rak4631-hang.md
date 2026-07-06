# RAK4631 Companion Node Hangs & Contact Wipes — Investigation Learnings

Investigation of RAK 4631 companion nodes (firmware v1.16.0, iOS app) that become
unresponsive after days of uptime, sometimes losing all contacts after a battery-pull
recovery.

## Symptoms observed

Two distinct failure modes seen in the field (not one bug):

| # | Blue LED | Green LED | BLE advertising | USB enumeration | Interpretation |
|---|----------|-----------|-----------------|-----------------|----------------|
| 1 | Solid    | —         | Stopped         | (not tested)    | BLE stack alive but wedged holding a phantom connection |
| 2 | Off      | Solid     | None            | None (2 cables) | Full chip lockup — SoftDevice, FreeRTOS, TinyUSB all dead |

Contact wipe: after pulling the battery on a wedged node, it rebooted with all
contacts cleared, as if factory reset.

## Failure mode 1: BLE pairing stall (solid blue LED)

- Matches upstream [issue #2213](https://github.com/meshcore-dev/MeshCore/issues/2213)
  (RAK4631 + iOS, 1.14.0, still open/stale).
- Root cause identified in [PR #2089](https://github.com/meshcore-dev/MeshCore/pull/2089):
  commit `25ea953c` ("don't mark as connected until connection secured") means a
  central that connects at the GAP level but never completes encryption — reportedly
  more common with iPhones — occupies the single peripheral connection slot forever.
  - `_isDeviceConnected` stays false until `onSecured()`, and nothing times out an
    unsecured connection (`src/helpers/nrf52/SerialBLEInterface.cpp`).
  - The advertising watchdog in `checkRecvFrame()` explicitly skips while
    `_conn_handle` is valid, so it cannot recover this state.
  - Result: solid blue LED (Bluefruit conn LED), advertising never resumes, nobody
    can connect until power cycle.
- **PR #2089 fixes this** with a 15 s `BLE_SECURITY_TIMEOUT_MS`: force-disconnect a
  connection that hasn't secured, then immediately re-check advertising. Review
  feedback (double-disconnect guard, immediate advertising recheck) was addressed;
  the author tested 3 weeks stable on a RAK4631 and commented "Fixes #2213".
- **The PR was closed by its author (Apr 2026) without being merged** — no stated
  reason, no superseding commit. v1.16.0 contains the regression but not the fix.
- The diff still applies cleanly to current `main` and builds successfully
  (branch `pr2089-ble-security-timeout`, `RAK_4631_companion_radio_ble` target;
  RAM 62.3%, flash 72.9%).

## Failure mode 2: full chip lockup (solid green LED, everything dead)

- Observed live on `MeshCore-sm-window-1`: no BLE advertising (other nodes visible
  in the same scan), no USB enumeration on two known cables, blue LED off.
- The application firmware never drives the green LED on this board
  (`PIN_STATUS_LED` is not defined for RAK4631, so `UITask::userLedHandler()` is a
  no-op; the app only uses the blue conn LED via Bluefruit).
- The Adafruit core's `HardFault_Handler` is `NVIC_SystemReset()` — a clean hard
  fault reboots. Ending up frozen instead implies a lockup the fault handler can't
  run from (interrupts masked, fault-in-fault, latched bus state).
- **No hardware watchdog is configured anywhere in the firmware.**
  `NRF52Board.cpp` can *report* a watchdog reset reason but nothing ever starts or
  feeds the nRF52 WDT. A WDT fires regardless of CPU state and is the only
  self-recovery possible for this failure mode.
- Red LED when USB plugged in is just the base board's hardware charge indicator —
  not firmware-controlled, not diagnostic.

## Contact wipe mechanism

On the RAK4631 BLE companion build, contacts/channels/advert blobs live on a
dedicated 100 KB LittleFS partition in internal flash (`EXTRAFS` → CustomLFS at
`0xD4000`, `examples/companion_radio/main.cpp`).

Three compounding hazards:

1. **Auto-format on mount failure.** Both `InternalFS.begin()` and
   `CustomLFS::begin()` erase the entire flash region and reformat if LittleFS
   fails to mount. Any corruption ⇒ silent total wipe at next boot. Contacts and
   channels vanish while identity/prefs (on the other partition) survive — exactly
   "contacts cleared as if reset".
2. **Delete-then-rewrite saves.** `DataStore::openWrite()` on nRF52 does
   `fs->remove(filename)` then rewrites from scratch. Power loss mid-save loses the
   file even without LittleFS-level corruption. With `MAX_CONTACTS=350` a full save
   is up to ~50 KB.
3. **Frequent writes.** Every advert received from a known contact writes the raw
   advert blob to flash (`BaseChatMesh.cpp` `putBlobByKey`) *and* schedules a full
   contacts-file rewrite 5 s later (`LAZY_CONTACTS_WRITE_DELAY`). On a busy mesh
   this happens many times per hour, so a battery pull has a high chance of landing
   mid-write.

The hang and the wipe compound each other: hangs force battery pulls, and battery
pulls are what corrupt the filesystem.

Key experiment: recover a wedged node with a **single reset press** instead of a
battery pull. A clean reset cannot interrupt a flash write, so if contacts are
wiped even then, the filesystem was already corrupted before the reset.

## Diagnostics cheat-sheet for a wedged node (before power-cycling!)

1. Plug into a Mac/PC over a known **data** cable:
   - `RAK4631` mass-storage drive mounts ⇒ sitting in UF2 bootloader.
   - Serial port enumerates ⇒ RTOS alive; open at 115200 (BLE build has
     `BLE_DEBUG_LOGGING=1`) and watch while attempting an app connection.
   - Nothing enumerates ⇒ full lockup (or SYSTEMOFF).
2. BLE scan (nRF Connect / LightBlue / `blew scan`): advertising as `MeshCore-*`
   ⇒ app stack alive; silent ⇒ SoftDevice down (or holding a connection — check
   blue LED).
3. Recovery order: single reset press → double-press (bootloader) → battery pull
   (last resort; risks contact wipe).

## Fixes

| Fix | Status | Addresses |
|-----|--------|-----------|
| Apply PR #2089 (BLE security timeout) | Built on branch `pr2089-ble-security-timeout`; field-tested upstream | Failure mode 1 |
| Enable + feed nRF52 hardware watchdog | Not implemented | Failure mode 2 (auto-reboot from any lockup) |
| Atomic contact saves (temp file + rename) | Not implemented | Contact wipe hazard 2 |
| Reduce flash write frequency / debounce advert-triggered saves | Not implemented (upstream reportedly improved this later) | Contact wipe hazard 3 |
| Comment on PR #2089 / issue #2213 asking to reopen | Drafted | Getting the fix upstream |

## Resolution (2026-07-06, `sm-window-1`)

The wedged node turned out to be in a **boot crash-loop caused by corrupted
filesystem data**, proven by elimination:

1. Bootloader mounted fine (hardware good), full flash backed up via `CURRENT.UF2`.
2. Reflashing the app — both the patched build *and* official 1.16.0 — did not fix
   it: still no USB, no BLE. USB comes up before `setup()`, so only a reset loop in
   the first moments of boot fits.
3. Erasing the filesystem regions (MeshCore web flasher's "erase all user data"
   utility) fixed it instantly — the same firmware images then booted fine.

Notably the ExtraFS superblock in the backup *looked* valid and contact data was
visible, so the corruption was in deeper metadata — consistent with
`LFS_NO_ASSERT=1` letting corrupt values propagate to a hard fault
(`HardFault_Handler` → `NVIC_SystemReset` → loop).

This also retroactively explains the historical contact wipes: same corruption
class, milder outcome (mount fails cleanly → auto-format) vs. this incident
(mount crashes → boot loop).

The PR #2089 patch was then verified live: an unpaired BLE connection from a Mac
(bleak, no pairing) was force-disconnected by the node after ~13.6 s and
advertising resumed immediately. Stock firmware holds such a connection forever.

## Build notes

- PlatformIO (installed in a venv): `pio run -e RAK_4631_companion_radio_ble`
- UF2 for drag-and-drop flashing: add `-t create_uf2` →
  `.pio/build/RAK_4631_companion_radio_ble/firmware.uf2`
- `firmware.zip` (same dir) works for OTA DFU via nRF Connect.
- GitHub serves any PR as a raw patch at `<pr-url>.diff`; verify against a checkout
  with `git apply --check`.

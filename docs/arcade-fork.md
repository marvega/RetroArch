# arcade fork of RetroArch

Branch `arcade` of https://github.com/marvega/RetroArch carries five small patches on top of
upstream master `29816dd948b4bd6ef34960148330b16573a85364` (2026-10-07) for the arcadedash
Nintendo Switch launcher (online arcade multiplayer with FBNeo). One commit per patch; to move to
a newer upstream: `git fetch upstream && git rebase upstream/master`.

| # | Option / behaviour | Files |
|---|---|---|
| 1 | `--netplay-password=PW`: answers the host's password challenge from the command line instead of the on-screen keyboard. Hashed exactly like the dialog path (`"%08lX"` salt + password, SHA-256) and sent right after NICK, which is the order the host reads them. Unset: dialog as before. | `retroarch.c`, `network/netplay/netplay.h`, `netplay_defines.h`, `netplay_private.h`, `netplay_frontend.c` |
| 2 | `--return-to=PATH` (libnx only): on the final exit (`frontend_switch_exitspawn` with `FRONTEND_FORK_NONE`, where hbmenu would come back) calls `envSetNextLoad(PATH, PATH)`. When the menu NRO forks into a core NRO the option is appended to the fork arguments as one quoted token so the core returns to the launcher too. | `retroarch.c`, `frontend/drivers/platform_switch.c` |
| 3 | Client connect retry: when `netplay_new()` fails for a client (TCP connect refused or host unresolved) a `retro_task` scheduled 2 s later re-runs `CMD_EVENT_NETPLAY_INIT_DIRECT` against the same `address\|port\|session`, up to 5 times, showing "Retrying netplay connection (n/5)...". The last failure keeps the usual "Failed to initialize netplay." Handshake and password failures are not retried; hosting is unaffected. | `network/netplay/netplay.h`, `netplay_frontend.c`, `msg_hash.h`, `intl/msg_hash_us.h` |
| 4 | `netplay_relay_fallback = "true"` (config only, default false): a host in direct mode that the internet cannot reach (lobby announce reply `connectable=0`, UPnP failed, or UPnP external address private/CGNAT) closes the direct session and hosts again through `netplay_custom_mitm_server` (handle `custom`, whatever `netplay_mitm_server` says), showing "No direct connection from the internet. Hosting through the relay server." Once per hosting session, only while no client is connected or handshaking, only with a custom relay set; runs from a main-thread task, not from the HTTP/NAT callbacks. The direct room stops announcing and the relay room is announced at once. If the relay cannot be reached it hosts directly again. Settings are not modified. | `configuration.c/.h`, `network/netplay/netplay.h`, `netplay_frontend.c`, `msg_hash.h`, `intl/msg_hash_us.h`, `tools/settings_table.reference` |
| 5 | libnx exit: `socketExit()` moves from `frontend_switch_deinit()` to `frontend_switch_shutdown()`, which `main_exit()` calls after `net_http_deinit()`. A host that announced to the lobby keeps a pooled HTTP connection; closing it after `socketExit()` was a data abort in `_close_r` (crash on quit, no `--return-to`). | `frontend/drivers/platform_switch.c` |

## Launching a core NRO with these options

hbloader (libnx `argv.c`) splits on whitespace and honours a `"` only when it starts a token, so
values that may contain spaces are passed as whole quoted tokens. `argv[0]` must be the core NRO
path (it becomes `RARCH_PATH_CORE`):

```
/retroarch/cores/fbneo_libretro_libnx.nro "/retroarch/downloads/dino.zip" -C 192.168.1.39 --port 55435 --nick Switch "--netplay-password=1234" "--return-to=/switch/arcadedash.nro"
```

## Building for Switch (same steps as the libretro buildbot, `dist-scripts/dist-cores.sh`)

```
make -f Makefile.libnx HAVE_STATIC_DUMMY=1 -j                      # menu: retroarch_switch.nro
make -C FBNeo/src/burner/libretro platform=libnx -j                # core archive
cp FBNeo/src/burner/libretro/fbneo_libretro_libnx.a libretro_libnx.a
make -f Makefile.libnx clean && make -f Makefile.libnx APP_TITLE=fbneo LIBRETRO=fbneo -j
mv retroarch_switch.nro fbneo_libretro_libnx.nro
```

Scripted with Docker `devkitpro/devkita64` in the arcadedash repo (`tools/retroarch-build/build.sh`).

## Notes

- Message files other than `intl/msg_hash_us.h` are generated from Crowdin, so the retry
  notification is English until it is translated upstream.
- Patches 1, 3 and 4 are platform-independent and candidates for upstream; patch 2 is Switch-specific.
- Patch 4 and the lobby (`libretro/netplay-lobby-server-go`): a room is keyed on nick + IP + port and an update within
  5 s of the last one is rejected, so the relay room is announced on host setup's usual 5 s delay (a relay on the same
  port as the direct room makes it an update of the same room). Rooms not refreshed for 60 s are dropped.
- Patch 4 timing (2026-10-11, Starlink CGNAT, relay 146.181.42.30:55435): lobby verdict 0-3.4 s after the first
  announce (+5 s), relay room listed ~5.4 s after that.
- Switch: `netplay_nat_traversal = true` hangs RetroArch before the first frame (black screen, quit combo dead, no
  announce); keep it false on libnx and rely on the lobby's `connectable` verdict. RetroArch must run with full RAM
  (title override): in applet mode EGL fails and `gl2_init` calls a NULL GL pointer.

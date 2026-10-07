# arcade fork of RetroArch

Branch `arcade` of https://github.com/marvega/RetroArch carries three small patches on top of
upstream master `29816dd948b4bd6ef34960148330b16573a85364` (2026-10-07) for the arcadedash
Nintendo Switch launcher (online arcade multiplayer with FBNeo). One commit per patch; to move to
a newer upstream: `git fetch upstream && git rebase upstream/master`.

| # | Option / behaviour | Files |
|---|---|---|
| 1 | `--netplay-password=PW`: answers the host's password challenge from the command line instead of the on-screen keyboard. Hashed exactly like the dialog path (`"%08lX"` salt + password, SHA-256) and sent right after NICK, which is the order the host reads them. Unset: dialog as before. | `retroarch.c`, `network/netplay/netplay.h`, `netplay_defines.h`, `netplay_private.h`, `netplay_frontend.c` |
| 2 | `--return-to=PATH` (libnx only): on the final exit (`frontend_switch_exitspawn` with `FRONTEND_FORK_NONE`, where hbmenu would come back) calls `envSetNextLoad(PATH, PATH)`. When the menu NRO forks into a core NRO the option is appended to the fork arguments as one quoted token so the core returns to the launcher too. | `retroarch.c`, `frontend/drivers/platform_switch.c` |
| 3 | Client connect retry: when `netplay_new()` fails for a client (TCP connect refused or host unresolved) a `retro_task` scheduled 2 s later re-runs `CMD_EVENT_NETPLAY_INIT_DIRECT` against the same `address\|port\|session`, up to 5 times, showing "Retrying netplay connection (n/5)...". The last failure keeps the usual "Failed to initialize netplay." Handshake and password failures are not retried; hosting is unaffected. | `network/netplay/netplay.h`, `netplay_frontend.c`, `msg_hash.h`, `intl/msg_hash_us.h` |

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
- Patches 1 and 3 are platform-independent and candidates for upstream; patch 2 is Switch-specific.

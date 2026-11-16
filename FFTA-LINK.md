# FFTA link-battle build (unofficial mGBA fork)

This branch is mGBA (commit 143adf5, October 2026 development build) with one change that lets
**Final Fantasy Tactics Advance link battles** (co-op link missions) work between two linked mGBA windows.
In standard mGBA they end in "Link Broken" as soon as a unit acts.

**This is not an official mGBA build.** It is not made, reviewed or supported by the mGBA team. Please do not
report problems with it to the mGBA project; open an issue on this fork instead. The change was written with AI
assistance (Claude, by Anthropic) and checked by testing; it is not offered upstream, because mGBA does not accept
AI-generated code. The cause is documented in mGBA issue #3056 (https://github.com/mgba-emu/mgba/issues/3056).

## What changed

In a multiplayer link, FFTA's battles switch the serial port to General Purpose mode and run a keep-alive that
toggles the link pins and watches the partner's. Standard mGBA does not pass those pin levels between linked
players, so the partner looks dead and the game reports "Link Broken". This branch shares them while every linked
player is in General Purpose mode: SC and SD act as shared lines (any player driving low pulls them low, otherwise
high), each player's SI reads the previous player's SO, and a player's own output pins read back what it wrote.
Nothing else changes (normal multiplayer transfers, timing, single-player play).

Second change (keep-alive timing): mGBA lets one linked window run up to about two frames ahead before the other
catches up. FFTA's keep-alive flips a pin once per frame and checks the partner once per frame, so reading the
partner's latest pins often showed an even number of flips since the last check, i.e. no change; runs of 10 to 16
"unchanged" checks were normal and anything that lengthened them reached the game's limit of 70 ("Link Broken").
Each pin change is now queued for the other players and taken in one per write of the link register, so every
flip is seen once whatever the thread timing. A partner that really stops still reads as stopped. Reads of the
register (including a debugger's or a script's) have no side effects.

Files: `include/mgba/gba/interface.h`, `include/mgba/internal/gba/sio/lockstep.h`, `src/gba/io.c`,
`src/gba/sio/lockstep.c`. Licence: MPL-2.0, as the rest of mGBA.

Tested: FFTA (USA), two linked windows on one Windows PC: co-op link missions played through; with the second
change, ten switches into and out of General Purpose mode in one battle with the keep-alive's "unchanged" count
staying at 0 (it reached 16 before). Not tested with other games that use General Purpose mode, or with network
play.

Known issue: with a Lua script that pauses the game often (for example a breakpoint-based recorder), the switch
from General Purpose mode back to normal multiplayer can occasionally fail to reconnect, ending in "Link Broken".
This has not been seen without such scripts.

## Playing

1. Start mGBA, load the game, then File > New multiplayer window and load it in the second window.
   Give each window its own save (simplest: two copies of the ROM in separate folders, each with its .sav).
2. On both, open the link menu and start the co-op mission.

## Building on Windows (MSYS2)

1. Install MSYS2 from https://www.msys2.org, then open **MSYS2 MINGW64** from the Start menu (not UCRT64 or MSYS).
2. Run `pacman -Syu`, reopening MINGW64 and repeating until it says there is nothing to do.
3. Then (paste with Shift+Insert, one line at a time):

```
pacman -S --needed --noconfirm git mingw-w64-x86_64-{gcc,cmake,ninja,pkgconf,qt5-base,qt5-multimedia,qt5-tools,SDL2,lua54,libzip,libepoxy,libelf,sqlite3,libpng,zlib,freetype}
git clone -b ffta-link-battles https://github.com/tatelee1990-ffta/mgba.git
cd mgba
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_QT=ON -DBUILD_SDL=OFF -DUSE_FFMPEG=OFF -DUSE_DISCORD_RPC=OFF -DUSE_LUA=5.4 -DM_CORE_GB=OFF -DUSE_MINIZIP=OFF -DBUILD_SHARED=OFF -DBUILD_STATIC=ON
ninja -C build
```

4. Start it from the MINGW64 window with `./build/mGBA.exe` (it uses MSYS2's DLLs).

Why these settings: MSYS2's `lua` package is now Lua 5.5, which this mGBA does not use (`lua54` provides 5.4);
MSYS2's minizip package names a file it does not ship (`USE_MINIZIP=OFF`; libzip is used instead); and a separate
`libmgba.dll` stops the exe from starting with "32 bit pseudo relocation out of range" (`BUILD_STATIC=ON`,
`BUILD_SHARED=OFF` builds the core into the exe).

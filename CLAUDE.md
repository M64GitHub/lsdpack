# lsdpack (M64GitHub fork) — notes for Claude

Fork of https://github.com/jkotlinski/lsdpack (GPL-2.0-or-later, by Johan Kotlinski).
Records LSDj songs by running LSDj in an embedded Gambatte emulator and logging
every sound register write; `player.s` replays the log from another ROM.

## Build and run

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build
    ./build/lsdpack -d -t lsdj.gb        # LSDj 9.x; writes lsdj.s to the current dir

- Needs an LSDj ROM + .sav (same basename). Never commit them (`*.gb`, `*.sav` are ignored).
- Always run on COPIES: Gambatte writes `<rom>.sav` back on exit.
- Assemble player/example: `rgbasm -o player.o player.s` etc. (README). RGBDS 1.0 syntax:
  `ldh [$FFxx]` (full address), `DEF X = ...` for constants.

## How recording works (lsdpack.cpp)

- `load_song()` drives LSDj's UI with key presses to load each of the 32 song slots.
- `play_song()` presses START and records until LSDj writes NR52=0 (an HFF command),
  or `-l` seconds, or aborts after 1 emulated hour.
- `on_ff_write()` records writes to $FF10-$FF3F; `on_lcd_interrupt()` records a tick end.
- The tick hook lives in `libgambatte/src/memory.cpp` (interrupt dispatch): it fires when
  the CPU takes the vector in `g_tick_irq_address` — 0x48 (LCD STAT, old LSDj) or
  0x50 (timer, `-t`, LSDj 9.x). This global is defined in lsdpack.cpp; anything else that
  links libgambatte from this repo must define it too.
- `writer.cpp` + `rules/` compress the stream (tick ends fold into the previous command's
  bit 7 as FLAG_END_TICK; repeats use bit 6 + a count byte). Count ticks via both
  `CMD_END_TICK` and `FLAG_END_TICK` comments in the .s output, expanding repeats (1 + count).

## Verifying a change

- Tick rate: ticks in the .s divided by recorded seconds should be ~360 (DMG + `-t`).
  0 ticks = wrong interrupt; ~720 = recorded on CGB.
- To see which interrupts an LSDj version uses, count `address` per vector in the
  dispatch in memory.cpp (temporary instrumentation; don't commit it).
- Play the result: boot.s example ROM (README), ideally on DMG / in an emulator.

## Conventions

- Mark modified upstream files with a "Modified 2026 (M64GitHub fork)" line under the
  original copyright (GPL-2 §2a) and add a CHANGELOG entry tagged "(M64GitHub fork)".
- Keep changes small and upstream-friendly: they may be offered back to jkotlinski/lsdpack.

# lsdpack

> **This is a fork of [jkotlinski/lsdpack](https://github.com/jkotlinski/lsdpack)** that works with
> current LSDj (tested with **LSDj 9.3.9**), **RGBDS 1.0** and **CMake 4**. lsdpack is written by
> Johan Kotlinski, the author of LSDj; this fork only adds what newer versions need.
> See [Recording LSDj 9.x songs](#recording-lsdj-9x-songs) and [Changes in this fork](#changes-in-this-fork).
>
> Want an album ROM with a logo and a music visualizer? [lsdj-album](https://github.com/M64GitHub/lsdj-album)
> builds one from your LSDj save, using this fork.

Records LSDj songs for .gbs conversion or use in stand-alone Game Boy ROMs. (E.g. your own music albums, demos, games...)

## Building
Requires CMake (3.5 or newer, 4.x works), [RGBDS](https://github.com/gbdev/rgbds) 1.0 and a C++ compiler. Exact build steps are platform dependent - see [Running CMake](https://cmake.org/runningcmake/). For example:

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build

![C++ CI with CMake](https://github.com/jkotlinski/lsdpack/workflows/CMake-Ubuntu/badge.svg)
![C++ CI with CMake](https://github.com/jkotlinski/lsdpack/workflows/CMake-Windows/badge.svg)

## Recording Songs

All songs must first be prepared so that they eventually stop with the HFF command. Place your .sav and .gb file in the same directory and run e.g. `./lsdpack.exe lsdj.gb` to record the songs to `lsdj.s`. To record songs from several .gb files, add them all to the command line, like `./lsdpack.exe 1.gb 2.gb 3.gb`.

**Work on copies of your .gb and .sav:** the built-in emulator writes the .sav back when lsdpack exits, and loading songs changes LSDj's working song.

## Recording LSDj 9.x songs

    ./lsdpack -d -t lsdj.gb

LSDj 9.x drives its sound engine from the **timer interrupt**. Older versions used the LCD (STAT) interrupt, and that is where upstream lsdpack marks the end of each player tick. With LSDj 9.x it therefore records **no ticks at all**: the player then runs through a whole song in one call (in boot.s: a black screen and garbled timing).

| Interrupts per second while LSDj 9.3.9 plays | LCD STAT | timer |
|---|---|---|
| emulated CGB (default) | 0 | 720 |
| emulated DMG (`-d`) | 0 | 360 |

* `-t` marks a tick on every timer interrupt.
* `-d` records on an emulated DMG, where LSDj's 360 Hz matches the player's call rate (six calls per frame). On CGB the timer runs at 720 Hz, so `-t` without `-d` would play back at half speed; lsdpack warns about it.
* `-l SECONDS` stops recording each song after that many seconds, for songs that do not end with HFF.

boot.s calls the player from a timer at 65536 Hz / 183 = 358.1 Hz. LSDj 9.x ticks at 360.2 Hz on DMG, so a reload value of `256-182` (360.1 Hz) matches slightly better.

## Game Boy Sound System (GBS)

Create .gbs files using the commands below. For compatibility reasons, .gbs files should be below 512 kB, so each .gbs file will contain one song only.

    # dump songs
	./lsdpack.exe -g lsdj.gb

    # assemble player
	rgbasm -o player.o player.s

    # assemble songs individually
	rgbasm -o lsdj-1.o lsdj-1.s
	rgbasm -o lsdj-2.o lsdj-2.s
    ...

    # make one .gb file for each song
	rgblink -o player-1.gb player.o lsdj-1.o
	rgblink -o player-2.gb player.o lsdj-2.o
    ...

    # create the final .gbs files, one by one
	./makegbs.exe -t "Better Off Alone" -a "Alice Deejay" -c "(C) Violent Music 1997" player-1.gb
    ...

## Playing Songs from Your Own Code

An example Game Boy player ROM can be built using RGBDS:

    rgbasm -o boot.o boot.s
    rgbasm -o player.o player.s
    rgbasm -o lsdj.o lsdj.s
    rgblink -o player.gb boot.o player.o lsdj.o
    rgbfix -v -m 0x19 -p 0xff player.gb

### player.s

Contains the player code. Following symbols are exported:

    ; IN: a = song number
    ; OUT: -
    ; SIDE EFFECTS: trashes af
    ;
    ; Starts playing a song. If a song is already playing,
    ; make sure interrupts are disabled when calling this.
    ;
    LsdjPlaySong::

    ; IN: -
    ; OUT: -
    ; SIDE EFFECTS: changes ROM bank, trashes af
    ;
    ; Call this six times per screen update,
    ; evenly spread out over the screen.
    ;
    LsdjTick::

    ; A table that holds the ROM position for each song.
    ; Each entry is 4 bytes big.
    SongLocations::
    SongLocationsEnd::

### boot.s

An example for how to use the player. Displays CPU usage
using raster bars. Pressing A skips to the next song.

![Screenshot](/docs/screenshot.png)

## Changes in this fork

* LSDj 9.x support: `-t` (tick on timer interrupt) and `-l` (cut songs after N seconds). Tested with LSDj 9.3.9.
* player.s and boot.s ported to RGBDS 1.0 (`ldh [$FFxx]` instead of `ldh [$xx]`, `DEF` for constants).
* CMakeLists.txt accepts CMake 4.
* CI uses actions/checkout v4.

Upstream: [jkotlinski/lsdpack](https://github.com/jkotlinski/lsdpack).

## How Does It Work?

lsdpack plays back LSDj songs using an emulated Game Boy Color and records direct writes to the sound chip. This recording can be played back from another ROM using a custom player.

The included player is very fast and can easily play songs that would choke LSDj on a Game Boy Classic. Since recordings take a lot of ROM, an MBC5 cartridge is recommended.

ROM+CPU consumption varies with song contents. Features like sample playback or FAST/DRUM P and V commands are especially demanding, since they update the sound chip 360 times/second.

To reduce CPU consumption and interfere less with game logic and graphics, it can help to batch up player calls. Examples:

 * Full speed (default): Call player six times/frame, evenly spaced out. Allows ~11 kHz sample playback ("1x" sample speed).
 * Half speed: Make two player calls in a row, three times/frame. Allows ~6 kHz sample playback ("0.5x" sample speed).
 * Normal speed: Make six player calls in a row, once a frame from VBL interrupt. Useful if sample playback is not needed.

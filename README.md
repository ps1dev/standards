# [PSX.Dev](https://psx.dev/) standards

File formats that PlayStation 1 homebrew tools and libraries have agreed on, so that a
file written by one of them can be read by the others.

| Format | Document | What it carries |
| --- | --- | --- |
| GPU dump | [GPUDUMP.md](GPUDUMP.md) | A stream of GPU commands captured over a timespan, replayable on an emulator or on hardware. |
| SPU dump | [SPUDUMP.md](SPUDUMP.md) | A stream of SPU register writes, for capture and replay and for tracker-style playback. |
| PS Music | [PSM.md](PSM.md) | MIDI preprocessed into fixed-size events, paired with a VAB bank, for a player to walk at runtime. |

All three are packet-based, 32-bit aligned, little-endian and extensible: a reader skips
packet types it does not know using the length field, so an old tool keeps working
against a newer file.

Reference players for the SPU dump and PS Music formats live in
[nugget](https://github.com/pcsx-redux/nugget): [spdplayer](https://github.com/pcsx-redux/nugget/tree/main/spdplayer)
and [psmplayer](https://github.com/pcsx-redux/nugget/tree/main/psmplayer).

## Proposing a change

Open an issue or a pull request. A format here is only worth something if more than one
tool implements it, so a proposal is much stronger with a note on what is going to read
and write it. Extensions get a new packet type; what an existing type means does not
change.

Discussion happens on the [PSX.Dev Discord server](https://discord.gg/QByKPpH).

The documents are MIT-licensed; see [LICENSE](LICENSE).

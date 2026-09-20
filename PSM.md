# PS Music format

This document describes the PSM format for preprocessed MIDI event streams targeting the PlayStation 1. PSM files are designed to be interpreted at runtime by a PS1-side player library, paired with a VAB instrument bank and VAG sample data. The format trades offline preprocessing time for minimal runtime CPU cost: all MIDI parsing, multi-track merging, tempo conversion, and instrument resolution are done ahead of time by the offline tool, leaving the PS1 player with a flat array of fixed-size events to walk linearly.

## Design goals

- **Fixed-size events** for direct array indexing on MIPS (no variable-length encoding)
- **Pre-resolved instrument references** so the player never searches the instrument bank
- **Delta-tick timing** with pre-computed tick rates so the player only counts hblanks
- **Minimal PS1 RAM footprint** (8 bytes per event, 16-byte header)
- **Paired with standard VAB** for instrument definitions and sample data

## Relationship to other formats

PSM replaces the SPUDUMP register write stream for use cases where runtime flexibility matters (dynamic tempo, channel muting, game-driven audio control). The tradeoff: SPUDUMP is a dumb replay of pre-baked register writes (zero CPU, large files); PSM requires a real-time voice allocator and pitch computation (small CPU cost, much smaller files).

PSM uses the Sony VAB format verbatim for instrument definitions. The offline tool (midi2psm) takes MIDI + SoundFont (.sf2) as input and produces .psm + .vab as output.

## File structure

A PSM file consists of a header followed by a flat array of events. The format is little-endian throughout.

### Header (16 bytes)

| Offset | Size | Type   | Field      | Description |
|--------|------|--------|------------|-------------|
| 0x00   | 4    | char[4]| magic      | `"PSM\0"` (0x50, 0x53, 0x4D, 0x00) |
| 0x04   | 4    | uint32 | version    | Format version (currently 1) |
| 0x08   | 4    | uint32 | tickRate   | Initial tick rate in 16.16 fixed-point Hz |
| 0x0C   | 4    | uint32 | eventCount | Number of events in the event array |

The tick rate uses the same encoding as the SPUDUMP tick rate packet: the value represents ticks per second multiplied by 65536. For example, 960 ticks/sec (480 TPQN at 120 BPM) is encoded as `960 << 16` = 0x03C00000.

### Event (8 bytes each)

| Offset | Size | Type   | Field     | Description |
|--------|------|--------|-----------|-------------|
| 0x00   | 2    | uint16 | deltaTick | Ticks since previous event (0-65535) |
| 0x02   | 1    | uint8  | type      | Event type (see below) |
| 0x03   | 1    | uint8  | channel   | MIDI channel (0-15) |
| 0x04   | 4    | uint32 | data      | Type-specific payload |

Events are stored in chronological order. The deltaTick field is relative to the previous event (the first event's deltaTick is relative to the start of playback). For gaps longer than 65535 ticks, a LONG_WAIT event (type 0xFF) provides additional ticks.

## Event types

### 0x00 - NOTE_ON

Play a note. The instrument lookup has been pre-resolved by the offline tool.

| Data bits | Field      | Description |
|-----------|------------|-------------|
| 7-0       | note       | MIDI note number (0-127) |
| 15-8      | velocity   | MIDI velocity (1-127, 0 is not used) |
| 23-16     | program    | VAB program index (0-127) |
| 31-24     | toneIndex  | Index into the program's tone array (0-15) |

The player uses `program` and `toneIndex` to look up the VagAtr entry in the VAB, which contains the sample address, ADSR values, center note, pan, and volume. No searching is required.

### 0x01 - NOTE_OFF

Release a note.

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | note  | MIDI note number (0-127) |
| 31-8      | -     | Reserved (0) |

The player finds and releases all active voices matching (channel, note). If the sustain pedal is active, voices are marked as held instead of released.

### 0x02 - PITCH_BEND

Set the pitch bend value for a channel.

| Data bits | Field | Description |
|-----------|-------|-------------|
| 15-0      | bend  | Signed 16-bit bend value (-8192 to 8191, 0 = center) |
| 31-16     | -     | Reserved (0) |

The player updates pitch registers for all active voices on the channel.

### 0x03 - CC_VOLUME (CC#7)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Volume (0-127) |
| 31-8      | -     | Reserved (0) |

### 0x04 - CC_PAN (CC#10)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Pan (0=left, 64=center, 127=right) |
| 31-8      | -     | Reserved (0) |

### 0x05 - CC_EXPRESSION (CC#11)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Expression (0-127) |
| 31-8      | -     | Reserved (0) |

### 0x06 - CC_SUSTAIN (CC#64)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Sustain pedal (0-63 = off, 64-127 = on) |
| 31-8      | -     | Reserved (0) |

When the sustain value goes below 64, the player releases all sustain-held voices on the channel.

### 0x07 - CC_MODULATION (CC#1)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Modulation wheel depth (0-127) |
| 31-8      | -     | Reserved (0) |

Controls vibrato depth. The player applies a periodic pitch offset to active voices.

### 0x08 - CC_REVERB (CC#91)

| Data bits | Field | Description |
|-----------|-------|-------------|
| 7-0       | value | Reverb send level (0-127) |
| 31-8      | -     | Reserved (0) |

Controls per-channel reverb. Voices on channels with reverb > 0 have their SPU reverb enable bit set.

### 0x09 - PROGRAM_CHANGE

| Data bits | Field   | Description |
|-----------|---------|-------------|
| 7-0       | program | New VAB program index (0-127) |
| 31-8      | -       | Reserved (0) |

Updates the channel's current program. Subsequent NOTE_ON events on this channel will use the new program. Note that NOTE_ON events also carry the program index explicitly, so this event is primarily for the player's internal state tracking.

### 0x0A - TEMPO_CHANGE

| Data bits | Field    | Description |
|-----------|----------|-------------|
| 31-0      | tickRate | New tick rate in 16.16 fixed-point Hz |

The player recalculates its hblank counter interval based on the new tick rate.

### 0x0B - LOOP_POINT

| Data bits | Field | Description |
|-----------|-------|-------------|
| 31-0      | -     | Reserved (0) |

Marks the current position as the loop restart point. When the player reaches the END event and the song is set to loop, it returns to this position. Only one loop point per song is valid; a second overrides the first.

### 0x0C - END

| Data bits | Field | Description |
|-----------|-------|-------------|
| 31-0      | -     | Reserved (0) |

Signals the end of the event stream. If looping is enabled and a loop point was set, the player returns to the loop point. Otherwise, playback stops.

### 0xFF - LONG_WAIT

| Data bits | Field          | Description |
|-----------|----------------|-------------|
| 31-0      | additionalTicks | Extra ticks to wait (added to deltaTick) |

Used when the gap between events exceeds 65535 ticks. The total wait is `deltaTick + additionalTicks`. Multiple LONG_WAIT events can be chained for arbitrarily long gaps. LONG_WAIT events have no musical effect; they only advance time.

## Playback

A minimal PSM player performs the following steps:

1. Load the VAB file: parse VabHdr, store ProgAtr and VagAtr tables, DMA the VAG body to SPU RAM.
2. Load the PSM file: read header, store pointer to event array.
3. Configure a hardware timer (hblank counter) based on the initial tick rate.
4. Each timer tick, advance through events:
   - Accumulate deltaTicks. When the accumulated time matches the current tick, process the event.
   - For NOTE_ON: look up ProgAtr[program] and VagAtr[toneIndex] to get sample address, ADSR, center note, volume, pan. Allocate a voice, compute pitch from note number and center note, set SPU registers, key on.
   - For NOTE_OFF: find active voice(s) on (channel, note), key off or mark sustain-held.
   - For CC events: update channel state, propagate to active voices as needed.
   - For TEMPO_CHANGE: recalculate timer interval.
   - For LOOP_POINT: record current event index as loop target.
   - For END: jump to loop point or stop.

## Pitch computation

The PSM player needs to compute SPU pitch register values at runtime. The SPU pitch register uses 0x1000 as the base rate (44100 Hz playback). The formula is:

```
pitch = (noteFreq / rootFreq) * (sampleRate / 44100) * 0x1000
```

On MIPS without an FPU, this is implemented using a precomputed 128-entry semitone-to-frequency-ratio table in 16.16 fixed point, with the sample rate and root key adjustments applied as integer multiplies and shifts.

## Usage notes

- PSM files are always paired with a VAB file. The program and tone indices in NOTE_ON events reference the VAB's ProgAtr and VagAtr tables directly.
- The offline tool (midi2psm) resolves all SF2 region lookups at conversion time. The PS1 player never needs to search for matching regions.
- Channel 10 (0-indexed as 9) is conventionally the drum channel, using VAB program 128+ for drum presets.
- The player should reserve voices 0 through (voiceCount-1) for music and leave the remaining voices available for sound effects via a programmatic API.

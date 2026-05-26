# SPU Dump file format

This document describes a common format for structured SPU register write streams on the PlayStation 1. The format serves two purposes: capture and replay of SPU activity from emulators or real hardware, and composition-driven playback from music tools such as trackers and converters. It is designed to be simple enough that a PS1-side player can be implemented in a few dozen lines of C, while providing enough structure for seeking, looping, and dynamic gameplay integration.

Like the GPU dump format, the SPU dump format is packet-based, 32-bit aligned, little-endian, and extensible. Unknown packet types must be skipped gracefully using their length field.

## Structure of an SPU Dump File

An SPU dump file consists of a magic header followed by a series of packets. The magic header is exactly 16 bytes long. A packet consists of a header followed by a payload. The header of a packet is 4 bytes long, and the payload can vary in size depending on the type of packet. The total size of a packet (header + payload) must be a multiple of 4 bytes. All lengths are expressed in 32-bit words. Strings are zero-padded to ensure 32-bit alignment. The format is little-endian throughout.

### Magic Header

| Byte 0-3 | Byte 4-7 | Byte 8-11 | Byte 12-15 |
|----------|----------|-----------|------------|
| `P S X S`| `P U D U`| `M P v 1` | `r 1 \0 \0`  |

The magic header identifies the file format and version. The first 10 bytes (`PSXSPUDUMP`) are a unique identifier, while the remaining bytes (`v1r1`) indicate the format version. Major version changes indicate breaking changes; minor version changes indicate non-breaking additions. A tool that opens an SPU dump file with an unknown major version must treat it as an error. Unknown packet types within a known version must be skipped without error.

### Packet Structure

Each packet consists of a header and a payload. The header is 4 bytes long:

| Byte 0-2 | Byte 3  |
|----------|---------|
| Length   | Type    |

- **Length**: The length of the payload in 32-bit words (not including the header), stored in the lowest 3 bytes.
- **Type**: The type of the packet, stored in the highest byte. Determines how to interpret the payload.

## Packet Types

### Core Packets

| Type | Description                       |
|------|-----------------------------------|
| 0x00 | SPU register write                |
| 0x01 | Wait                              |
| 0x02 | End of pattern                    |
| 0x03 | Loop point                        |
| 0x04 | Trace begin                       |

### Structure Packets

| Type | Description                       |
|------|-----------------------------------|
| 0x10 | Order table                       |
| 0x11 | Pattern header                    |
| 0x12 | Subsong table                     |
| 0x13 | Macro definition                  |

### Sample Data Packets

| Type | Description                       |
|------|-----------------------------------|
| 0x20 | Sample directory                  |
| 0x21 | Sample data                       |

### Timing Packets

| Type | Description                       |
|------|-----------------------------------|
| 0x30 | Tick rate                         |

### Metadata Packets

| Type | Description                       |
|------|-----------------------------------|
| 0x40 | Title                             |
| 0x41 | Author                            |
| 0x42 | Game ID                           |
| 0x43 | Comment                           |
| 0x44 | Subsong name                      |
| 0x45 | Voice count                       |

## Description of Packet Types

### Core Packets

- **SPU register write (0x00)**: The payload consists of one or more 32-bit words, each encoding a single operation. The upper 16 bits are the register offset and the lower 16 bits are the value. Multiple writes in a single packet are processed sequentially in order. This batching is useful for operations where multiple registers must be written atomically, such as setting up a voice's pitch, ADSR, and volume before issuing a key-on. A payload length of 0 is valid and has no effect.

  Register offsets below 0x200 are real SPU register writes relative to the SPU base address (0x1F801C00). Offsets in the virtual register range have special meaning:

  - **0xEFFF (inline wait)**: The value field is a tick count. The player pauses for this many ticks before processing the next word. Multiple consecutive inline waits are additive (used for tick counts exceeding 0xFFFF). This allows collapsing a wait into the same packet as the register writes that precede it, eliminating a separate wait packet and its 4-byte header overhead.
  - **0xF000-0xFFFF (macro invocation)**: See the macro definition packet (0x13) below.

- **Wait (0x01)**: The payload is a single 32-bit word indicating the number of timer ticks to wait before processing the next packet. The tick rate is defined by the tick rate packet (0x30). A wait of 0 is valid and has no effect. Consecutive wait packets are additive.

- **End of pattern (0x02)**: The payload size is 0. Signals that the current pattern is complete. The player advances to the next entry in the current order table and begins processing the corresponding pattern. If the current order is the last entry, behavior depends on the order table's loop setting.

- **Loop point (0x03)**: The payload size is 0. Marks the current stream position as the loop restart point within the current pattern. When the player reaches the end of pattern (0x02) and the pattern is set to loop, it returns to this position. Only one loop point per pattern is valid; a second loop point overrides the first.

- **Trace begin (0x04)**: The payload size is 0. Indicates that all preceding packets were synthetically generated to restore SPU state, and that subsequent packets represent actual captured data. This is analogous to the GPU dump format's trace begin packet. For composition-driven files, this packet is not required.

### Structure Packets

- **Order table (0x10)**: Defines the playback order for a subsong. The payload consists of:
  - 1 word: flags. Bit 0: loop (1 = loop to loop point or start when order ends, 0 = stop). Bits 1-31: reserved.
  - 1 word: loop target order index (which order entry to loop back to when the end is reached, ignored if loop flag is 0).
  - N words: pattern indices, one per order entry. Each index refers to a pattern header packet (0x11) by its ordinal position in the file (0-based).

  If no order table is present, the file is treated as a flat stream with no seeking support. Multiple order tables may be present for subsong support, and are associated with subsongs by their ordinal position in the file.

- **Pattern header (0x11)**: Defines the start of a pattern's data. The payload consists of:
  - 1 word: byte offset from the start of the file to the first packet of this pattern's register write stream.

  All packets between a pattern header's target offset and the corresponding end of pattern (0x02) belong to that pattern. The first packets in a pattern's stream should be SPU register writes (0x00) that fully restore all active voice state, forming a complete SPU snapshot. This ensures any pattern can be entered cold via seeking.

- **Subsong table (0x12)**: Defines multiple subsongs within a single file. The payload consists of:
  - N words: order table indices, one per subsong. Each index refers to an order table packet (0x10) by its ordinal position in the file (0-based).

  If no subsong table is present, the first order table in the file is used. Subsong names can be associated via subsong name metadata packets (0x44) by ordinal position.

- **Macro definition (0x13)**: Defines a reusable sequence of register writes that can be invoked by index during playback. This is a compression mechanism: register values that are constant for a given instrument or sample zone (such as ADSR parameters and sample start address) are stored once in a macro and referenced by index instead of being repeated inline for every note. The payload consists of:
  - 1 word: macro index (lower 16 bits), reserved (upper 16 bits). Indices must be sequential starting from 0.
  - N words: register writes in the same format as an SPU register write packet (0x00). Each word is a 16-bit register offset (upper) and 16-bit value (lower). The register offsets are voice-relative, using voice 0 as the base. The player adds `voice * 0x10` to each offset at invocation time to target the correct voice.

  Macro definitions must appear before any macro invocations that reference them. The player builds an internal table mapping macro indices to their write sequences.

  Macro invocations are encoded as virtual register writes within SPU register write packets (0x00). A write word with offset >= 0xF000 is interpreted as a macro invocation: the macro index is `offset & 0x0FFF`, and the value field is the target voice number. The player looks up the macro by index, iterates its register writes, adds `voice * 0x10` to each offset, and writes the values to the SPU. This encoding avoids the overhead of a separate packet for each invocation, allowing macro invocations to be freely intermixed with regular register writes in the same packet.

### Sample Data Packets

- **Sample directory (0x20)**: Optional. Describes individual samples within the sample data blob for out-of-band programmatic playback (sound effects, jingles, or other game-triggered audio). This packet is not required for music stream playback - the register writes in the stream already contain the correct SPU RAM addresses for every voice the song uses. The sample directory exists solely to support a player API (analogous to `MOD_PlaySoundEffect()`) that needs to look up a sample by index and configure a spare voice to play it.

  The payload consists of:
  - 1 word: number of samples (N).
  - N x 2 words per sample:
    - Word 0: SPU RAM address in 8-byte units (upper 16 bits), length in 8-byte units (lower 16 bits).
    - Word 1: loop address in 8-byte units (upper 16 bits), flags (lower 16 bits). Flag bit 0: has loop.

  Addresses are in 8-byte units to match the SPU hardware's addressing granularity.

- **Sample data (0x21)**: Raw SPU ADPCM sample data, ready to DMA to SPU RAM. The payload is a contiguous block of pre-encoded PS1 SPU ADPCM data with loop flags baked into the ADPCM block headers. The player uploads this data to SPU RAM starting at a base address specified in the first word of the payload, followed by the raw ADPCM data. The layout within SPU RAM is determined by the exporting tool; the register writes in the stream reference the correct addresses directly.

  The payload consists of:
  - 1 word: SPU RAM base address in 8-byte units.
  - Remaining words: raw ADPCM data, to be DMA'd contiguously starting at the base address.

### Timing Packets

- **Tick rate (0x30)**: Sets the timer tick rate for interpreting wait packets. The payload is a single 32-bit word representing the tick rate in units of 1/65536 Hz (fixed-point 16.16). For example, 50 Hz would be encoded as `50 << 16` = 0x00320000. This allows fractional Hz values for precise timing. If no tick rate packet is present, the default is 50 Hz (PAL vblank rate). A tick rate packet may appear anywhere in the stream to change the rate mid-playback. The player should reconfigure its hardware timer accordingly.

### Metadata Packets

- **Title (0x40)**: Zero-padded string containing the song title.
- **Author (0x41)**: Zero-padded string containing the author name.
- **Game ID (0x42)**: Zero-padded string in the format `SLUS-12345` identifying the source game, for capture-mode files.
- **Comment (0x43)**: Zero-padded string for tool identification, version numbers, or other context.
- **Subsong name (0x44)**: Zero-padded string naming a subsong. Multiple subsong name packets are associated with subsongs by ordinal position.
- **Voice count (0x45)**: A single 32-bit word indicating the number of SPU voices used by the music stream, starting from voice 0. For example, a value of 16 means the stream uses voices 0-15, and voices 16-23 are available for the game to use for sound effects via the player's programmatic playback API. If this packet is not present, the player should assume all 24 voices are used by the stream.

## File Layout

A well-formed SPU dump file should be laid out in the following order, though parsers must not depend on this ordering except where noted:

1. Magic header (required, must be first)
2. Metadata packets (title, author, game ID, comment)
3. Tick rate packet
4. Sample directory and sample data
5. Subsong table (if multiple subsongs)
6. Order tables
7. Pattern headers
8. Pattern data streams (register writes, waits, loop points, end of pattern markers)

For capture-mode files without musical structure, the file may omit order tables, pattern headers, and subsong tables entirely. In this case the file is a flat stream of register writes, waits, and optionally a trace begin marker separating state restoration from captured data.

## Playback

A minimal player performs the following steps:

1. Read the magic header and verify the version.
2. Parse the sample directory and upload sample data to SPU RAM via DMA.
3. Parse the order table for the active subsong.
4. Read the tick rate and configure a hardware timer (hblank counter or root counter).
5. Begin processing the first pattern's stream:
   - For each SPU register write packet: write all register values to the SPU.
   - For each wait packet: pause for the specified number of timer ticks.
   - For each end of pattern packet: advance to the next order entry and load that pattern.

Seeking to an arbitrary order position is accomplished by setting the current order index and jumping to the corresponding pattern's stream offset. Because each pattern begins with a full SPU state snapshot, no history is required.

## Usage Notes

- For composition-driven files, the SPU state snapshot at the head of each pattern should write all registers for every voice used by the song, including volume, pitch, ADSR, and sample address registers. Voices not used by the song should not be touched, leaving them available for sound effects.
- For capture-mode files, a trace begin packet (0x04) should separate the initial state restoration from live data. The state restoration section should write all 24 voices' registers and all global SPU registers to ensure clean replay from any starting point.
- The register write packet's batching capability should be used for key-on writes: accumulate all voice setup writes in one packet, then issue the key-on in a subsequent packet (or the same packet, after the setup writes) to ensure correct ordering.
- Sound effects in games can use SPU voices above the song's voice count. The voice count metadata packet (0x45) declares how many voices the stream uses, starting from voice 0. Voices above that count are guaranteed untouched by the stream and can be freely used by the game for sound effects via the player's programmatic playback API.

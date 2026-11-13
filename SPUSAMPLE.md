# SPU sample signatures and replacement packs

This document describes how to identify an ADPCM sample in PlayStation 1 sound RAM by its content, and a SoundFont-based pack format that maps those identities to replacement audio. An emulator that implements both can export the samples a game uploads, let someone replace them with higher quality recordings, and play the replacements in place of the originals when the game keys them on, while the original sample keeps driving everything the game can observe.

The pack is an ordinary SoundFont 2 file, so it can be opened and edited in any SoundFont editor. What this document adds is the meaning of the sample names, the pitch rules, and what a player does at key-on.

## Sample signature

### Sound RAM and blocks

Sound RAM is 512 KiB (0x80000 bytes). An ADPCM sample is a run of 16-byte blocks. Byte 0 of a block holds the shift and filter, byte 1 holds the flags, and bytes 2 to 15 hold 28 four-bit samples. The flags used here are:

| Bit | Mask | Meaning    |
|-----|------|------------|
| 0   | 0x01 | Loop end   |
| 1   | 0x02 | Repeat     |
| 2   | 0x04 | Loop start |

The walk below ignores the other bits of byte 1. The signature hashes them with the rest of the block.

### Extent

A sample's start address is the voice's start address register at key-on, multiplied by 8 and rounded down to a multiple of 16. From the start address, blocks are read one at a time. The first block whose loop end bit is set is the sample's last block, and the sample's extent runs from the start address up to the end of that block.

If the walk reaches address 0x80000 without finding a loop end bit, the sample has no extent and no signature, and it is never replaced. The walk does not wrap around to address 0.

The loop block of a sample is the last block of its extent, the last block included, that has the loop start bit set. A sample loops when its last block has the repeat bit set and it has a loop block. These two properties are used when exporting a sample (see [Writing a pack](#writing-a-pack)); they are not part of the signature.

### Signature

The signature is the MD5 digest ([RFC 1321](https://www.rfc-editor.org/rfc/rfc1321)) of every byte of the extent, block headers included, taken as a 64-bit number from the first 8 bytes of the digest, most significant byte first. Written as text it is 16 hexadecimal digits, which are the first 16 digits of the usual hexadecimal form of the MD5 digest.

Writers use lowercase digits. Readers accept either case.

Only the extent's bytes go into the signature. The address the sample is stored at does not, so the same sample uploaded to another address has the same signature, and neither does any loop address the game writes to a voice register.

## Pack format

A pack is a SoundFont 2 bank (a RIFF file of form type `sfbk`). Readers also accept SF3 banks, whose samples are Ogg Vorbis streams: a sample header whose sample type has bit 4 (0x10) or bit 5 (0x20) set is compressed, its start and end are byte offsets of the Vorbis stream in the `smpl` chunk, and only the stream's first channel is used.

### Replacements

Each sample header (`shdr` record) other than the terminal one is read on its own. The name is the header's 20-byte name field up to its first zero byte. A sample header is a replacement when:

- its name is exactly 16 hexadecimal digits, which are the signature of the sample it replaces,
- its sample rate is not 0, and
- its data lies inside the `smpl` chunk (or, for SF3, decodes) and has at least one point.

Other sample headers are ignored, so a pack can carry other samples. When two replacements carry the same signature, the later sample header in the file is the one used.

From each replacement header the reader takes:

| Field              | Use                                                                 |
|--------------------|---------------------------------------------------------------------|
| sample data        | The points from start to end. 16-bit; any `sm24` chunk is ignored.  |
| `dwSampleRate`     | The rate the points were recorded at, R.                            |
| `byOriginalPitch`  | The root key, K: the MIDI note the sample sounds at rate R. A value above 127 is read as 60. |
| `chPitchCorrection`| The correction, C, in cents, signed. Positive raises the pitch.     |
| `dwStartloop`, `dwEndloop` | The loop, if the sample loops (below).                     |

### Loops

A replacement loops when an instrument zone that plays it (its `sampleID` generator names the sample) has `sampleModes` set to 1 or 3. A zone without its own `sampleModes` uses the one in its instrument's global zone, if that zone sets one. Value 3 is played as a plain loop: the loop is not left at key-off.

The loop runs from `dwStartloop` up to, but not including, `dwEndloop`, both counted from the start of the sample. For an SF3 sample they count points of the decoded stream. A loop is only used when its start is before its end and its end is not past the last point; otherwise the replacement does not loop.

Presets, instruments and all other generators (key ranges, tuning, envelopes, filters, stereo links) are ignored. A pack's replacements are reached through their names, never through a preset.

## Playing a pack

Sample rates and pitches below are in terms of the SPU's 44100 Hz output: a voice at pitch 0x1000 plays its sample at one point per output sample.

### Key-on

At each key-on, the player computes the extent and signature of the sample at the voice's start address, from sound RAM as it is at that key-on. If the pack has a replacement for that signature, the voice plays the replacement until its next key-on. Otherwise it plays the original. A pack loaded while a voice plays takes effect at that voice's next key-on.

Voices in noise mode are not replaced.

### What is replaced

The replacement stands in for the voice's decoded ADPCM sample: the point the voice would have produced from its sample data is replaced by the replacement's point. The voice's envelope applies to it, and the result goes to the voice's volume, the mix and the reverb input.

Everything else keeps coming from the original sample, which the voice keeps decoding underneath: the end flags and the end-of-sample status, the loop flags and the repeat address, sound RAM IRQ address hits, the capture buffers for voices 1 and 3, and the input that a frequency-modulating voice passes to the voice above it.

### Pitch

The replacement starts at its first point at key-on. For each output sample, its playhead advances by

```
step x R / 44100 x 2^((N - K) / 12 + C / 1200)
```

where `step` is the voice's sample step for that output sample, in points of the original per output sample (1.0 at pitch 0x1000), frequency modulation included; R, K and C come from the replacement; and N is the note the original sample sounds at pitch 0x1000 (below).

So a replacement whose root key and correction describe its own pitch sounds the same note as the original at every pitch the game plays it at, whatever note the original was recorded on.

When a looping replacement's playhead reaches its loop end, the loop length is subtracted until it is back inside the loop. When a replacement that does not loop runs past its last point, the voice is silent for the rest of the note; it does not fall back to the original.

### The original's note

N is a fractional MIDI note: `69 + 12 x log2(f / 440)`, where f is the fundamental frequency of the original sample decoded at 44100 Hz. Note 69 is 440 Hz, which is A4 in scientific pitch notation. When the original has no fundamental, or when f rounds to a note outside 0 to 127, N is 60.

The original is decoded from its start address with the filter history cleared, as key-on does. A looping original is decoded round its loop until there are at least as many points past the loop start as the analysis needs, with the filter history carried across the loop as playback does. The analysis window is then:

- for a looping original, the 16384 points that follow the loop start, decoding round the loop;
- for a one-shot original, up to 16384 points starting after its attack, where the attack is the first 2048 points or the first eighth of the sample, whichever is shorter.

The estimator that finds f in that window is not specified (see [Unspecified](#unspecified)).

## Writing a pack

A writer that exports samples from sound RAM so that the unedited pack plays back unchanged writes, for each sample:

- the name: its signature, 16 lowercase hexadecimal digits;
- the sample data: the extent decoded with the filter history cleared, every block in order, 28 points per block, clamped to 16 bits, at rate 44100;
- the root key: N rounded to the nearest note, and the correction: `(root key - N) x 100` cents rounded, limited to -50 to 50. When N is 60 because the sample has no fundamental or its note is out of range, the root key is 255, which readers take as 60, and the correction is 0;
- if the sample loops: a zone with `sampleModes` 1, a loop start of `(loop block address - start address) / 16 x 28` points, and a loop end equal to the number of points. The sample data then continues past the loop end with 8 points copied from the loop start; a player of the pack plays the loop and never reaches them.

A loop address the game writes to a voice register is not part of a sample and is not exported.

Played back unedited, such a pack gives N - K + C/100 = 0 to within the half-cent rounding of the correction, so each replacement plays at the original's pitch.

## Test vectors

### Signatures

Each vector is a sample's blocks in hexadecimal, one block per line, and the signature of the extent walked from the given start address.

V1, one block with the loop end bit only. Signature `d7e212b88d167b07`.

```
0c 01 22 33 44 55 66 77 88 99 aa bb cc dd ee ff
```

V2, three blocks: loop start on block 0, loop end and repeat on block 2. The loop block is block 0 and the sample loops. Signature `849b04794e5240df`.

```
1b 04 32 43 54 65 76 87 98 a9 ba cb dc ed fe 0f
2a 00 42 53 64 75 86 97 a8 b9 ca db ec fd 0e 1f
39 03 52 63 74 85 96 a7 b8 c9 da eb fc 0d 1e 2f
```

The same bytes stored at 0x30000 and walked from 0x30007 give a start address of 0x30000 and the same signature, `849b04794e5240df`.

V3, as V2 but with loop start on blocks 0 and 1. The loop block is block 1. Signature `cb3116b83d4b0854`.

```
1b 04 32 43 54 65 76 87 98 a9 ba cb dc ed fe 0f
2a 04 42 53 64 75 86 97 a8 b9 ca db ec fd 0e 1f
39 03 52 63 74 85 96 a7 b8 c9 da eb fc 0d 1e 2f
```

V4, two blocks, the last one carrying loop end, repeat and loop start (flags 0x07). The loop block is block 1, the last block itself. Signature `2618c05175d2688f`.

```
1b 00 32 43 54 65 76 87 98 a9 ba cb dc ed fe 0f
2a 07 42 53 64 75 86 97 a8 b9 ca db ec fd 0e 1f
```

V5, one block with flags 0xf9: loop end, plus bits that do not affect the walk but are hashed. Signature `9fcd6f024aae4596`.

```
0c f9 22 33 44 55 66 77 88 99 aa bb cc dd ee ff
```

V6, blocks from 0x7ffd0 to the end of sound RAM with no loop end bit: no extent, no signature.

### Reference pack

[spusample/reference.sf2](spusample/reference.sf2) (6634 bytes, MD5 `76b4f71ee3849c10c23231852f5ff922`) has three samples:

| Sample header      | Points | Rate  | Root key | Correction | Loop       |
|--------------------|--------|-------|----------|------------|------------|
| `849b04794e5240df` | 508    | 44100 | 69       | -4         | 400 to 500 |
| `d7e212b88d167b07` | 2205   | 22050 | 255      | 0          | none       |
| `not-a-signature`  | 64     | 44100 | 60       | 0          | none       |

A reader finds two replacements and ignores the third sample. The first is a 441 Hz sine, looped over one 100-point period, and replaces V2. The second is decaying noise, unpitched, and replaces V1. V3 has no replacement.

At pitch 0x1000 the playhead of the V2 replacement advances 0.593231317 points per output sample when the original's note N is 60, and 0.997692177 when N is 69. The V1 replacement advances 0.5 points per output sample when N is 60.

## Unspecified

These are left to the implementation, and two implementations may differ on them:

- The fundamental frequency estimator. Two players whose estimators measure a different N for the same original play its replacement at pitches that differ by that difference, and a player that finds no fundamental where another finds one uses N = 60. The pack does not record the original's note.
- How the replacement is interpolated between its points, and across its loop.
- The exact moment within a key-on at which sound RAM is read for the signature, relative to transfers into sound RAM that are in progress.
- What happens to a playing replacement when emulator state is restored from a save state.
- Loading more than one pack at once.
- A version marker. A pack has none beyond the SoundFont version in its `ifil` chunk.

## Implementations

[PCSX-Redux](https://github.com/pcsx-redux/pcsx-redux) writes packs from its SPU Samples window (Export SoundFont...) and plays them (Load replacements...).

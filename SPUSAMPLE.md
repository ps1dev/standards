# SPU sample signatures and replacement packs

This document describes how to identify an ADPCM sample in PlayStation 1 sound RAM by its content, and a SoundFont-based pack format that maps those identities to replacement audio. An emulator that implements both can export the samples a game uploads, let someone replace them with higher quality recordings, and play the replacements in place of the originals when the game keys them on, while the original sample keeps driving everything the game can observe.

The pack is an ordinary SoundFont 2 file, so it can be opened and edited in any SoundFont editor. What this document adds is the meaning of the sample names, the pitch rules, and what a player does at key-on.

Playing a pack is the simpler half and the one most implementations need: the extent walk, the signature, the records and the pitch rule. Writing a pack also needs a pitch estimator and the loop rules of [Writing a pack](#writing-a-pack). The Python snippets below are reference sketches of each step, with error handling left to the reader. For how the SPU decodes ADPCM and plays voices, see the [psx-spx Sound Processing Unit](https://psx-spx.consoledev.net/soundprocessingunitspu/) page.

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

The loop block of a sample is the last block of its extent, the last block included, that has the loop start bit set. A sample loops when its last block has the repeat bit set and it has a loop block. A sample whose last block has the repeat bit set but which has no loop block jumps, at its end, to the address in the voice's loop register, which the game sets. These properties are used when exporting a sample (see [Writing a pack](#writing-a-pack)); they are not part of the signature.

### Signature

The signature is the MD5 digest ([RFC 1321](https://www.rfc-editor.org/rfc/rfc1321)) of every byte of the extent, block headers included, taken as a 64-bit number from the first 8 bytes of the digest, most significant byte first. Written as text it is 16 hexadecimal digits, which are the first 16 digits of the usual hexadecimal form of the MD5 digest.

Writers use lowercase digits. Readers accept either case.

Only the extent's bytes go into the signature. The address the sample is stored at does not, so the same sample uploaded to another address has the same signature, and neither does any loop address the game writes to a voice register.

```python
import hashlib

def extent(ram, start_register):
    start = (start_register * 8) & ~15
    for addr in range(start, 0x80000, 16):
        if ram[addr + 1] & 0x01:          # loop end: this is the last block
            return start, addr + 16
    return None                           # no loop end before 0x80000: no signature

def signature(ram, start, end):
    return int.from_bytes(hashlib.md5(ram[start:end]).digest()[:8], 'big')
```

MD5 is used because every language ships it in its standard library; no security property is needed. Its output is cut to 64 bits. Sound RAM holds at most 32768 extents (16 bytes each), and the chance that two different extents in one sound RAM image share a 64-bit signature is below n squared over 2 to the 65th ([birthday problem](https://en.wikipedia.org/wiki/Birthday_problem)): about 3 x 10^-11 at that maximum, and about 2 x 10^-15 for 300 samples. Speed was measured against xxHash on one x86-64 core: hashing 512 KiB, the largest extent a single key-on can have, takes about 1.06 ms with MD5 and 0.056 ms with XXH64. Over a whole game soundtrack (90 tracks) the busiest second of key-ons hashed 439040 bytes, about 0.9 ms of MD5 per second of playback before the size check described in [Key-on](#key-on), which was judged small enough to keep a hash every standard library has.

## Pack format

A pack is a SoundFont 2 bank (a RIFF file of form type `sfbk`). Readers also accept SF3 banks, whose samples are Ogg Vorbis streams: a sample header whose sample type has bit 4 (0x10) or bit 5 (0x20) set is compressed, its start and end are byte offsets of the Vorbis stream in the `smpl` chunk, and only the stream's first channel is used.

### Replacements

Each sample header (`shdr` record) other than the terminal one is read on its own. The name is the header's 20-byte name field up to its first zero byte. A sample header is a replacement when:

- its name is exactly 16 hexadecimal digits, which are the signature of the sample it replaces,
- its sample rate is not 0,
- its data lies inside the `smpl` chunk (or, for SF3, decodes) and has at least one point, and
- the bank's comment has a record for its signature (see [Sample records](#sample-records)).

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

The loop runs from `dwStartloop` up to, but not including, `dwEndloop`. In an SF2 sample both are point indices into the `smpl` chunk, like `dwStart`, so the loop starts `dwStartloop - dwStart` points into the sample and ends `dwEndloop - dwStart` points into it; a `dwStartloop` before `dwStart` means no loop. In an SF3 sample they count points of the decoded stream from its start. A loop is only used when its start is before its end and its end is not past the last point; otherwise the replacement does not loop.

Presets, instruments and all other generators (key ranges, tuning, envelopes, filters, stereo links) are ignored. A pack's replacements are reached through their names, never through a preset.

### Sample records

The bank's comment, the `ICMT` chunk of its `INFO` list read up to its first zero byte, holds one record per original sample: the size of its extent, and the note it sounds at pitch 0x1000 (N, below). Each record is one line:

```
spu-sample <signature> <size> <note>
```

`spu-sample`, one space, the signature as 16 hexadecimal digits, one space, the size, one space, and the note. The size is the number of bytes the signature hashes, as one to six decimal digits: a multiple of 16, from 16 to 524288. The note is one to three decimal digits, optionally followed by a `.` and one or more decimal digits, and is less than 128. For example, `spu-sample 849b04794e5240df 48 57.5000`.

Lines are separated by line feeds, and a carriage return before a line feed is ignored. A line that does not have exactly this form is not a record and is ignored, so the comment can carry other text. When two records carry the same signature, the later one is used. A record whose signature matches no replacement is ignored.

A pack author can keep a note about a sample in the same comment, as a line `spu-note <signature> <text>` whose text runs to the end of the line. It is not a record, so readers ignore it like any other text.

```python
import re

RECORD = re.compile(r'spu-sample ([0-9a-fA-F]{16}) ([0-9]{1,6}) ([0-9]{1,3}(\.[0-9]+)?)')

def read_records(icmt):                   # icmt: the ICMT chunk's bytes
    records = {}
    for line in icmt.split(b'\0', 1)[0].decode('latin-1').split('\n'):
        if line.endswith('\r'):
            line = line[:-1]
        m = RECORD.fullmatch(line)
        if not m:
            continue
        size, note = int(m[2]), float(m[3])
        if size % 16 == 0 and 16 <= size <= 524288 and note < 128:
            records[int(m[1], 16)] = (size, note)    # a later record replaces an earlier one
    return records
```

A replacement is then any SoundFont sample header that meets the four conditions above, the record for its signature being one of them; reading the samples themselves is ordinary SoundFont parsing.

The records live in the comment because SoundFont editors keep it: Polyphone 2.2, re-saving a bank, keeps `ICMT` byte for byte and drops both a private `INFO` sub-chunk and a private top-level `LIST`.

## Playing a pack

Sample rates and pitches below are in terms of the SPU's 44100 Hz output: a voice at pitch 0x1000 plays its sample at one point per output sample.

### Key-on

At each key-on, the player computes the extent and signature of the sample at the voice's start address, from sound RAM as it is at that key-on. If the pack has a replacement for that signature, and the replacement's record carries the extent's size, the voice plays the replacement until its next key-on. Otherwise it plays the original.

The walk reads one byte per block and the hash reads every byte, so a player can compare the extent's size with the records' sizes first. A size that no record carries cannot match, and the player can skip the signature. A player that does not compare sizes first gets the same result. No logger or other record of past key-ons is needed to play a pack. A pack loaded while a voice plays takes effect at that voice's next key-on.

```python
def key_on(voice, ram, pack):
    voice.replacement = None
    voice.silent = False
    found = extent(ram, voice.start_register)
    if found is None or voice.noise:
        return
    start, end = found
    if end - start not in pack.sizes:      # optional shortcut, same result
        return
    rep = pack.replacements.get(signature(ram, start, end))
    if rep is not None and rep.size == end - start:
        voice.replacement = rep
        voice.position = 0.0
```

A player may cache extents and signatures by start address, provided it drops what a write can change. The game changes sound RAM only by transfers (DMA or manual writes), but the SPU writes two areas by itself: the capture buffers at 0x0000-0x0FFF, and, while reverb is on, the reverb work area from the reverb base address to the end of sound RAM. A cache that is cleared on every transfer and never holds an extent overlapping those areas gives the same result as no cache.

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

```python
def advance(voice, step):                 # step: original points per output sample, FM included
    rep = voice.replacement
    root = 60 if rep.root_key > 127 else rep.root_key
    voice.position += step * rep.rate / 44100 * 2 ** ((rep.note - root) / 12 + rep.correction / 1200)
    if rep.loops:
        if voice.position >= rep.loop_end:     # same as subtracting the loop length until inside
            length = rep.loop_end - rep.loop_start
            voice.position = rep.loop_start + (voice.position - rep.loop_start) % length
    elif voice.position >= len(rep.points):
        voice.silent = True               # until the next key-on
```

### The original's note

N is the note the original sample sounds at pitch 0x1000, as the pack's record for the replacement's signature gives it. A player does not measure it.

## Writing a pack

A writer that exports samples from sound RAM so that the unedited pack plays back unchanged writes, for each sample:

- the name: its signature, 16 lowercase hexadecimal digits;
- the sample data: the extent decoded with the filter history cleared, every block in order, 28 points per block, clamped to 16 bits, at rate 44100;
- the root key: N rounded to the nearest note, and the correction: `(root key - N) x 100` cents rounded, limited to -50 to 50. When N is 60 because the sample has no fundamental or its note is out of range, the root key is 255, which readers take as 60, and the correction is 0;
- a record with the extent's size and N, the note written with four decimal places. N is 60 when the sample has no fundamental, and that is recorded too;
- if the sample loops: a zone with `sampleModes` 1, a loop start of `(loop block address - start address) / 16 x 28` points, and a loop end equal to the number of points. The sample data then continues past the loop end with 8 points copied from the loop start; a player of the pack plays the loop and never reaches them. Only the first pass through the loop is exact: the original decodes each later pass with the filter history that its loop end left, while the replacement repeats the points of the first pass. Where those histories differ, so do the later passes.

A sample whose last block has the repeat bit set and which has no loop block takes its loop from the voice's loop register: of the values the writer saw in that register at the sample's key-ons that lie inside the extent, the most frequent one is its loop block address, the lowest one on a tie. When no such value was seen, the sample is exported as a one-shot. A loop address is never part of the signature, so when a game keys the same sample bytes with different loop addresses inside them, the pack carries the chosen loop for all of those key-ons and plays back unchanged only for the ones that used it.

When a bank's comment would not fit every record, the records go into the comment before any other text the writer adds after its first paragraph, and the samples whose records do not fit are not replacements.

Played back unedited, such a pack gives N - K + C/100 = 0 to within the half-cent rounding of the correction, so each replacement plays at the original's pitch.

### How a writer measures N

N is a fractional MIDI note: `69 + 12 x log2(f / 440)`, where f is the fundamental frequency of the original sample decoded at 44100 Hz. Note 69 is 440 Hz, which is A4 in scientific pitch notation. When the original has no fundamental, when that note is below 0, or when it rounds to a note above 127, N is 60.

The original is decoded from its start address with the filter history cleared, as key-on does. A looping original, which includes a sample given its loop by the loop register as above, is decoded round its loop until there are at least as many points past the loop start as the analysis needs, with the filter history carried across the loop as playback does. The analysis window is then:

- for a looping original, the 16384 points that follow the loop start, decoding round the loop;
- for a one-shot original, up to 16384 points starting after its attack, where the attack is the first 2048 points or the first eighth of the sample, whichever is shorter.

The estimator that finds f in that window is not specified. Because N is recorded, a player never depends on it. PCSX-Redux uses YIN (A. de Cheveigné and H. Kawahara, "YIN, a fundamental frequency estimator for speech and music", Journal of the Acoustical Society of America 111(4), 2002); any estimator will do.

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

[spusample/reference.sf2](spusample/reference.sf2) (6712 bytes, MD5 `d59b43035988e0d7a0cc3ac0d17e4257`) has three samples, and its comment carries two records, `spu-sample 849b04794e5240df 48 57.5000` and `spu-sample d7e212b88d167b07 16 60.0000`:

| Sample header      | Points | Rate  | Root key | Correction | Loop       |
|--------------------|--------|-------|----------|------------|------------|
| `849b04794e5240df` | 508    | 44100 | 69       | -4         | 400 to 500 |
| `d7e212b88d167b07` | 2205   | 22050 | 255      | 0          | none       |
| `not-a-signature`  | 64     | 44100 | 60       | 0          | none       |

A reader finds two replacements and ignores the third sample. The first is a 441 Hz sine, looped over one 100-point period, and replaces V2. The second is decaying noise, unpitched, and replaces V1. V3 has no replacement.

At pitch 0x1000 the playhead of the V2 replacement advances 0.513463394 points per output sample, from its recorded N of 57.5 (it would be 0.593231317 for an N of 60). The V1 replacement, recorded at N = 60, advances 0.5 points per output sample.

## Unspecified

These are left to the implementation, and two implementations may differ on them:

- The fundamental frequency estimator a writer uses to measure N.
- How the replacement is interpolated between its points, and across its loop.
- The exact moment within a key-on at which sound RAM is read for the signature, relative to transfers into sound RAM that are in progress.
- What happens to a playing replacement when emulator state is restored from a save state.
- Loading more than one pack at once.
- A version marker. A pack has none beyond the SoundFont version in its `ifil` chunk.

## References

- [psx-spx: Sound Processing Unit](https://psx-spx.consoledev.net/soundprocessingunitspu/): sound RAM, ADPCM blocks, voices and pitch.
- [SoundFont 2.04 specification](https://www.synthfont.com/sfspec24.pdf). Libraries that read it include [FluidSynth](https://www.fluidsynth.org/) and, for Python, [sf2utils](https://pypi.org/project/sf2utils/). [Polyphone](https://www.polyphone.io/) edits packs and keeps the records.
- RFC 1321 for MD5, and de Cheveigné and Kawahara (2002) for the YIN estimator.

## Implementations

[PCSX-Redux](https://github.com/pcsx-redux/pcsx-redux) writes packs from its SPU Samples window (Export SoundFont...) and plays them (Load replacements...). Sample records are written and read from [pcsx-redux#2685](https://github.com/pcsx-redux/pcsx-redux/pull/2685) on.

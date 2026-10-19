# Kuroko Format Specification

This is the English translation of the Japanese specification [kuroko.ja.md](kuroko.ja.md). If the two differ, the Japanese version is authoritative.

- Document version: **1.0 (draft)**
- Target layout: `version` **5** of `KUROKO.MANIFEST` (differences from 3 and 4 are in Appendix C)
- Status: **Draft**. The content may change (changes are recorded in CHANGELOG.md).

> **Rights**
>
> - This document is provided under the [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
>   license. You may freely copy, modify, and redistribute it, including for commercial purposes, provided you give attribution.
> - **Anyone may implement reading and writing of the Kuroko format free of charge.** No permission or notification is required to implement it.
> - **The author will not assert any of the author's patents against implementing or using the Kuroko format.**
> - Implementations that claim compatibility under the name "Kuroko" are expected to follow the provisions of this document.

---

## 0. About This Document

### 0.1 Normative and Informative Content

Each section of this document consists of **normative** content (what the format requires) and **informative** content (explanations, implementation notes, and examples that aid understanding).
Sections and items marked "(informative)" are informative; everything else is normative. All appendices are informative.

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in
BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals, as shown here.

| Key word | Meaning |
|---|---|
| **MUST** / **MUST NOT** | Absolute requirement / prohibition |
| **SHOULD** / **SHOULD NOT** | May be deviated from when there is a valid reason, but only with an understanding of the consequences |
| **MAY** | Optional |

### 0.2 Roles

- **Writer**: something that creates Kuroko files (an authoring tool, a converter, etc.).
- **Reader**: something that reads Kuroko files (a player, a validator, etc.).
- **Player**: a reader that presents video, audio, and subtitles.

### 0.3 Notation

- `…` in JSON examples denotes an omission. Values in examples are for illustration and are not defaults.
- Terms such as "segment", "part", and "title" are defined in Section 2.

---

## 1. Purpose and Design Philosophy (informative)

Kuroko is a multimedia container format that stores video, audio, and subtitles in the standard Matroska (mkv) container,
while a **controller** provides **how playback proceeds** (the semantics of navigation).
The name comes from the image of a kuroko (a black-clad puppeteer in Japanese theater) operating Matroska (a matryoshka doll).

- Extension: **`.krk`**
- Controllers: **`none`** (plain playback without navigation; Part I), **`dvd-video-vm`**, which adds a DVD-Video-compatible
  virtual machine on top of it (Part II), and **`kuroko-tiny`**, which declares menus, buttons, and actions in JSON (Part III)

Kuroko has the following goals.

1. **Achieve high compression while faithfully preserving the video cadence.**
   MPEG-2 could express interlaced video and 24p soft telecine (3:2 pulldown) through flags such as `picture_structure`,
   `repeat_first_field`, `top_field_first`, and `progressive_frame`. For H.264, H.265, and AV1, encoders and tools that preserve
   this faithfully are not common; re-encoding tends to bake the pulldown into hard telecine or to deinterlace the video, losing
   the original display cadence. Kuroko carries telecine and interlace information as **metadata and container
   timestamps**, so that the original cadence can be played back independently of the encoder.
   It can also reproduce **video whose cadence changes midway**. With per-frame flags, MPEG-2 could represent video in which
   interlaced footage (scenes shot on video, dissolves, burned-in captions, etc.) is inserted in the middle of a 24p film
   (3:2 pulldown) feature, or video whose pulldown cycle breaks midway. Kuroko carries **the display duration of each frame in the
   timestamps and the interlaced intervals as time intervals** (Section 10.2), so the same transitions can be reproduced as-is
   whether the video is encoded with AV1, H.264, or H.265.
2. **Enable multimedia with menus through controllers.**
   By abstracting how playback proceeds as a controller, interactive content with menus, branching, and chapters can be expressed.
3. **Carry over the information of existing material with as little loss as possible.**
   Chapters, multiple audio streams (languages, dual mono, multichannel), subtitles, program information, menus, etc. are carried without omission.
   How to present them is left to the player, but the format preserves the information.
4. **Make the structure visible.**
   No proprietary multiplexing is used; content is held in a standard container (mkv) and structured JSON. General-purpose tools
   can be used to inspect, validate, and create it.
5. **Keep audio at full range and apply loudness control optionally at playback time.**
   Dynamic range compression and dialogue level normalization are not baked in but held as metadata.

This is achieved by separating three layers.

1. **Shell**: ZIP. The contents can be viewed with standard OS features and existing tools.
2. **Payload**: Matroska. Frame boundaries, codec initialization data, timestamps, and seek indexes are delegated to it.
3. **Controller**: JSON that states explicitly "how to traverse and play".

`none` is defined first as the minimal framework, and interactive features such as menus and virtual machines are added by
placing a controller on top of the `none` structure. In addition, a main feature shared by multiple titles is **stored only once**,
and its boundaries can be played back **seamlessly**.

---

## 2. Terminology

| Term | Meaning |
|---|---|
| Kuroko file | A `.krk` file conforming to this document. |
| Controller | The kind of logic that drives playback. Indicated by `controller` in `KUROKO.MANIFEST`. |
| Title | A unit of playback. It has a name, audio, subtitles, and chapters, and refers to its content by a sequence of segments (a playlist). |
| Segment | A physical unit of media (`SEGMENTS/S####/`). It holds the actual video, audio, and subtitle data and their attributes (`seg.json`). |
| Part | A file unit into which a segment is further divided and encoded independently (Section 10.4). |
| Logical track | An audio or subtitle stream that a title refers to by number ("audio 1", etc.). |
| Physical track | An audio or subtitle stream within the actual data of a segment. Linked to a logical track by `source` (Section 12.2). |
| Cadence | How video is displayed (progressive, interlaced, soft telecine). |
| Time | Unless otherwise stated, a number in milliseconds (fractions allowed). |

---

## 3. Container

A Kuroko file is a **ZIP** file (only the extension is `.krk`).

- The writer MUST make **the first entry `KUROKO.MANIFEST`**. This entry MUST be **uncompressed (Stored)**.
  A reader can identify a file as Kuroko just by looking at its beginning, without extracting it.
- The writer MUST store **media files (`.mkv`, `.spu`, `.pgs`, images) uncompressed (Stored)**.
  This avoids recompression and allows readers to seek directly to positions within the file.
- Text such as JSON MAY be compressed with Deflate.
- When an entry exceeds 4 GiB, or when the number of entries or their positions require it, **ZIP64** MUST be used.
  Readers MUST be able to read ZIP64.
- Encrypted ZIP MUST NOT be used.
- The path separator is `/`, and paths are case-sensitive. Paths in this document are relative to the ZIP root.
- Readers MUST ignore entries not defined in this document (to leave room for future additions).

---

## 4. KUROKO.MANIFEST

The entry point, which only identifies the format and the kind of controller. Written in JSON.

```json
{ "format": "kuroko", "version": 5, "controller": "none", "createdBy": "Example Authoring Tool 1.0", "comment": "…" }
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `format` | string | required | Fixed `"kuroko"`. Readers identify Kuroko by this. |
| `version` | integer | required | Layout generation (Section 6). |
| `controller` | string | required | Kind of controller. `"none"` (Part I), `"dvd-video-vm"` (Part II), `"kuroko-tiny"` (Part III). |
| `createdBy` | string | optional | Name of the creating tool, etc. |
| `comment` | string | optional | Free text. The writer MAY record the encoding conditions (codec, quality, keyframe interval, etc.). Environment-dependent information (paths of executables, etc.) SHOULD NOT be written. |

- `KUROKO.MANIFEST` SHOULD be **entirely ASCII** (informative: if a single non-ASCII character is mixed into otherwise all-ASCII
  text, a text editor may fail to detect UTF-8 and garble just that character).
- Readers **MUST refuse to play files with an unknown `controller`** (playing without knowing how playback proceeds would
  behave differently from what the author intended).

---

## 5. Common JSON Rules

- The character encoding is **UTF-8 without BOM**. Non-ASCII characters SHOULD be written as-is, not replaced by `\uXXXX`.
- Keys are **camelCase** (e.g. `pixelWidth`).
- Readers **MUST ignore unknown keys**.
- Items marked "optional" in this document MAY be omitted. The value when omitted follows the "Default" column of each table.
- **Times and durations are numbers in milliseconds** (e.g. `12345.678`). Their key names end in `…Ms`. The reference point (the position of 0) is defined for each item.
- A time interval (of the form `[{"startMs":…, "endMs":…}]`) includes `startMs` and excludes `endMs`.
- **Numbers and positions**:
  - **Numbers** (`number`, `id`, `pgn`, `pgc`, etc., which users and commands refer to like names) are **1-based**.
  - **Positions** (`index`, `startCell`, `entryCell`, `segCell`, `titleCellStart`, values of `programMap`, etc., which are locations within a sequence) are **0-based**.
  - Exception: cell command numbers (`commandNr`, `cellCommand`) are 1-based numbers because "0 = none".
- **Languages** are **ISO 639-2 three-letter lowercase codes** (bibliographic codes: `"jpn"`, `"eng"`, `"fre"`). Omit if unknown.
- **Palette colors** are 24-bit values written as **6-digit hexadecimal strings** (Section 17, Section 20).

---

## 6. Versions and Compatibility

### 6.1 Layout Generation (`version`)

| `version` | Content |
|---|---|
| **5** | The form defined in this document. Writers MUST write `5`. |
| 3, 4 | Older formats (informative). The structure is the same as 5, but the notation of numbers, positions, times, palettes, menu kinds, and languages differs (Appendix C). Readers are encouraged to be able to read them. |
| 1, 2 | Older formats (informative). The container is not ZIP, and they are outside the scope of this document. Readers need not read them. |

- Readers **SHOULD NOT open files with a `version` greater than what they know** (or, if they open them, they should inform the user).

### 6.2 Extension Rules

- When adding a new key, do so in a form where **playback does not break even if a reader that does not know it ignores it** (add it as an optional key).
  In this case, `version` need not be raised.
- When adding a new value to an enumeration (e.g. `video.codec`), follow the handling of "unknown values" defined for that item.
- The meaning of an existing key MUST NOT be changed. If it needs to change, add a key with a different name.

---

# Part I Controller `none`

`none` is the minimal framework of Kuroko and has no navigation. As **a set of named titles**, it defines only
"play video, audio, and subtitles in order with the correct cadence". Other controllers are added on top of this framework.

## 7. Layout

```
example.krk
├─ KUROKO.MANIFEST          controller: "none"
├─ CONTROL/disc.json        list of titles
├─ VIDEO/Tnn/title.json     title (logical): playlist, audio, subtitles, chapters, programs
└─ SEGMENTS/S####/          segment (physical)
   ├─ seg.json              media attributes
   ├─ av.mkv                video + audio (av.pNN.mkv when the segment has parts)
   ├─ loudness/<source>.json  gain changes for loudness control (optional; Section 11.2)
   └─ subs/                 subtitle data (Section 13)
```

- `Tnn` is the title number written in decimal with two or more digits (`T01`). `S####` is the segment identifier and
  SHOULD be `S` followed by four or more decimal digits (`S0001`). The actual paths are given by `disc.json` and `title.json`, so
  readers MUST NOT rely on the form of the names.
- Physical media is gathered under `SEGMENTS/`, and `title.json` holds only the logical structure. A single continuous piece of content is
  represented as "a playlist of one segment".

## 8. CONTROL/disc.json

```json
{
  "titles": [ { "id": 1, "name": "Main Feature", "durationMs": 1800000, "video": "VIDEO/T01" } ],
  "firstPlayback": { "title": 1, "angle": 1, "audio": 1, "subtitle": 0 }
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `titles` | array | required | List of titles. The order is the display order. |
| `titles[].id` | integer | required | Title number (1 or greater, unique within the file). |
| `titles[].name` | string | optional | Title name. |
| `titles[].durationMs` | number | optional | Title duration (for display in the list; the authoritative value is `timing.totalMs` in `title.json`). |
| `titles[].video` | string | required | Title directory (containing `title.json`). |
| `firstPlayback` | object | optional | The state in which playback starts (Section 8.1). |

### 8.1 Initial Playback State (`firstPlayback`)

Specifies the title, angle, audio, and subtitle used when playing the file from the beginning.

| Key | Meaning | When omitted or out of range |
|---|---|---|
| `title` | Starting title (`titles[].id`) | The first title in the list |
| `angle` | Angle number (1-based; Section 12.4) | 1 |
| `audio` | **Number** of the logical audio track (`audio[].number` in `title.json`) | The first audio track |
| `subtitle` | **Number** of the logical subtitle track (`subtitle[].number` in `title.json`). **0 = no subtitles** | No subtitles |

- A player MUST revert to the default only those items whose specification conflicts with the actual content (a nonexistent title, a number exceeding the angle count, etc.),
  and MUST NOT refuse playback.
- When resuming playback from where it was last left off (resume), the resume state MAY take precedence. `firstPlayback` is the default for "from the beginning".
- When the specified logical audio track is dual mono, playback SHOULD start with the main audio.
- For handling in `dvd-video-vm`, see Section 15.2.

## 9. SEGMENTS/S####/seg.json

`seg.json` is **the authoritative source of the segment's media attributes**. A segment in `none` is a single continuous unit of playback
with no internal divisions (cells).

```json
{
  "video": { "codec": "av1", "mode": "transcode",
             "pixelWidth": 1440, "pixelHeight": 1080, "displayWidth": 1920, "displayHeight": 1080,
             "aspect": "16:9", "frameRate": "30000/1001",
             "cadence": "interlaced", "tff": true, "interlaceMode": "weave", "fieldLayout": "single",
             "interlacedRanges": [ { "startMs": 0, "endMs": 30030 } ] },
  "audio": [
    { "startMs": 0, "endMs": 12635, "tracks": [
        { "track": 1, "source": 1, "codec": "opus", "channels": 2, "dualMono": true, "name": "Japanese\nEnglish", "language": "jpn" } ] },
    { "startMs": 12635, "endMs": 30030, "tracks": [
        { "track": 1, "source": 1, "codec": "opus", "channels": 2, "language": "jpn" },
        { "track": 2, "source": 2, "codec": "opus", "channels": 2, "name": "Commentary", "language": "jpn" } ] }
  ],
  "subtitle": [ { "source": 1, "track": 0, "type": "text" } ],
  "timing": { "totalMs": 30030 }
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `video` | object | required | Video attributes (Section 10). |
| `parts` | array | optional | Parts (Sections 10.4 and 10.5). When omitted, a single `av.mkv`. |
| `audio` | array | required | Changes of the audio configuration over time (Section 11). An empty array if there is no audio. |
| `subtitle` | array | required | List of the physical subtitle tracks this segment has (Section 13). An empty array if none. |
| `subtitle[].source` | integer | required | Stable key of the subtitle (Section 12.2). |
| `subtitle[].type` | string | required | `"text"` or `"image"`. |
| `subtitle[].track` | integer | optional | Informative value. Subtitles are not put into the mkv, so readers do not use it. |
| `timing.totalMs` | number | required | Segment duration (milliseconds). |

### 9.1 Media File (`av.mkv`)

- Video and audio are placed in a single Matroska file. **Subtitles MUST NOT be included** (subtitle data goes in `subs/`; Section 13).
- Timestamps are **0-based within the segment**. The first frame MUST be one from which decoding can start (a keyframe).
- Tracks are ordered **video first, then audio**. The number of video tracks is normally 1, or 2 with `fieldLayout:"dual"` (Section 10.3).
  The audio `track` is the track number within this mkv (0-based, counting video tracks as well).
- The Matroska timestamp precision SHOULD be **1 microsecond** (`TimestampScale` = 1000)
  (informative: to bake in per-field times; 1 millisecond cannot represent times at 59.94 fields per second).

---

## 10. Video

### 10.1 Video Attributes (`video` in `seg.json`)

| Key | Type | Required | Meaning | Default |
|---|---|---|---|---|
| `codec` | string | required | `"h264"`, `"h265"`, `"av1"` | `"h264"` |
| `mode` | string | optional | `"transcode"` (re-encoded) or `"copy"` (original bitstream as-is) | — |
| `pixelWidth`, `pixelHeight` | integer | required | Size of the encoded pixels. With `interlaceMode:"fields"`, the height is half the original (Section 10.3). | — |
| `displayWidth`, `displayHeight` | integer | required | Display size (after applying the pixel aspect ratio). | — |
| `aspect` | string | required | Display aspect ratio. `"16:9"`, `"4:3"`, etc. | — |
| `frameRate` | string | required | Frame rate of the original material (rational; `"30000/1001"`). | — |
| `cadence` | string | required | `"progressive"`, `"interlaced"`, `"soft-telecine"` (Section 10.2) | — |
| `tff` | boolean | optional | Field order. `true` = top field first. | `true` |
| `interlaceMode` | string | optional | Storage method for interlaced video. `"weave"`, `"fields"` (Section 10.3) | `"weave"` |
| `fieldLayout` | string | optional | Storage layout for `fields`. `"dual"`, `"single"` (Section 10.3) | `"single"` |
| `interlacedRanges` | array | optional | Intervals that are actually interlaced (Section 10.2). | No intervals |

- **Unknown `codec`**: readers MUST NOT attempt to decode the video of that segment, and SHOULD inform the user that it is
  not supported (informative: decoding it as a different codec produces corrupted video).
- In an `"av1"` mkv, the codec initialization data (`av1C`) MUST be placed in CodecPrivate.
- For `"h264"` and `"h265"`, **parameter sets** (SPS and PPS for H.264; VPS, SPS, and PPS for H.265) MUST be placed in the stream
  **before each keyframe** (`repeat-headers` in x264/x265). Readers can start decoding from any keyframe without relying on
  CodecPrivate (informative: this avoids reinitializing the decoder for seeking, concatenating segments, and concatenating parts with
  different resolutions). Writers SHOULD also place correct data in CodecPrivate (for general-purpose players).
- Reserved: `fieldTiming` (explicit number of displayed fields per frame) and `color` (color gamut, transfer characteristics, HDR static metadata) are
  reserved for the future. Writers MUST NOT use these names with a different meaning.

### 10.2 Cadence

A player reproduces the original display cadence as follows. **Display times are baked into the mkv timestamps**
(including the display duration of each soft telecine frame). The metadata is used to decide "which frames to deinterlace".

- **`"progressive"`**: displayed as-is.
- **`"interlaced"`**: interlaced material (e.g. 59.94i, 50i). Frames within the `interlacedRanges` intervals are
  deinterlaced by the player (bob, etc.) and displayed at the field rate.
- **`"soft-telecine"`**: 24p film carried with 3:2 or similar pulldown. Kuroko **keeps the 24p frames** and
  **bakes the display duration of each frame** (2 fields or 3 fields under the pulldown) **into the mkv timestamps**.
  This reproduces the original 24p and the telecined display without baking in hard telecine.
  If the film contains actual interlaced intervals (dissolves, etc.), those intervals are written in `interlacedRanges`.

#### Video Whose Cadence Changes Midway (informative)

The cadence MAY change any number of times within a single segment. The following mechanisms are combined to express, independently of the codec,
the transitions that the MPEG-2 per-frame flags (`progressive_frame`, `repeat_first_field`, `top_field_first`) could express.

| What MPEG-2 expressed | How Kuroko expresses it |
|---|---|
| Per-frame display duration (2 fields or 3 fields of 3:2 pulldown, breaks in the cycle) | The mkv timestamp of each frame |
| Which frames are interlaced (video scenes within film, dissolves) | The `interlacedRanges` intervals |
| Field order (`top_field_first`) | `tff` |
| When the storage height should differ between film and interlaced (`fields`) | Parts (Section 10.4) |

For example, in video where a 59.94i video scene is inserted in the middle of a 24p film feature, the film portion is stored as
24p frames with display durations according to the pulldown, the video portion is stored as interlaced frames at 29.97 frames per second, and the times of
the video portion are written in `interlacedRanges`. The player displays the film portion as-is and the video portion deinterlaced.
Encoding is done as a sequence of progressive frames, so any of AV1, H.264, or H.265 may be used.

`interlacedRanges` is a sequence of **intervals of frames that are actually interlaced and should be deinterlaced** (times
0-based within the segment). Frames outside the intervals MUST NOT be deinterlaced (deinterlacing a progressive picture
lowers the resolution and causes flicker). Omission or an empty array means "no intervals".

### 10.3 Interlaced Storage Methods

- **`"weave"`**: the top and bottom fields are **woven into one (full-height) picture** and encoded. H.265 and AV1 have no
  field coding tools, so the woven picture is encoded as progressive (repacking the fields is lossless).
  H.264 MAY use field coding (PAFF, etc.). The player splits alternate lines into top and bottom and deinterlaces.
- **`"fields"`**: the top and bottom are each encoded as **a progressive frame of half the height**.
  `pixelHeight` is half the original. For interlaced video with motion, comb artifacts disappear and compression is more effective than `weave`.
  The player restores each half-height frame as one field to full height for display. The storage layout is indicated by `fieldLayout`.
  - **`"dual"` (recommended)**: top and bottom are split into **two video tracks**. **Track 0 = top (even lines),
    track 1 = bottom (odd lines)**. Each track is at the original frame rate, and **the two fields of the same original frame have
    the same timestamp**. Top and bottom are determined by the track number, so they cannot be mixed up. Audio starts at track 2.
  - **`"single"`**: a single track contains top, bottom, top, … alternately at half height and double frame rate.
    Top and bottom are inferred from the timestamp phase, so they may be mixed up at boundaries. New files SHOULD use `"dual"`.
- **`fields` is used only for actually interlaced intervals.** Soft telecine frames do not have a constant display duration, so
  they cannot be represented with `fields`. Soft telecine frames MUST be stored at full height.
- When film and actual interlaced video are mixed within the same segment, it MAY be split into **parts (Section 10.4)**, storing
  the interlaced parts as `fields` and the film parts at full height. The entire mixture MAY also be stored as `weave`.

### 10.4 Parts (`parts`)

When the frame height (resolution) or storage method changes within a single segment, the segment is divided into
**parts** rather than into separate segments (informative: a single encoded stream cannot carry two resolutions; segments are not split
because the logical structure of titles is held per segment).

```json
"parts": [
  { "file": "av.p00.mkv", "durationMs": 92500,  "pixelHeight": 480, "interlaceMode": "weave",  "fieldLayout": "single" },
  { "file": "av.p01.mkv", "durationMs": 845000, "pixelHeight": 240, "interlaceMode": "fields", "fieldLayout": "dual" }
]
```

| Key | Type | Required | Meaning | Default |
|---|---|---|---|---|
| `file` | string | required | Part file (path relative to the segment directory). | — |
| `durationMs` | number | required | Part duration. Determines the position within the segment (the concatenation offset). | — |
| `pixelHeight` | integer | optional | Informative value. Encoded height. | — |
| `interlaceMode` | string | optional | Storage method of this part. | `"weave"` |
| `fieldLayout` | string | optional | Storage layout of this part. Determines the number of video tracks (2 for `"dual"`). | `"single"` |
| `still` | boolean | optional | Still part (Section 10.5). | `false` |

- When `parts` is present, `av.mkv` is not used. Readers MUST concatenate the parts in order, offsetting each by **the sum of the
  `durationMs` of the preceding parts**, and play them as a single segment.
- The timestamps of each part are 0-based within the part.
- The track configuration is constant within a part and MAY differ between parts (a `"dual"` part has 2 video tracks + audio;
  others have 1 video track + audio).
- Audio SHOULD be **present without gaps** throughout the `durationMs` of each part. Audio extending beyond the end of a part
  MUST be truncated by the reader at the part boundary.

### 10.5 Still Parts (`still`)

A part whose **video duration is much shorter than `durationMs`** is marked with `still: true`. A typical case is "one video frame + a few seconds of audio"
(an interval where only audio plays over a still image).

- The time a part occupies is always determined by `durationMs`. The player MUST **keep displaying the last video frame until the end of `durationMs`**,
  and MUST play the audio for the full length of the part.
- After feeding the video of a still part, the player MUST **immediately drain** the frames held by the decoder
  (informative: decoders may hold frames for reordering and not output them until the next input arrives; Appendix A.1).
- The end of a title or segment SHOULD be determined by waiting not only for the end of the video but **until audio playback has finished**.

---

## 11. Audio

### 11.1 Audio Configuration (`audio` in `seg.json`)

`audio` is **a sequence of time intervals (spans)**. The `tracks` of each span represent **the entire set of audio selectable in that interval**.
This expresses changes over time in the number of audio tracks, their names, and dual mono, such as at program changes in broadcasts.

| Key | Type | Required | Meaning | Default |
|---|---|---|---|---|
| `startMs`, `endMs` | number | required | Interval of the span (0-based within the segment). | — |
| `tracks` | array | required | Audio tracks in this interval. | — |
| `tracks[].track` | integer | required | Track number within the mkv (Section 9.1). | — |
| `tracks[].source` | integer | required | Stable key linking to the logical track (Section 12.2). | — |
| `tracks[].codec` | string | required | `"opus"`, `"ac3"`, `"dts"`, `"pcm"`, `"mp2"`, `"aac"`, etc. | — |
| `tracks[].channels` | integer | required | Number of channels. | — |
| `tracks[].dualMono` | boolean | optional | Whether it is dual mono (L = main, R = sub) in this interval. MAY be true only when `channels` is 2. | `false` |
| `tracks[].name` | string | optional | Audio name. Multilingual names are separated by a newline (`"Japanese\nEnglish"` = main, sub). | — |
| `tracks[].language` | string | optional | Language (ISO 639-2 three-letter code). | — |
| `tracks[].loudness` | object | optional | Loudness control (Section 11.2). | No control |

- Span boundaries are the points where the configuration (count, names, dual mono) of any track changes.
- A track that has no actual data in an interval is not included in the `tracks` of that interval's span.
- **Audio whose channel count changes midway** (e.g. stereo at the opening and 5.1ch in the main feature) MUST be placed in separate mkv tracks per interval
  and referred to separately by spans (informative: in Matroska the channel count is a track attribute, and for Opus it is also
  in the initialization data, so it cannot be changed midway through a single track). Each track has data only in the intervals it covers.
- A player SHOULD present the `tracks` of the span containing the currently presented time as the audio choices.
- For a dual mono track, the player SHOULD allow selecting main (L), sub (R), or main + sub separately.
  `dualMono` on a track whose `channels` is not 2 MUST be ignored.
- Audio SHOULD be stored at full range. Multichannel (5.1, 7.1) MAY be preserved.

### 11.2 Loudness Control (`loudness`)

Audio **samples are stored at full dynamic range**, and dynamic range control (DRC) and dialogue level normalization are
**not baked in but held as metadata**, to be applied optionally at playback time.

```json
"loudness": { "type": "ac3", "dialogLevelDb": -27.0, "file": "loudness/1.json" }
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `type` | string | required | Control type. `"ac3"` (11.2.1), `"dts"` (11.2.2). Determines the meaning of the values and how they are applied. |
| `dialogLevelDb` | number | optional | Dialogue level for the `"ac3"` type (11.2.1). |
| `dialogGainDb` | number | optional | Dialogue level for the `"dts"` type (11.2.2). |
| `file` | string | optional | File of gain changes (envelope) (path relative to the segment directory). If absent, there is no envelope. |

- **The type is independent of the stored codec.** For example, a track re-encoded from AC-3 to Opus MAY carry `"ac3"`-type control.
- When adding a new type, define the value range and how it is applied, and give it a name. The meaning of existing types is not changed.

**Envelope file** (SHOULD be placed at `loudness/<source>.json`):

```json
{
  "type": "ac3",
  "drc":   { "mode": "ac3-line", "points": [ [0, 0.0], [5.333, -3.5], [10.667, -6.0], [16.0, 0.0] ] },
  "drcRf": { "mode": "ac3-rf",   "points": [ … ] }
}
```

- `type`: the same type name as in `seg.json` (also written here so that the file can be read on its own).
- `points`: **a sequence of change points** `[time ms, gain dB]`. The gain of each point is **constant until the next point** (stepwise). Times take
  **the start of that track's audio as 0**. Consecutive identical values SHOULD be merged into one point. An envelope consisting only of 0 dB MAY be omitted.

#### 11.2.1 `"ac3"` Type

- **Dialogue level**: `dialogLevelDb`. It is **a measured value, not a gain to apply**, representing
  **the dialogue level of that track** (the amount below 0 dBFS; a negative value).
- **Envelopes**: `drc` (`mode: "ac3-line"`; normal compression; −24 to +24 dB), `drcRf` (`mode: "ac3-rf"`; stronger compression; optional).

#### 11.2.2 `"dts"` Type

- **Dialogue level**: `dialogGainDb`. **A gain applied directly to the decoder output** (a negative value; 0 = no normalization).
  It has a different meaning from `dialogLevelDb` of the `"ac3"` type (which is a measured value). The two MUST NOT be confused.
- **Envelope**: `drc` (`mode: "dts"`; embedded dynamic range coefficients).

#### 11.2.3 Application at Playback

- A player MAY apply loudness control **optionally**. **The default SHOULD be not to apply it (full range).**
  Unknown `type`s MUST NOT be applied.
- When applying, the gain is multiplied into the decoded PCM (`× 10^(dB/20)`). The times of `points` are mapped to sample positions,
  and a constant gain is applied per interval.
- **Dynamic range** (both types): `applied_dB = drc_dB × s` (`s` is a strength from 0 to 1). For the `"ac3"` type, when stronger compression
  is needed, use `drcRf` instead of `drc`.
- **Dialogue level** (**the method of application is opposite depending on the type**):

  | Type | How to apply |
  |---|---|
  | `"ac3"` | Choose a target dialogue level `targetDb`, and `applied_dB = targetDb − dialogLevelDb`. Example: to bring `dialogLevelDb = −27` to `targetDb = −31`, −4 dB. **The value of `dialogLevelDb` itself MUST NOT be applied.** |
  | `"dts"` | `applied_dB = dialogGainDb`. Applied as-is. |

---

## 12. VIDEO/Tnn/title.json

A title holds **only the logical structure** (playlist, logical tracks, chapters, programs), and refers to the actual media under `SEGMENTS/`.
It **MUST NOT hold media attributes (codec, channel count, dual mono, resolution, cadence)** (all of these are in `seg.json`).

```json
{
  "id": 1, "name": "Main Feature",
  "timing": { "totalMs": 1800000 },
  "audio": [ { "number": 1, "language": "jpn", "source": 1 } ],
  "subtitle": [ { "number": 1, "type": "text",  "language": "jpn", "name": "Subtitles", "source": 1 },
                { "number": 2, "type": "image", "language": "jpn", "name": "Subtitles (image)", "source": 2 } ],
  "chapters": [ { "number": 1, "startMs": 0, "name": "Opening" } ],
  "events": [ { "startMs": 0, "endMs": 1800000, "name": "Program Name", "eventId": 1024,
                "startTime": "2026-01-01T19:00:00+09:00", "desc": "…" } ],
  "playlist": [ { "seg": "S0001", "startMs": 0 } ]
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `id` | integer | required | Title number (`titles[].id` in `disc.json`). |
| `name` | string | optional | Title name. |
| `timing.totalMs` | number | required | Title duration. |
| `audio` | array | required | Logical audio tracks (Section 12.2). `number` (1-based), `language` (optional), `source`. |
| `subtitle` | array | required | Logical subtitle tracks. `number` (1-based), `type` (`"text"` / `"image"`), `language`, `name` (optional), `source`. |
| `chapters` | array | optional | Chapters. `number`, `startMs` (time within the title), `name` (optional). |
| `events` | array | optional | Timed program information (Section 12.3). |
| `playlist` | array | required* | Sequence of segments (Section 12.1). *A title with angles uses `angles` (Section 12.4). |
| `angleType`, `angles` | | optional | Angles (Section 12.4). |

### 12.1 Playlist

- `playlist[].seg`: segment identifier (`SEGMENTS/<seg>/`).
- `playlist[].startMs`: start time of that segment on the title's timeline.
- A player MUST add `startMs` to the times of each segment's video, audio, and subtitles to arrange them on the title's timeline.
- Multiple titles MAY refer to the same segment (sharing).

### 12.2 Logical and Physical Tracks (`source`)

`source` is **a stable key** linking logical tracks and physical tracks.

1. Obtain `audio[number = N].source` of the logical audio track.
2. From the `tracks` of the span containing the presentation time in the segment being played, select the track with the same `source`.
3. Its `track` is the track within the mkv.

- Writers MUST use **the same `source` for tracks with the same content in all segments**.
  This way, even if the configuration changes at segment boundaries or program changes, the title does not need to be rebuilt.
- The same applies to subtitles (`subtitle[].source` → `subtitle[]` in `seg.json` → the data in `subs/`; Section 13).
- Informative: when creating from a broadcast TS, the original stream (equivalent to a PID) is suitable as `source`; when creating from a DVD, the physical substream number is.

### 12.3 Program Information (`events`)

Represents content whose **display name changes** depending on the playback position (such as program changes in a broadcast recording).

| Key | Type | Required | Meaning |
|---|---|---|---|
| `startMs`, `endMs` | number | required | Interval (time within the title). |
| `name` | string | optional | Program name. |
| `eventId` | integer | optional | Program identifier. |
| `startTime` | string | optional | Broadcast start time (ISO 8601). |
| `desc` | string | optional | Description of the content. |

- A player SHOULD determine the display name in the order "the `name` of the `events` entry containing the presentation time → if none, the title's `name`".

### 12.4 Angles (`angleType: "bd"`)

**Alternative versions** of the same content are arranged within one title. **Only the playlist differs per angle**;
duration, chapters, audio, and subtitles are common to the title.

```json
{
  "id": 2, "name": "Main Feature",
  "angleType": "bd",
  "angles": [
    { "number": 1, "playlist": [ { "seg": "S0001", "startMs": 0 }, { "seg": "S0005", "startMs": 120245 } ] },
    { "number": 2, "playlist": [ { "seg": "S0003", "startMs": 0 }, { "seg": "S0005", "startMs": 120245 } ] }
  ],
  "timing": { "totalMs": 8170695 },
  "audio": [ … ], "subtitle": [ … ], "chapters": [ … ]
}
```

- **`angles` and `playlist` are mutually exclusive.** A title that has `angles` MUST NOT have `playlist`.
- `angles[].number`: a 1-based serial number. `angles[].name`: an optional name.
- `angleType`: `"bd"` (this section; each angle is a separate playlist for the whole title), `"dvd"` (Section 18 of Part II).
  **For an unknown value, the player MUST play only angle 1.**
- Writers MUST confirm that **duration, chapters, audio, and subtitles are the same for all angles** before grouping them as angles.
  When this does not hold, write them as separate titles.
- Player behavior (informative): when switching to another angle of the same title, switch while keeping the position, audio, and subtitle.
  Resume keeps (title, angle, position, audio, subtitle).

---

## 13. Subtitles

Subtitles are **placed physically in `SEGMENTS/S####/subs/`** and referenced from `subtitle[].source` in `title.json`
(`subtitle[number].source` → `subtitle[]` in `seg.json` → `subs/<kind>/<source>`). They are not put into the mkv.
The kinds of subtitles are the **text type** (13.1, 13.2) and the **image type** (13.3).

### 13.1 Text Type (`subs/text/<source>.json`)

Timed text, written as lines and runs of characters with the same formatting.

```json
{ "type": "text", "lang": "jpn", "name": "Subtitles",
  "charset": "unicode",
  "canvas": { "width": 960, "height": 540 },
  "cues": [ { "startMs": 38810, "endMs": 41710,
              "lines": [
                { "x": 438, "y": 413,
                  "runs": [ { "text": "東京", "w": 36, "h": 36, "hs": 4, "vs": 24, "color": "#00FF00" },
                            { "text": "目黒区の自由が丘｡", "w": 36, "h": 36, "hs": 4, "vs": 24, "color": "#00FF00", "size": "medium" } ] },
                { "x": 558, "y": 395,
                  "runs": [ { "text": "めぐろ", "w": 18, "h": 18, "hs": 2, "vs": 6, "color": "#00FF00", "size": "small" } ] } ] } ] }
```

| Key | Type | Required | Meaning | Default |
|---|---|---|---|---|
| `type` | string | required | `"text"`. | — |
| `lang` | string | optional | Language. | — |
| `name` | string | optional | Name. | — |
| `charset` | string | optional | Rules for characters and rendering (Section 13.2). | `"unicode"` |
| `canvas` | object | required | Coordinate system for positions (`width`, `height`). The player scales the whole canvas to the video display area and overlays it. | — |
| `drcs` | array | optional | Glyph shapes of external characters (Section 13.2). | — |
| `cues` | array | required | Units of display. | — |

**cue**:

- `startMs`, `endMs`: display interval (0-based within the segment). It **MUST be finite** (for a display with no end,
  the time of the next display or of the erase is put into `endMs` at import to make it finite).
- `writing`: `"horizontal"` (default), `"vertical"` (vertical writing).
- `lines`: sequence of lines.

**Line (`lines[]`)**:

- `x`, `y`: **the top-left of the character box (cell) of the first character** of the line (canvas coordinates). Characters are laid out from left to right (top to bottom for vertical writing).
- `runs`: sequences of characters with the same formatting.

**Run (`runs[]`)**:

| Key | Meaning |
|---|---|
| `text` | String (Unicode). |
| `w`, `h` | Character size (width and height of the reference character; values before shrinking by `size`). |
| `hs`, `vs` | Character spacing and line spacing. The box of one character is width `w+hs` × height `h+vs` (shrinks or stretches according to `size`). |
| `size` | `"normal"` (default), `"medium"` (1/2 horizontally), `"small"` (1/2 vertically and horizontally; ruby, etc.), `"tall"` (2× vertically), `"wide"` (2× horizontally), `"double"` (2× vertically and horizontally). |
| `color` | Character color. |
| `bg` | Character background color (fills the box). |
| `edge` | Outline color (no outline if absent). |
| `deco` | Sequence of `"underline"`, `"bold"`, `"italic"`. |
| `box` | Enclosure. A combination of `"t"`, `"b"`, `"l"`, `"r"` (`"tblr"` = all four sides). |
| `flash` | `"normal"`, `"inverted"` (flashing). The player need not render it. |

- Colors are `#RRGGBB`, or `#RRGGBBAA` including opacity.
- **An omitted item inherits the value of the immediately preceding run in the same line.** The first run of a line MUST specify
  `w`, `h`, `hs`, `vs`, and `color`. Other defaults are: `size` normal, and no background, outline, decoration, enclosure, or flashing.
  To return to "none" midway, state it explicitly: `bg:"#00000000"`, `edge:"none"`, `deco:[]`, `box:""`, `flash:"none"`, `size:"normal"`.

### 13.2 Rules for Characters and Rendering (`charset`)

- **`"unicode"`** (default): ordinary Unicode text. The player chooses the font.
- **`"arib"`**: indicates that the subtitles originate from **captions of Japanese digital terrestrial and BS broadcasting (ARIB STD-B24)**.
  - Characters are held as Unicode. ARIB additional symbols and additional kanji are written as their corresponding Unicode characters.
  - **DRCS** (external characters whose shapes are sent in the caption data for each program) are represented in `text` by
    **characters of the Supplementary Private Use Area (sequentially from U+F0000)**, and their shapes are held in `drcs[]`. Even for the same broadcast code, if the shape differs, a different character is used (`drcs[]` has one entry per shape).
  - Medium-size (`size:"medium"`) alphanumerics and symbols are written as half-width Unicode, as displayed.
  - The player SHOULD render with a font that includes the ARIB external characters, and SHOULD render DRCS with the shapes in `drcs[]`.
  - One display corresponds to one unit of caption data, and **is replaced by the next caption data (including erase-only data)**.
    At import, `endMs` is determined by this rule.
- **`drcs[]`**: `char` (the private-use character used in `text`; in the form `"U+F0000"`), `width`, `height` (dots),
  `depth` (bits per pixel; 1 = 2 levels, 2 = 4 levels), `pattern` (the ARIB DRCS pattern data as-is: a bit string packed in
  row order from the top-left, in base64).

### 13.3 Image Type (`subs/img/<source>/index.json`)

Image subtitles. Each cue has the following three representations, and **the player gives priority in the order `pgs` → `spu` → `steps`**
(representations it cannot interpret are skipped). Writers MUST write at least one.

```json
{ "type": "image", "lang": "jpn", "canvas": { "width": 720, "height": 480 },
  "cues": [ { "startMs": 12345, "endMs": 15000, "spu": "0001.spu" },
            { "startMs": 16000, "endMs": 18000,
              "steps": [ { "atMs": 0, "x": 100, "y": 420, "image": "0002.png", "opacity": 1.0 } ] } ] }
```

- **`spu`**: raw DVD-Video subpicture data (for one display) in the same directory. All changes during display
  (fades, color selection per line range) are contained within it. Colors are rendered with the title's `palette` (Section 17 of Part II).
- **`pgs`**: raw Blu-ray Presentation Graphic Stream data in the same directory. **From the Display Set that starts the display
  up to just before the erase** is put into one file. The palette is in the stream.
  - Known limitation (informative): display times per Display Set are not held. When one cue has multiple Display Sets
    (fades by palette updates, etc.), the player MAY decode them in order and show the final state for the entire cue.
- **`steps`**: generic images (PNG) and opacity. `atMs` (time from the start of the cue), `x`, `y` (canvas coordinates),
  `image` (a PNG in the same directory), `opacity` (0 to 1). There are no constraints such as on the number of colors.
- `canvas`: size of the coordinate system (the video display size).

---

## 14. What `none` Cannot Represent (informative)

- Dynamic HDR metadata (Dolby Vision RPU, HDR10+).
- Advanced text subtitles (ASS/SSA animation, karaoke, embedded fonts).
- Multiple simultaneous videos (picture-in-picture), data broadcasting, timed data (GPS, etc.).
- Navigation such as menus, branching, and operation restrictions (handled by controller extensions; Part II, Part III).

---

# Part II Controller `dvd-video-vm`

A controller adds a navigation layer on top of the `none` structure. `dvd-video-vm` expresses menus, branching, and operation restrictions with
**a DVD-Video-compatible virtual machine (VM)**. Segments (physical), `seg.json`, and subtitles use `none` as-is,
and menus, VM commands, and the logical-to-physical mapping are added on top.

- **This document does not define the semantics of VM commands.** Each command is an 8-byte DVD-Video navigation command (Section 21),
  and its semantics follow the DVD-Video virtual machine. This document defines only how commands are stored in the Kuroko structure.
  (Informative: the semantics of the commands can be learned from publicly available analysis documents and open-source DVD navigation implementations.)
- DVD-Video terms (VMG, VTS, PGC, cell, PCI, etc.) are used with their DVD-Video meanings.

## 15. Overview

### 15.1 Correspondence with DVD-Video (informative)

| DVD-Video | Kuroko | Notes |
|---|---|---|
| Disc (VMG) | `CONTROL/disc.json` | Region, FirstPlay, list of menus, VM commands |
| Title | `VIDEO/Tnn/title.json` | Referenced by serial number by VM JumpTT, etc. |
| Sequence of PGC cells / VOB | `SEGMENTS/` (deduplicated by sharing unit; Section 19) | |
| Audio (logical-to-physical mapping table) | `title.json` (logical → physical) + `seg.json` (physical → track) | |
| Subpicture (per display mode) | Image-type subtitles (Section 13.3) + per-display-mode mapping in `title.json` | 4:3, wide, letterbox, pan & scan |
| Menu (PGC, buttons) | `MENU/<domain>/PGC<nn>/` (Section 20) | Timed button information |
| VM commands (PRE, POST, CELL, FirstPlay) | Embedded in JSON (Section 21) | No separate binary |
| Operation restrictions / cell still / cell commands | `uopControl` / `still` / `commandNr`, `cellCommand` | |
| Still-image cell (one video frame + a few seconds of audio) | Still part (Section 10.5) | The cell duration is mapped to `durationMs` |
| 16-color palette | `palette` | Rendering of buttons and subtitles |

### 15.2 Playback With and Without Executing the VM

A player MAY play in either of the following ways.

- **Playback executing the VM**: executes FirstPlay, menus, PGC PRE and POST, cell commands, cell stills, and resume (RSM),
  all as in DVD-Video.
- **Playback without executing the VM** (playback as a media player): selects and plays titles directly. **It executes no VM commands
  at all** (including the POST at the end of a title, cell commands, and cell stills). Operation restrictions (UOPs) are not applied either.
  At the end of a title, it MAY proceed to the next title. The state in which playback starts follows `firstPlayback` (Section 8.1).

A file with `controller: "none"` is always played as "playback without executing the VM".

## 16. Extensions to CONTROL/disc.json

```json
{
  "region": 0,
  "firstPlay": { "domain": "vmg", "pgc": 0 },
  "firstPlayCommands": [ { "hex": "3006000000420000", "asm": "JumpSS VMGM menu=Title" } ],
  "firstPlayback": { "title": 1, "audio": 1, "subtitle": 0 },
  "titles": [ { "id": 1, "vts": 1, "vtsTitle": 1, "menuDomain": "VTS01", "durationMs": 4659633.9, "video": "VIDEO/T01" } ],
  "menuDomains": [
    { "id": "VMGM", "pgcs": [ { "pgc": 1, "type": "Title", "path": "MENU/VMGM/PGC01",
                                "commands": { "pre": [ … ], "post": [ … ], "cell": [ … ] }, "uopControl": 0 } ] }
  ],
  "titleDomains": [
    { "vts": 6, "pgcs": [ { "pgc": 1, "title": 7 },
                          { "pgc": 2, "title": 0, "commands": { "pre": [ … ], "post": [ … ], "cell": [ … ] } } ] }
  ]
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `region` | integer | optional | Region mask (the DVD-Video value). |
| `firstPlay` | object | optional | PGC executed at startup. `domain` (`"vmg"` or a menu domain), `pgc` (number; 0 = the FirstPlay PGC). |
| `firstPlayCommands` | array | optional | Commands of the FirstPlay PGC (Section 21). |
| `firstPlayback` | object | optional | Initial state for playback without executing the VM (Section 8.1). **Distinct from `firstPlay`.** |
| `titles[]` | array | required | In addition to the `none` items (Section 8): `vts` (VTS number), `vtsTitle` (title number within the VTS), `menuDomain` (the related VTS menu domain). |
| `menuDomains[]` | array | optional | Menu domains. `id` (`"VMGM"`, `"VTS01"`, …), `pgcs[]`. |
| `menuDomains[].pgcs[]` | | | `pgc` (number), `type` (`"Title"`, `"Root"`, `"Subpicture"`, `"Audio"`, `"Angle"`, `"Chapter"`, or absent), `path` (menu directory; Section 20), `commands` (PRE, POST, and CELL commands), `uopControl` (32 bits of prohibited operations; 0 = all allowed). |
| `titleDomains[]` | array | optional | Title domains (the VTS PGC table; Section 16.1). |

- `audio` and `subtitle` of `firstPlayback` are written as the `number` in `title.json`, as in `none`.

### 16.1 Title Domains (`titleDomains`)

The VM's Link-family commands (LinkPGCN, next, previous, and up PGC) refer to a PGC **within the current domain**. During title playback,
they MUST be resolved within the title domain of that VTS. The VTS PGC table may contain **intermediate PGCs that have no title number**
(such as ones that only jump elsewhere in PRE), so the following is held for each PGC number.

- `pgc`: PGC number within the VTS (1-based).
- `title`: the corresponding title number (`titles[].id`). **0 = an intermediate PGC with no title number.**
- `commands`: commands of an intermediate PGC (PRE, POST, CELL). Commands of a title's PGC are in `pgcCommands` of `title.json`, so they are not held here.

## 17. VIDEO/Tnn/title.json (`dvd-video-vm`)

The `title.json` of `dvd-video-vm` also holds **only the logical structure**. Media attributes are in `seg.json`.

```json
{
  "audio": [ { "number": 1, "source": 128, "language": "eng", "astCtl": 0 } ],
  "subtitle": [ { "number": 1, "source": 32, "spstCtl": 0, "language": "jpn",
                  "modes": { "normal": 32, "wide": 33, "letterbox": -1, "panScan": -1 } } ],
  "chapters": [ { "number": 1, "startCell": 0, "startMs": 0 } ],
  "playlist": [ { "seg": "S0001", "titleCellStart": 0, "cellCount": 1, "startMs": 0 } ],
  "programs": [ { "pgn": 1, "entryCell": 0 } ],
  "cells": [ { "index": 0, "startMs": 0, "still": 0, "commandNr": 0, "seg": "S0001", "segCell": 0, "angle": 0 } ],
  "palette": [ "108080", "EB8080", … ],
  "uopControl": 0,
  "pgcCommands": { "pre": [ … ], "post": [ … ], "cell": [ … ] },
  "timing": { "totalMs": 9647567.2, "frameRate": "30000/1001", "cellBaseFrame": [ … ], "chapterFrame": [ … ] }
}
```

| Key | Meaning |
|---|---|
| `audio[]` | Logical audio tracks (Section 17.1). |
| `subtitle[]` | Logical subtitle tracks (Section 17.2). |
| `chapters[]` | Chapters. `number` (1-based), `startCell` (0-based number of the starting cell), `startMs`. |
| `playlist[]` | Sequence of segments (Section 17.3). A title with angles uses `angles` (Section 18). |
| `programs[]` | Programs. `pgn` (1-based), `entryCell` (position of the starting cell; **0-based**). |
| `cells[]` | Cells (Section 17.3). |
| `palette` | 16-color subpicture palette. Each element is a 24-bit value as 6 hexadecimal digits (in the same Y, Cr, Cb order as DVD-Video). |
| `uopControl` | Operations prohibited by the title's PGC (32 bits; 0 = all allowed). |
| `pgcCommands` | Commands of the title's PGC (`pre`, `post`, `cell`; Section 21). |
| `timing.totalMs` | Title duration. |
| `timing.frameRate`, `cellBaseFrame`, `chapterFrame` | Informative values. Frame numbers, in display order, of the start of each cell and chapter (`cellBaseFrame` has the number of cells + 1 entries, the last being the total number of frames). |

### 17.1 Logical Audio

- `number`: logical audio number (1-based). **The DVD logical audio number (the 0-based number referenced by the VM command SetSTN and by SPRMs) is `number − 1`.**
- `source`: **the physical substream number** that the logical audio refers to. Linked to `tracks[].source` in `seg.json` (Section 12.2).
  Informative: AC-3 is `0x80+n`, DTS is `0x88+n`, LPCM is `0xA0+n`, MPEG audio is `0x1C0+n`.
- `astCtl`: the actual stream number referred to by the PGC's audio stream control (−1 if not available).
- The logical-to-physical mapping MAY differ per title (PGC) (the same physical audio MAY be referred to by multiple logical audio tracks).
  For this reason, the mapping is held on the title side.

### 17.2 Logical Subtitles (Per Display Mode)

- `number`: logical subtitle number (1-based). The DVD logical subpicture number is `number − 1`.
- `source`: the representative physical substream number (`0x20+n`).
- `spstCtl`: the actual stream number referred to by the PGC's subpicture control (−1 if not available).
- `modes`: **physical substream numbers per display mode** (`normal` = 4:3, `wide` = 16:9, `letterbox`, `panScan`;
  −1 for those absent). The player selects the physical number for the current display mode and renders `subtitle[]` in `seg.json` (the one with a matching `source`) →
  `subs/img/<source>/`.

### 17.3 Cells and Playlist

- `cells[]` corresponds **one-to-one with the PGC's cells** (because the VM, chapters, and programs refer to cells by number).
  - `index`: cell number (0-based).
  - `startMs`: start time on the title's timeline.
  - `still`: cell still time (seconds). 0 = none, −1 = infinite (waits for user input).
  - `commandNr`: number of the cell command executed at the end of the cell (a **1-based** index into `pgcCommands.cell`; 0 = none).
  - `seg`, `segCell`: the segment containing this cell's data, and the cell's order within it (0-based).
  - `angle`: order within the angle block (Section 18; 0 = outside a block).
- **The cell duration** is determined by the difference from the next cell's `startMs` (the title duration for the last cell). However, cells
  of other angles in the same angle block are skipped when counting (Section 18).
- `playlist[]`: `seg` (segment), `titleCellStart` (number of the first cell that the segment covers), `cellCount` (number of cells),
  `startMs` (start time of the segment on the title's timeline). The segment corresponds to the title's cells `[titleCellStart, titleCellStart+cellCount)`.
- The position of a cell within its segment is obtained as "the cell's `startMs` − the segment's `playlist[].startMs`".

### 17.4 Still-Image Cells (informative)

Some DVD-Video titles consist of a succession of still-image cells of "one to a few video frames + a few seconds of audio" (such as a series of warning screens). The cell
duration is held by the IFO and cannot be determined from the video stream. The writer makes such cells **still parts** (Section 10.5) and puts the IFO cell
duration into `durationMs` (Appendix B.1).

## 18. Angles (`angleType: "dvd"`)

DVD-Video multi-angle works **per cell**: cells shooting the same interval from different angles follow one another in the PGC's cell sequence
as an **angle block**. It is not, as in BD, "a separate playlist for the whole title".

```json
{
  "angleType": "dvd",
  "angles": [
    { "number": 1, "playlist": [ { "seg": "S0001", "titleCellStart": 0, "cellCount": 1, "startMs": 0 },
                                 { "seg": "S0004", "titleCellStart": 6, "cellCount": 60, "startMs": 38200 } ] },
    { "number": 2, "playlist": [ { "seg": "S0005", "titleCellStart": 1, "cellCount": 1, "startMs": 0 },
                                 { "seg": "S0004", "titleCellStart": 6, "cellCount": 60, "startMs": 38200 } ] }
  ],
  "cells": [ { "index": 0, "startMs": 0, "angle": 1, "seg": "S0001", "segCell": 0 },
             { "index": 1, "startMs": 0, "angle": 2, "seg": "S0005", "segCell": 0 },
             { "index": 6, "startMs": 38200, "angle": 0, "seg": "S0004", "segCell": 0 } ]
}
```

- **`cells[]` keeps the cells of all angles and corresponds one-to-one with the PGC** (thinning it out by angle would shift the numbers referred to by
  LinkCN, chapters, and programs).
- `cells[].angle`: order within the angle block = angle number (1-based). 0 = outside a block (played for all angles).
- **Cells in the same block share `startMs`** (each angle has the same duration, so the timeline does not depend on the angle).
- **Only `playlist` differs per angle.** Segments outside blocks appear with the same identifier in the `playlist` of every angle
  (the data is stored only once).
- Cells in an angle block MUST be one cell = one segment (the cells played for a given angle have non-consecutive numbers
  and cannot be expressed as a contiguous range).
- `timing`, `chapters`, `programs`, `audio`, `subtitle`, and `pgcCommands` are common regardless of angle.
- The player MUST keep the VM's angle number (SPRM 3) consistent with the current angle. Switching behaves as in Section 12.4.

## 19. Segment Sharing

A configuration in which multiple titles share a main feature and differ only in their introductions (seamless branching) MAY be divided into
**segments per shared range**, storing the data only once.

- The writer MAY group ranges at the same original physical position into the same segment.
- The player MUST concatenate the playlist's segments seamlessly. Video is presented continuously from the first keyframe of each segment,
  and audio is joined **with sample accuracy, taking the encoder delay (Opus pre-skip, etc.) into account**.
- The presentation time is `time within the segment + playlist[].startMs of the title`. **Negative timestamps MUST NOT be treated as "no timestamp"**
  (informative: because of Opus pre-skip, the first audio packet of a segment has a real time such as −6.5 ms).

## 20. Menus (`MENU/<domain>/PGC<nn>/`)

A menu holds a PGC split into **files per cell**.

```
MENU/<domain>/PGC<nn>/
├─ menu.json   menu information and the cell sequence
├─ C00.mkv …   video + audio per cell
└─ C00.spu …   button graphics per cell (raw subpicture data; optional)
```

### 20.1 menu.json

| Key | Type | Meaning |
|---|---|---|
| `version` | integer | The same value as the layout (`5`). |
| `domain` | string | `"VMGM"`, `"VTS01"`, etc. |
| `pgc` | integer | PGC number. |
| `menuType` | string | Menu kind. `"Title"`, `"Root"`, `"Subpicture"`, `"Audio"`, `"Angle"`, `"Chapter"` (the same as `menuDomains[].pgcs[].type` in `disc.json`). An empty string if none applies. |
| `uopControl` | integer | Prohibited operations (32 bits). |
| `nextPgc`, `prevPgc`, `goUpPgc` | integer | Numbers of the next, previous, and up PGCs (0 = none). |
| `palette` | array of strings | 16-color palette (each element is a 24-bit Y, Cr, Cb value as 6 hexadecimal digits; the same as `title.json`). |
| `video` | object | `codec` (`"h264"`, `"h265"`, `"av1"`), `width`, `height`, `aspect`, `interlaced`, `tff`. |
| `audio[]` | array | Audio stream control (in logical audio order). `available`, `number` (the stream number that logical audio refers to; the DVD-Video value), `codec` (`"opus"`, `"ac3"`, etc.). |
| `spstCtl[]` | array | Subpicture control (in logical number order). `available`, `n43`, `wide`, `lb`, `ps` (physical numbers per display mode). |
| `commands` | object | PGC commands (`pre`, `post`, `cell`). |
| `programMap` | array of integers | Position of the starting cell of each program (0-based). |
| `cells[]` | array | Cells (Section 20.2). |

### 20.2 Cells (`cells[]`)

| Key | Meaning |
|---|---|
| `index` | Cell number (0-based). |
| `still` | Still time (seconds). 0 = none, 1 to 254 = finite, −1 = infinite. |
| `cellCommand` | Number of the cell command executed at the end of the cell (1-based; 0 = none). |
| `durationMs` | Duration of this cell's mkv. |
| `seamlessNext` | Whether it continues seamlessly into the next cell. |
| `static` | Whether it is a still cell with only one actual frame. If true, the player SHOULD display it at full resolution without deinterlacing. |
| `cadence` | Cadence (`"interlaced"`, `"film"`, `"progressive"`, `"mixed"`). A record for human readers. |
| `interlacedRanges` | Intervals to be deinterlaced (**times within this cell**). An empty array = none. |
| `av` | This cell's video + audio file (`"C01.mkv"`). |
| `spu` | This cell's button graphics file (`"C01.spu"`; an empty string if none). |
| `pci[]` | Timed button information (Section 20.3). |

- The player concatenates and plays the `av` of the cells being played in order. The times of each cell are offset by the sum of the `durationMs` of the preceding cells.
- Stopping and looping are expressed by the cell's `still` (≠0) and `cellCommand` (≠0).
- When encountering a `menu.json` without `interlacedRanges`, the player MAY decide based on `video.interlaced` (informative: older writers).

### 20.3 Button Information (`pci[]`)

Each element is the highlight information of a DVD-Video PCI (the interval of one VOBU), valid only between `fromMs` and `toMs` (times within the cell).
A still cell has one element.

| Key | Meaning |
|---|---|
| `fromMs`, `toMs` | Valid interval. |
| `highlightStatus` | Highlight status (0 = no highlight). |
| `vobuUop` | Operations prohibited in this interval (the player decides by OR-ing it with the PGC's `uopControl`). |
| `forcedSelect`, `forcedActivate` | Number of the button to be forcibly selected / activated (0 = none). |
| `buttons[]` | `number` (1-based), `x0`, `y0`, `x1`, `y1` (rectangle), `colorGroup` (0 to 2), `autoAction` (if nonzero, executed immediately when selected), `up`, `down`, `left`, `right` (destination button numbers), `command` (button command; Section 21). |
| `colorGroups[]` | Color groups (up to 3). `selectColor`, `selectContrast`, `actionColor`, `actionContrast` (4 elements each; colors are palette indexes, contrasts are opacities). |

## 21. JSON Representation of VM Commands

VM commands are **embedded in JSON without a separate binary**. Command sequences appear in `firstPlayCommands`,
`menuDomains[].pgcs[].commands`, and `titleDomains[].pgcs[].commands` of `disc.json`, `pgcCommands` of `title.json`, and
`commands` and `cells[].pci[].buttons[].command` of `menu.json`. Each command takes the following form.

```json
{ "hex": "3006000000420000", "asm": "JumpSS VMGM menu=Title" }
```

- `hex`: the 8-byte command **to be executed** (16 hexadecimal digits).
- `asm`: an annotation for human readers. Readers MUST NOT use it for execution.
- Readers SHOULD also accept the string-only form (`"3006000000420000"`) as `hex` (informative: older writers).
- When creating another controller, its navigation can likewise be held in JSON.

---

# Part III Controller `kuroko-tiny`

`kuroko-tiny` is a controller that **declares menus, buttons, and actions in JSON**. It is intended to represent "multi-title content with menus
and subtitles" created with authoring tools, and has neither a command sequence (virtual machine) nor state variables.
It is not subject to DVD constraints (4-color button graphics, SD resolution, etc.), and can use high-resolution images, transparency, and video backgrounds.

## 22. Overview

### 22.1 Layout

The title structure is the same as `none` (`CONTROL/disc.json`, `VIDEO/Tnn/title.json`, `SEGMENTS/`, subtitles; Part I).
`kuroko-tiny` adds only the navigation file `CONTROL/nav.json` and a folder per menu, `MENU/<menuId>/`.

```
example.krk
├─ KUROKO.MANIFEST              controller: "kuroko-tiny"
├─ CONTROL/disc.json            list of titles (same as Part I)
├─ CONTROL/nav.json             navigation (Section 23)
├─ MENU/<menuId>/menu.json      one menu (Section 24)
├─ MENU/<menuId>/…              background images and videos, BGM, button images
├─ VIDEO/Tnn/title.json         title (same as Part I)
└─ SEGMENTS/S####/              segment (same as Part I)
```

### 22.2 Playback Following and Not Following the Controller

- In **playback following the controller**, the reader starts from `firstPlay` in `nav.json` and follows the menus and button actions (Section 26).
- In **playback as a media player**, the reader MAY play without using `nav.json` and `MENU/`, the same as `none`
  (selecting and playing titles directly).
- The reader decides which way to play (by user settings, etc.).

## 23. CONTROL/nav.json

```json
{
  "firstPlay": { "showMenu": "main" },
  "menus": [ "main", "chapters" ],
  "keys": { "menu": "main", "titleMenu": "chapters" },
  "titles": [
    { "title": 1, "onEnd": { "showMenu": "chapters" } },
    { "title": 2, "onEnd": { "playTitle": { "title": 3 } } }
  ]
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `firstPlay` | action | optional | Action when the file is opened (Section 25). If absent, the first title is played. |
| `menus` | array of strings | required | List of menu identifiers. For each identifier, `MENU/<menuId>/menu.json` MUST exist. |
| `keys.menu` | string | optional | Menu shown by the player's "Menu" operation. If absent, `keys.titleMenu`. |
| `keys.titleMenu` | string | optional | Menu shown by the player's "Title Menu" operation. If absent, `keys.menu`. |
| `titles[].title` | integer | required | Title `id` (`titles[].id` in `disc.json`). |
| `titles[].onEnd` | action | optional | Action when that title has been played to the end. |

- **Menu identifiers** MUST be 1 to 64 characters consisting only of alphanumerics, `-`, and `_` (because they become folder names).
- When a title not in `titles`, or a title without `onEnd`, ends: if `firstPlay` is an action that shows a menu,
  that menu is shown; otherwise playback ends.

## 24. MENU/<menuId>/menu.json

```json
{
  "id": "main",
  "canvas": { "width": 1920, "height": 1080 },
  "background": { "video": "bg.mkv", "loop": true },
  "defaultButton": "play",
  "rememberSelection": true,
  "timeout": { "afterMs": 60000, "action": { "playTitle": { "title": 1 } } },
  "buttons": [
    { "id": "play", "rect": { "x": 760, "y": 820, "w": 400, "h": 80 },
      "images": { "normal": "play_n.png", "selected": "play_s.png", "activated": "play_a.png" },
      "nav": { "down": "chapters" },
      "action": { "playTitle": { "title": 1 } } },
    { "id": "chapters", "rect": { "x": 760, "y": 920, "w": 400, "h": 80 },
      "images": { "normal": "ch_n.png", "selected": "ch_s.png" },
      "nav": { "up": "play" },
      "action": { "showMenu": "chapters" } }
  ]
}
```

| Key | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | required | Menu identifier. MUST be the same as the folder name. |
| `canvas` | object | required | Size of the coordinate system (`width`, `height`, in pixels). Section 24.3. |
| `background` | object | required | Background. Section 24.1. |
| `audio` | object | optional | Background audio (BGM). Section 24.1. |
| `defaultButton` | string | optional | `id` of the button selected initially. If absent, the first button. |
| `rememberSelection` | boolean | optional | If true, when returning to this menu, the previously selected button is selected again. Default is false. |
| `timeout` | object | optional | Action `action` taken when `afterMs` (milliseconds) elapse without any operation. |
| `buttons` | array | required | Buttons (Section 24.2). MAY be empty. |

All file names (background, audio, button images) are names relative to `MENU/<menuId>/` and MUST NOT include folders.

### 24.1 Background and Audio

- `background` MUST have **exactly one of** `image` (a still image file name) and `video` (a video file name).
  - `image`: MUST be **PNG** (so that every reader decodes it identically).
  - `video`: Matroska (the video follows the same codec rules as titles; if it has audio, it is played as BGM). If `loop` is true,
    it repeats from the beginning after playing to the end. If false, it stops on the last picture.
- `audio`: `{ "file": "bgm.mka", "loop": true }`. BGM (Matroska audio) when the background is a still image. If `loop` is true, it repeats.
  When the background is a video, the reader need not use `audio` (the audio in the video is played).

### 24.2 Buttons

| Key | Type | Required | Meaning |
|---|---|---|---|
| `id` | string | required | Identifier unique within the menu. |
| `rect` | object | required | Area (`x`, `y`, `w`, `h`; canvas coordinates). MUST fit within the canvas. The hit area for mouse and touch, and the area in which the image is drawn. |
| `images.normal` | string | optional | Image when not selected. |
| `images.selected` | string | optional | Image when selected. |
| `images.activated` | string | optional | Image shown briefly at the moment of pressing. |
| `nav.up`, `nav.down`, `nav.left`, `nav.right` | string | optional | `id` of the button selected when moving in that direction. Selection does not move in directions without one. |
| `action` | action | required | Action when pressed (Section 25). |

- Images MUST be **PNG** (with transparency). The reader scales them to the size of `rect` and overlays them on the background.
- In a state with no image, nothing is overlaid (it is also possible to draw the graphics into the background and overlay only a frame when selected).
- **Text is rendered into images by the writer.** The reader does not draw button text (so that it looks the same in every reader).
- The selected-state effect is expressed only by images (there are no specifications such as color or thickness).

### 24.3 Coordinates and Display

The reader scales the whole canvas to the video display area (the background display area) and overlays it. The display aspect ratio of the menu is the aspect ratio
of the canvas. The pixel dimensions of the background image or video MAY differ from the canvas (it is scaled to the canvas size).

## 25. Actions

An action is written as an object having **exactly one of** the following keys.

| Action | Meaning |
|---|---|
| `{ "playTitle": { "title": 1 } }` | Plays the title from the beginning. |
| `{ "playTitle": { "title": 1, "chapter": 3 } }` | Plays the title from the specified chapter (1-based). |
| `{ "showMenu": "chapters" }` | Shows the menu. |
| `{ "resume": {} }` | Returns to the position of the title that was playing before the menu was shown (also restoring the audio and subtitle selection). Does nothing if there is no position to return to. |
| `{ "setAudio": 2 }` | Selects the audio (`audio[].number` in `title.json`). The menu stays shown. Applies to titles played afterward. |
| `{ "setSubtitle": 1 }` | Selects the subtitle (`subtitle[].number` in `title.json`; 0 = no subtitles). The menu stays shown. Applies to titles played afterward. |
| `{ "stop": {} }` | Ends playback. |

- Writers MUST NOT write an action with two or more keys. Readers MUST treat an action with two or more known keys, and an action
  with only unknown keys, as **an action that does nothing** (future actions are added with new keys).
- If the number in `setAudio` or `setSubtitle` does not exist in the title being played, the reader ignores that setting.

## 26. Player Behavior

- **Start**: in playback following the controller, `firstPlay` is executed (if absent, the first title is played).
  When resuming from the previous playback position, the player MAY play from that position without executing `firstPlay`.
- **Menu selection**: the button selected when a menu is shown is the previously selected button if `rememberSelection` is true and there is one;
  otherwise `defaultButton`; and if that is also absent, the first button. When the menu is shown again because the background loops, the selection is not changed.
- **Operation**: arrow keys move the selection according to `nav`, and Enter executes `action`. With a mouse, a button is selected when the pointer moves over it,
  and executed on click. With touch, it is executed on tap.
- **Moment of pressing**: if `images.activated` exists, it is shown for a short time (roughly 0.1 to 0.2 seconds) before the action is executed.
- **Timeout**: in a menu with `timeout`, the time without operation is counted from when the menu is shown (or from the last operation),
  and when it reaches `afterMs`, `timeout.action` is executed. Counting need not continue while paused.
- **Resume**: when a menu is shown during title playback (by the player's menu key or an action within the title), the reader
  remembers that position and the audio and subtitle selection, and returns there with `resume`. When another title is played, the remembered position is discarded.
- **End of title**: follows the `onEnd` rules of Section 23.
- **Player's menu key**: when pressed during title playback, the menu in `keys` is shown (and the "resume" position is remembered).

## 27. Rules for Writers

Writers MUST satisfy the following. When a reader encounters a file that does not satisfy them, it MAY ignore the offending part
(e.g. an action pointing to a nonexistent menu does nothing).

- For each identifier in `menus` of `nav.json`, `MENU/<menuId>/menu.json` exists, and its `id` is the same as the identifier.
- The titles (`playTitle.title`, `titles[].title`) and menus (`showMenu`, `keys`) that actions refer to exist.
- Button `id`s are unique within the menu, and `nav` and `defaultButton` point to existing buttons.
- Each button's `rect` fits within the canvas, and its width and height are positive.
- `background` has exactly one of `image` and `video`. Images (background, buttons) are PNG.
- File names do not include folders, and the files they point to are in `MENU/<menuId>/`.

---

# Appendices

## Appendix A Implementation Notes (informative)

### A.1 Frames Held by the Decoder

- Decoders such as H.264 and H.265 may hold frames for reordering and not output them until the next input arrives.
  **Where a stream ends** (still parts, the end of menu video, the end of a file), the held frames must be
  taken out of the decoder (drained). Feeding further input to a drained decoder can be unstable, so recreating it is the reliable approach.
- Temporal deinterlacing filters (yadif, etc.) also do not output the last frame until the next frame arrives. Where a stream ends,
  the held frames must also be taken out of the filter, with their correct times, before it is discarded. Discarding without taking them out loses the last frame;
  continuing into the next stream without discarding mixes in old pictures.

### A.2 Part Boundaries

- When moving from a `"dual"` part to a full-height part, drain the decoder of the bottom track as well before closing the stage that combines it with the top
  (the bottom has no input in the full-height interval, so the last field remains).
- It is advisable to evaluate the end of `interlacedRanges` half a field inward (the end of an interval is the end of the display of the last frame, so
  it may exceed the time of the first frame of the next part by less than 1 ms).

### A.3 Audio Gaps and Silence Filling

- Many audio outputs pack PCM and play it regardless of timestamps. If the supply is interrupted in time, everything after it shifts earlier by that amount, and the shifts accumulate.
  Within the same segment, if the time of the next audio is later than "the end of what has been queued so far", it is advisable to **fill the difference with silence**.
- Writers SHOULD cut out a part's audio by "**the display duration of that part**". With soft telecine, the times of the original material and
  the times accumulated from fields differ, so cutting by the times of the original material can come up one field short.

### A.4 Changes in Audio Configuration

- It is advisable for the player to rebuild the audio choices when **the span to which the presentation time belongs has changed from before**. On a seek, reset the "current span" to
  undetermined, and rebuild it when the first frame is presented (rewriting it ahead of time misses the change).

## Appendix B Import Guidelines (informative)

### B.1 From DVD-Video

- **Still-image cells**: for each cell, if "IFO cell duration > actual video duration + 1 second", treat it as a still-image cell and make it a still part.
  The actual video duration is measured as **the actual number of fields including RFF × the field period** (estimating it as number of frames × 2
  misidentifies even ordinary 3:2 pulldown cells as still images).
- **Interleaved blocks**: for branches physically arranged in alternation, extract only the relevant branch using the navigation information (VOB_ID, Cell_ID).
- **Subtitles**: write the subpictures of each physical substream (per display mode) raw as `.spu`, one per display unit.
  Map the start and end of display to the cue's `startMs` and `endMs`. Changes during display are inside the subpicture, so they are not expanded.
- **Audio loudness**: map AC-3 `dialnorm` to `dialogLevelDb`, each block's `dynrng` to `drc`, and `compr` to `drcRf`.
  For DTS, map the dialogue level gain in the core header to `dialogGainDb`, and the embedded DRC coefficients to `drc`. LPCM and MPEG audio have none.
- **Angles**: store all angles as they are physically (do not merge them even if the video is the same).

### B.2 From Blu-ray

- **Subtitles (PGS)**: do not decode. Follow the sequence of segments to determine only the start of display and the erase (a composition with zero objects), and
  write the byte sequence from the Display Set that starts the display up to just before the erase as-is to `.pgs`.
- **Angles**: confirm that each angle of the same PlayItem has the same duration and configuration, and use `angleType: "bd"`.

### B.3 From Broadcast MPEG-2 TS

- **Service = title**: one title per selectable service (those referring to the same video are merged into one).
- **Program = `events[]`**: segments are not split even when the program changes; this is expressed by timed `events`.
- **Changes over time in audio and video** (dual mono ⇔ stereo, interlaced ⇔ film) are not split either, but expressed by spans and intervals.
  Segments are split only for changes that do not fit in a single encoded stream (resolution, aspect ratio, codec, audio channel count).
- **Subtitles (ARIB)**: use the text type (`charset: "arib"`). `source` is the subtitle PID (the second language is PID + 0x10000).

## Appendix C Version History (informative)

| Document version | Layout (`version`) | Content |
|---|---|---|
| 1.0 (draft) | 5 | First public version. Controllers `none`, `dvd-video-vm`, `kuroko-tiny`. |

History of layout generations: 1 = a single mkv per title (ISO container), 2 = segmentation (ISO container),
3 = ZIP container and separation of logical and physical, 4 = angles and PGS subtitles, 5 = the form with unified notation (this document).

### C.1 Differences Between Versions 3/4 and 5

The structure is the same; only the following notations differ. It is advisable for readers to distinguish them by the `version` in `KUROKO.MANIFEST`.

| Item | 3, 4 | 5 |
|---|---|---|
| `audio[].number` and `subtitle[].number` in the `title.json` of `dvd-video-vm` | The DVD logical number itself (0-based) | 1-based (DVD logical number + 1) |
| `audio` and `subtitle` of `firstPlayback` | Position in the `title.json` sequence (1-based) | `number` in `title.json` |
| Duration in `titles[]` of `disc.json` | `duration` (a `"hh:mm:ss.fffffff"` string; `dvd-video-vm` only) | `durationMs` (number) |
| `programs[].entryCell` in `title.json` and `programMap` in `menu.json` | 1-based | 0-based |
| `menuType` in `menu.json` | Number (2 to 7) | Name string |
| `palette` in `menu.json` | Sequence of numbers | Sequence of 6-digit hexadecimal strings |
| Language | 2-letter ISO 639-1 (`"en"`) for `dvd-video-vm`, 3 letters otherwise | ISO 639-2 three-letter lowercase |
| `version` in `menu.json` | `3` | `5` |

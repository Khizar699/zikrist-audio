# zikrist-audio

Open-licensed azan recordings for Zikrist, an offline Quran reader. The app downloads a
voice only when the reader turns the azan on, checks every file against `manifest.json`, and plays it at prayer time.

Every recording here comes from Wikimedia Commons or freesound under CC0, CC BY or CC BY-SA. Each file keeps its own
licence: see [LICENSE.md](LICENSE.md) and [ATTRIBUTION.md](ATTRIBUTION.md).

## Where the files are served

The app reads a fixed tag, never a branch:

```
https://cdn.jsdelivr.net/gh/Khizar699/zikrist-audio@azan-v1/<path>
https://raw.githubusercontent.com/Khizar699/zikrist-audio/azan-v1/<path>   (fallback)
```

`<path>` is a `path` from `manifest.json`, for example
`https://cdn.jsdelivr.net/gh/Khizar699/zikrist-audio@azan-v1/azan/london/full.m4a`.

A published tag is never moved. Any change to a file ships as a new tag (`azan-v2`, …) with a new manifest `version`,
because iOS caches notification sounds by name.

## Layout

```
manifest.json
azan/<voiceId>/full.m4a        the whole regular azan
azan/<voiceId>/clip.caf        its iOS notification sound
azan/<voiceId>/fajr.m4a        the whole Fajr azan, with "aṣ-ṣalātu khayrun mina n-nawm"
azan/<voiceId>/fajr-clip.caf   its iOS notification sound
```

| Voice | Variant | Length | Form | Licence |
|---|---|---|---|---|
| `london` (default) | regular | 116.1 s | 15 phrases | CC0 1.0 |
| `nabawi` | regular | 185.9 s | 15 phrases | CC BY 3.0 |
| `shalkar` | regular | 130.6 s | 15 phrases | CC BY-SA 4.0 |
| `aaqib` | regular | 84.1 s | 15 phrases | CC BY-SA 4.0 |
| `malmo` (default Fajr) | Fajr | 229.5 s | 15 phrases + Fajr phrase | CC BY 3.0 |
| `ghana` | Fajr | 83.1 s | 15 phrases + Fajr phrase | CC BY-SA 4.0 |

## The files

- `full.m4a` / `fajr.m4a`: AAC-LC, mono, 44.1 kHz, 96 kbps, `+faststart`. Leading and trailing silence trimmed to about
  0.4 s, then two-pass `loudnorm=I=-16:TP=-1.5:LRA=11`.
- `clip.caf` / `fajr-clip.caf`: IMA4, 22.05 kHz, mono, under 29.5 s (iOS plays the default sound for anything 30 s or
  longer). Cut from the start of the full file at the end of a breath gap, with a 0.6 s fade-out, so it never stops in
  the middle of a word.

## manifest.json

```ts
{
  version: 1;
  tag: "azan-v1";
  defaultVoice: string;      // a voice with a regular recording
  defaultFajrVoice: string;  // a voice with a Fajr recording
  voices: {
    id: string; name: string; place: string; muezzin: string | null;
    form: "sunni15" | "maliki" | "shafii" | "shia";
    license: string; licenseUrl: string; sourceUrl: string; attribution: string;
    regular: Variant | null;
    fajr: Variant | null;
  }[];
}
type Variant = {
  full: { path: string; bytes: number; md5: string; seconds: number };
  clip: { path: string; bytes: number; md5: string; seconds: number };
  shortEndSec: number;        // end of the opening takbir, in whole phrases
  phrases: number[] | null;   // start second of each call: 12, or 14 at Fajr
};
```

- `md5` is the lowercase hex MD5 of the file; `seconds` is rounded to 0.1.
- `shortEndSec` is the end of the fourth "Allāhu akbar" when that comes within 29.5 s, otherwise the end of the second.
- `phrases` lists the start of each call in the full file's timeline, in this order: Allāhu akbar ×2, Allāhu akbar ×2,
  the first shahada twice, the second shahada twice, ḥayya ʿala-ṣ-ṣalāh twice, ḥayya ʿala-l-falāḥ twice, (Fajr only)
  aṣ-ṣalātu khayrun mina n-nawm twice, Allāhu akbar ×2, lā ilāha illā-llāh.

## How each recording was checked

A file name is never trusted. Before a recording is listed:

1. Its licence is read from the Commons API or the freesound page, and the uploader's history is checked. Re-uploads of
   broadcasts, "own work" claims from accounts with copyright problems, and wrong public-domain tags are rejected.
2. Its breath groups are found from the RMS envelope and cross-checked with ffmpeg `silencedetect`.
3. Each breath group is transcribed with whisper.cpp (`ggml-medium`, language Arabic, DTW word timestamps), and unclear
   groups again with `large-v3-turbo` and in context.
4. It is listed only if it is one muezzin, in a known form, complete from the first "Allāhu akbar" to the final
   "lā ilāha illā-llāh", and free of clipping. A Fajr recording must contain both "aṣ-ṣalātu khayrun mina n-nawm" calls
   after the second "ḥayya ʿala-l-falāḥ".

The bundled takbir in the Zikrist app (`zikrist-azan-takbir.caf`, `zikrist_azan_takbir.ogg`) is the first 17.5 s of
`azan/london/full.m4a`, the CC0 voice, with the same 0.6 s fade.

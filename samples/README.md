# サンプル / Samples

機能ごとの小さな Kuroko ファイル。素材（試験の映像・音・字幕・メニューの画像）はすべて ffmpeg で合成したもので、
[kuroko-authoring](https://github.com/tmasuda-aaiwnet/kuroko-authoring) の `samples/make-samples.cs` で作りました
（映像 AV1・音声 Opus）。

Small Kuroko files, one per feature. All source material (test video, tones, subtitles, menu images) was synthesized
with ffmpeg, and the files were made with `samples/make-samples.cs` in
[kuroko-authoring](https://github.com/tmasuda-aaiwnet/kuroko-authoring) (AV1 video, Opus audio).

| ファイル / File | コントローラ / Controller | 中身 / Contents |
|---|---|---|
| `none-basic.krk` | `none` | 1 タイトル・640x360・6 秒・音声 1 本・章 2 つ / 1 title, 640x360, 6 s, 1 audio track, 2 chapters |
| `none-multi.krk` | `none` | 2 タイトル。タイトル 1 は音声 2 本（jpn・eng）と字幕 2 本（テキスト型）/ 2 titles; title 1 has 2 audio tracks (jpn, eng) and 2 text subtitle tracks |
| `none-interlaced.krk` | `none` | 720x480・29.97i（上のフィールドが先）をフィールド分離（`fields`・`dual`）で格納 / 720x480 29.97i (top field first) stored as separated fields (`fields`, `dual`) |
| `tiny-menu.krk` | `kuroko-tiny` | `none-multi` の 2 タイトルに、画像の背景とボタン 2 つのメニュー / the 2 titles of `none-multi` with a menu (image background, 2 buttons) |

## 権利 / Rights

これらのサンプルは [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/deed.ja)（権利を放棄）で提供します。
自由に使ってかまいません。

These samples are dedicated to the public domain under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).

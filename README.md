# Kuroko フォーマット仕様

[English](#english)

Kuroko（黒子・拡張子 `.krk`）は、映像・音声・字幕を Matroska（mkv）に収め、その再生の進め方を
**コントローラ**が与えるマルチメディアのコンテナ形式です。メニュー・字幕付きの多タイトルを表せ、
インターレースや 24p のソフトテレシネといった元の表示のケイデンスを保ったまま、H.264・H.265・AV1 で小さく格納できます。

- 仕様書：[日本語（正本）](spec/kuroko.ja.md)・[English](spec/kuroko.en.md)
- 状態：**1.0 草案**（内容は変わることがあります。変更は [CHANGELOG.md](CHANGELOG.md)）
- サンプル：[samples/](samples/)
- 実装：[kuroko-authoring](https://github.com/tmasuda-aaiwnet/kuroko-authoring)（読み書きの SDK・オーサリングツール。MIT）、
  [kuroko-converter](https://github.com/tmasuda-aaiwnet/kuroko-converter)（DVD・Blu-ray・放送 TS からの変換器。GPL-3.0-or-later）

## 特長

- **殻は ZIP、中身は Matroska、進め方は JSON**。OS の標準機能や汎用の道具で中身を見て、確かめて、作れます。
- **ケイデンスを保つ**：テレシネ・インターレースの情報をメタデータとタイムスタンプで持ち、エンコーダに依存せず元の表示で再生できます。
  **途中でケイデンスが変わる映像**——24p のフィルム（3:2 プルダウン）の本編の途中にインターレースの映像が挟まる、
  プルダウンの周期が途中で崩れる、など MPEG-2 のフラグで表せていたもの——も、**AV1・H.264・H.265 で再現**できます。
  フレームごとの表示時間はタイムスタンプに、インターレースの区間は時刻の区間で持つので、コーデックの機能に頼りません。
- **コントローラ**：ナビゲーションの無い素の再生（`none`）、DVD-Video 互換の仮想マシンでメニューを表す `dvd-video-vm`、
  メニュー・ボタン・動作を JSON で宣言的に書く `kuroko-tiny`（高解像度の画像・動画の背景のメニュー）。
- **重複なく格納**：複数のタイトルが共有する本編を 1 回だけ持ち、継ぎ目なく再生できます。
- **音声はフルレンジ**：ラウドネスの制御はメタデータとして持ち、再生時に任意で当てます。

## 今後の予定

- 検証ツール（krk の中身が仕様どおりかを確かめる道具）

## 権利

- 仕様書は [CC BY 4.0](LICENSE) で提供します。出典を示せば、商用を含めて自由に使えます。
- サンプル（`samples/`）は [CC0 1.0](samples/README.md) です。
- **Kuroko 形式の読み書きは、誰でも無償で実装できます。** 許諾や届け出は要りません。
- **著者は、Kuroko 形式の実装・利用に対して、自らの特許を主張しません。**

## ご意見・不具合の報告

Issue で受け付けます。仕様書の変更は著者が行うため、プルリクエストは受け付けていません（[CONTRIBUTING.md](CONTRIBUTING.md)）。

© 2026 Takao Masuda（増田システムデザイニング）

---

## English

Kuroko (Japanese for the black-clad stagehand of traditional theater; file extension `.krk`) is a multimedia container
format. Video, audio and subtitles are stored in Matroska (mkv), and a **controller** describes how they are played.
It can represent multi-title discs with menus and subtitles, and stores H.264, H.265 or AV1 video compactly while
preserving the original display cadence of interlaced video and 24p soft telecine.

- Specification: [Japanese (authoritative)](spec/kuroko.ja.md) / [English](spec/kuroko.en.md)
- Status: **1.0 draft** (subject to change; see [CHANGELOG.md](CHANGELOG.md))
- Samples: [samples/](samples/)
- Implementations: [kuroko-authoring](https://github.com/tmasuda-aaiwnet/kuroko-authoring) (read/write SDK and authoring
  tool, MIT), [kuroko-converter](https://github.com/tmasuda-aaiwnet/kuroko-converter) (converter from DVD, Blu-ray and
  broadcast TS, GPL-3.0-or-later)

### Highlights

- **A ZIP shell, Matroska inside, JSON for navigation** — inspect, verify and create files with standard OS features and
  general-purpose tools.
- **Cadence preservation** — telecine and interlace information is kept as metadata and timestamps, so the original
  presentation is reproduced independently of the encoder, including video whose cadence changes midway.
- **Controllers** — plain playback without navigation (`none`), DVD-Video compatible virtual machine menus
  (`dvd-video-vm`), and declarative JSON menus with high-resolution image or video backgrounds (`kuroko-tiny`).
- **No duplication** — content shared by several titles is stored once and played seamlessly.
- **Full-range audio** — loudness control is metadata, applied optionally at playback.

### Planned

- A validator that checks whether a krk file conforms to the specification.

### Rights

- The specification is provided under [CC BY 4.0](LICENSE). The samples are [CC0 1.0](samples/README.md).
- **Anyone may implement reading and writing of the Kuroko format free of charge**, without permission or notification.
- **The author does not assert any of their patents against implementations or uses of the Kuroko format.**

### Feedback

Please use Issues. Pull requests are not accepted, as the specification is edited by the author
(see [CONTRIBUTING.md](CONTRIBUTING.md)).

© 2026 Takao Masuda (Masuda System Designing)

# iphone-duo-development

**English** | [日本語](#日本語)

A Claude skill for adapting iOS apps to **iPhone Duo** — Apple's foldable iPhone, announced 2026-09-09 and shipping 2026-10-23 with iOS 27.1.

The premise of this skill is that iPhone Duo is not a new platform. It is an iPhone whose window size changes a great deal. Most of the work is making layouts resizable, which is the same work as supporting iPad. The Duo-specific parts — the fold, vertical bars, asymmetric safe areas — sit on top of that.

## What it covers

- **Resizable layouts** — size classes per display and orientation, sizing against the container, and what to remove (`UIScreen.main`, idiom checks, orientation checks, fixed widths)
- **Asymmetric safe areas** — a failure mode that resizing the window does not surface, and the one portrait-locked apps are most exposed to
- **Vertical bars** — when toolbars and tab bars move to the side, why title-only items do not appear there, ordering, overflow priority, and the exceptions for sheets, split views and inspectors
- **The fold** — reserved regions (`division` and `occlusion`), arrangement views, and the principle that matters most: keep interactive elements off the fold
- **Build and verification environment** — the Xcode 27.1 requirement, why older toolchains refuse to create a Duo simulator, and how to verify before 27.1 ships
- **Release timing** — the review sequence, and the App Store featuring nomination deadline counted back from the ship date

## Layout

```
SKILL.md                          Core guidance, adoption checklist, common failures
references/bars.md                Vertical bars and toolbar APIs in detail
references/reserved-regions.md    The fold, reserved regions, arrangement views, multiple windows
references/environment.md         Toolchain, simulators, CI, release timing
```

For ordinary use `SKILL.md` alone is enough. The reference files are loaded when the work moves into their area.

## Installation

Copy it into Claude's skills directory.

```bash
cp -R iphone-duo-skill ~/.claude/skills/iphone-duo-development
```

To use it in a single project:

```bash
cp -R iphone-duo-skill <project>/.claude/skills/iphone-duo-development
```

It triggers on mentions of iPhone Duo, foldable devices, the inner or outer display, the fold, the hinge, `ArrangementView`, `ReservedRegion`, and vertical bars. It also triggers on work like making an existing iPhone app resizable or adopting size classes, even when Duo is not named.

## Sources

Built from Apple's primary material.

- Tech Talks 111461–111466 (cited with timecodes). Each session page has its transcript and official code samples embedded in the HTML; the commands for extracting them are in `references/environment.md`
- Human Interface Guidelines — *Designing for iPhone Duo*
- *Preparing your app for iPhone Duo* (Technology Overviews)
- SwiftUI / UIKit reference for the relevant iOS 27 APIs

The environment behavior written up in `references/environment.md` — the simulator's `Incompatible device` error, the device type and runtime profiles behind it, and how to tell which runtimes are available — was verified against an actually installed toolchain rather than copied out of documentation.

Transcript text is Apple's copyrighted work and is not included in this repository.

## License

MIT

---

# 日本語

[English](#iphone-duo-development) | **日本語**

iOS アプリを **iPhone Duo**（Apple の折りたたみ iPhone。2026-09-09 発表、2026-10-23 に iOS 27.1 とともに発売）へ対応させるための Claude スキルです。

このスキルの前提は、iPhone Duo は新しいプラットフォームではない、という点にあります。ウインドウのサイズが大きく変わる iPhone です。作業のほとんどはレイアウトをリサイズ可能にすることで、これは iPad 対応と同じ内容です。折り目・垂直バー・非対称なセーフエリアといった Duo 固有の部分は、その上に乗ります。

## 扱う範囲

- **リサイズ対応レイアウト** — ディスプレイと向きごとの size class、コンテナ基準のサイズ指定、取り除くもの（`UIScreen.main`、idiom 判定、向き判定、固定幅）
- **非対称なセーフエリア** — ウインドウのリサイズでは表に出ない失敗の型であり、縦向き固定のアプリが最もさらされている箇所
- **垂直バー** — ツールバーとタブバーが側面へ移る条件、タイトルだけの項目がそこに出ない理由、配置順、オーバーフローの優先度、シート・分割ビュー・インスペクタの例外
- **折り目** — 予約領域（`division` と `occlusion`）、arrangement view、そして最も重要な原則である「操作要素を折り目に置かない」
- **ビルドと検証環境** — Xcode 27.1 要件、古いツールチェーンで Duo シミュレータの作成が拒否される理由、27.1 が出る前の検証方法
- **リリースの時期** — 審査の段取りと、発売日から逆算した App Store フィーチャー申請の期限

## 構成

```
SKILL.md                          中核の指針、対応チェックリスト、ありがちな失敗
references/bars.md                垂直バーとツールバー API の詳細
references/reserved-regions.md    折り目、予約領域、arrangement view、複数ウインドウ
references/environment.md         ツールチェーン、シミュレータ、CI、リリース時期
```

通常の用途では `SKILL.md` だけで完結します。参照ファイルは、作業がその領域に入ったときに読み込まれます。

## 導入

Claude のスキルディレクトリへコピーします。

```bash
cp -R iphone-duo-skill ~/.claude/skills/iphone-duo-development
```

プロジェクト単位で使う場合は次のようにします。

```bash
cp -R iphone-duo-skill <project>/.claude/skills/iphone-duo-development
```

iPhone Duo、折りたたみ端末、内側・外側ディスプレイ、折り目、ヒンジ、`ArrangementView`、`ReservedRegion`、垂直バーへの言及で起動します。加えて、Duo と明示されていなくても、既存の iPhone アプリをリサイズ対応にする、size class を導入するといった作業でも起動します。

## 出典

Apple の一次資料から構成しています。

- Tech Talks 111461〜111466（タイムコード付きで引用）。各セッションのページには、トランスクリプトと公式コードサンプルが HTML に埋め込まれています。抽出用のコマンドは `references/environment.md` に記載しています
- Human Interface Guidelines — *Designing for iPhone Duo*
- *Preparing your app for iPhone Duo*（Technology Overviews）
- 関係する iOS 27 の API についての SwiftUI / UIKit リファレンス

`references/environment.md` に書かれた環境の挙動——シミュレータの `Incompatible device` エラー、その原因であるデバイスタイプとランタイムのプロファイル、入手可能なランタイムの判別——は、ドキュメントからの引き写しではなく、実際に導入済みのツールチェーンに対して検証した結果です。

トランスクリプト本文は Apple の著作物であるため、このリポジトリには含めていません。

## ライセンス

MIT

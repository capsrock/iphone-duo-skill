# iphone-duo-development

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

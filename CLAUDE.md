# Claude Data Analysis Marketplace

このリポジトリはデータ分析プラグインのマーケットプレイスです。

## リポジトリ構造

```
plugins/
└── data-analysis/          # データ分析プラグイン v2.0.0
    ├── .claude-plugin/     # プラグイン定義
    ├── agents/             # エージェント定義
    ├── commands/           # スラッシュコマンド（ウィザード）
    ├── skills/             # スキル（data-* 10個）
    ├── templates/          # ユーザーがコピーして使うテンプレート
    └── 使用ガイド.md
```

## プラグイン開発ルール

- スキルファイルは `skills/<skill-name>/SKILL.md` に配置する
- スキルのfrontmatterには `name` と `description` を必ず記載する
- 新スキルは既存の GL-1〜GL-10 ガイドラインに準拠させる
- ユーザー向けテンプレートは `templates/` に配置する

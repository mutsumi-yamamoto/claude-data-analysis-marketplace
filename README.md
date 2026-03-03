# Claude Data Analysis Plugin

**CRISP-DM拡張フレームワーク × 戦略コンサルタント行動規範** に基づくデータ分析プラグインです。
`analysis_context.md` をSSOT（唯一の真実の情報源）として管理し、Phase 0〜8 の体系的なデータ分析を Claude Code 上で実現します。

---

## インストール方法

Claude Code 上で以下の2行を実行してください。リポジトリのクローンは不要です。

```
/plugin marketplace add あなたのユーザー名/claude-data-analysis-marketplace
/plugin install data-analysis@data-analysis-marketplace
```

### スコープを指定してインストールする場合

| 用途 | コマンド |
|------|---------|
| 自分の全プロジェクトで使う（デフォルト） | `/plugin install data-analysis@data-analysis-marketplace --scope user` |
| チーム全員で共有する | `/plugin install data-analysis@data-analysis-marketplace --scope project` |
| このプロジェクトだけで使う | `/plugin install data-analysis@data-analysis-marketplace --scope local` |

---

## セットアップ（インストール後）

インストール後、分析を行うプロジェクトに以下の2ファイルをコピーしてください。

```bash
# CLAUDE.md（Claude Code が自動読み込みする設定ファイル）
cp ~/.claude/plugins/cache/data-analysis/templates/CLAUDE.md ./CLAUDE.md

# analysis_context.md（分析の文脈を管理するSSOTファイル）
cp ~/.claude/plugins/cache/data-analysis/templates/analysis_context.md ./analysis_context.md
```

コピー後のプロジェクト構造:

```
your-project/
├── CLAUDE.md              ← 会話開始時に自動読み込み
├── analysis_context.md    ← 分析コンテキスト（SSOT）
└── data/
    ├── your_data.csv
    └── docs/              ← 分析レポートの出力先
```

---

## 使い方

### 1. 分析を開始する

```
/data-analysis:da-start
```

### 2. 標準フロー（推奨）

```
/data-analysis:data-context    ← 必ず最初に実行（ヒアリング）
/data-analysis:data-define     ← Issue Tree・KPI・仮説の設計
/data-analysis:data-collect    ← データ要件定義（必要時）
/data-analysis:data-integrate  ← 複数ソース統合（必要時）
/data-analysis:data-explore    ← Phase 1: EDA
/data-analysis:data-clean      ← Phase 2: データクレンジング
/data-analysis:data-feature    ← Phase 3-5: 特徴量エンジニアリング
/data-analysis:data-model      ← Phase 6: モデリング・分析
/data-analysis:data-interpret  ← Phase 7-8: 結果解釈・レポート
```

### 3. フルサイクル一括実行

```
/data-analysis:data-analyze
```

---

## 主な機能

| 機能 | 内容 |
|------|------|
| **9フェーズ分析** | CRISP-DM拡張版 Phase 0〜8 を体系的に実行 |
| **SSOT管理** | `analysis_context.md` で分析の文脈を一元管理・引き継ぎ |
| **コンサルタント思考** | Issue Driven・Hypothesis Driven・So What?・MECE |
| **バイアス防止** | GL-1〜GL-10 の恣意的分析禁止ガイドラインを全スキルに適用 |
| **レポート自動生成** | エグゼクティブサマリー・技術詳細レポートを `data/docs/` に出力 |

## スキル一覧

| コマンド | 役割 |
|---------|------|
| `/data-analysis:data-context` | コンテキストヒアリング・更新 |
| `/data-analysis:data-define` | 問題定義・KPI・Issue Tree |
| `/data-analysis:data-collect` | データ要件定義・収集 |
| `/data-analysis:data-integrate` | 複数ソース統合・整合性検証 |
| `/data-analysis:data-analyze` | Phase 0〜8 フルサイクル |
| `/data-analysis:data-explore` | データ理解・EDA |
| `/data-analysis:data-clean` | データ品質評価・クレンジング |
| `/data-analysis:data-feature` | 特徴量エンジニアリング |
| `/data-analysis:data-model` | 分析・モデリング |
| `/data-analysis:data-interpret` | 結果解釈・レポート作成 |

---

## 対応データ形式

- CSV（UTF-8 / Shift-JIS）
- Excel（.xlsx）
- ZIP（自動展開後にCSVを処理）
- 複数ファイルの結合（`data-integrate` スキルで対応）

## 技術スタック

Python 3 / pandas / numpy / scipy / scikit-learn / matplotlib / seaborn / statsmodels / xgboost / lightgbm / shap

---

## ライセンス

MIT License

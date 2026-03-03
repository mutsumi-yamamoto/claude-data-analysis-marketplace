---
description: データ分析プロジェクトの開始ウィザードです。CRISP-DM拡張フレームワークに基づき、プロジェクトの目的・データ・進め方を対話的に確認し、最適なスキルを案内します。
---

# データ分析プラグイン v2 — 開始ウィザード

## ステップ1: データのアップロード確認

**最初に、分析対象のデータファイルをアップロードしてもらう。**

`data/` ディレクトリを確認する:

- **データがある場合**: ファイル名・形式（CSV/Excel/JSON等）・件数・列名を把握し、ステップ2へ進む。
- **データがない場合**: AskUserQuestion でデータの準備状況を確認する。
  - 「分析に使うデータファイルを `data/` フォルダへアップロードしてください。CSV / Excel / JSON に対応しています。」
  - データが手元にない場合は `/data-analysis:data-collect` でデータ要件定義から始める。

## ステップ2: analysis_context.md の確認

次に `analysis_context.md` が存在するか確認する。

- **ファイルがない場合**: `templates/analysis_context.md` をプロジェクトルートにコピーし、`/data-analysis:data-context` でヒアリングを開始する。
- **ファイルがある場合**: 内容を読み込んでからステップ3へ進む。

## ステップ3: データ内容を参照したプロジェクト状況の確認

ステップ1で把握したデータ内容（列名・型・件数・サンプル値）を提示しながら、AskUserQuestion で以下を確認する:

1. **分析テーマ**: データの列構成から想定される分析テーマを提示し、目的を確認する
2. **目的の種別**: 現状把握 / 予測 / 因果推論 / 異常検知 / クラスタリング
3. **意思決定への活用**: 誰が・どんな判断に使うか
4. **制約**: 期限・使える技術スタック

確認した内容と、データから読み取れる情報（データ期間・粒度・欠損状況など）を合わせて `analysis_context.md` に反映する。

## ステップ4: 推奨フローを案内する

### 新規プロジェクト（推奨順）

> **前提**: `data/` にデータファイルがアップロード済みであること。

```
① /data-analysis:data-context   — コンテキストヒアリング・更新（最初に実行）
② /data-analysis:data-define    — ビジネス課題・KPI・仮説の設計
③ /data-analysis:data-collect   — データ要件定義・収集・カタログ化
④ /data-analysis:data-integrate — 複数ソース統合・整合性検証

⑤ /data-analysis:data-explore   — データ理解・概要把握（Phase 1）
⑥ /data-analysis:data-clean     — データ品質評価・クレンジング（Phase 2）
⑦ /data-analysis:data-feature   — 特徴量設計・エンジニアリング（Phase 3-5）
⑧ /data-analysis:data-model     — 分析・モデリング実行（Phase 6）
⑨ /data-analysis:data-interpret — 結果解釈・レポート作成（Phase 7-8）
```

### フルサイクル一括実行

```
/data-analysis:data-analyze — Phase 0〜8 を順に実行
```

---

このプラグインは恣意的分析禁止ガイドライン（GL-1〜GL-10）および
CRISP-DM拡張フレームワーク（Phase 0〜8）に基づいて動作します。

$ARGUMENTS

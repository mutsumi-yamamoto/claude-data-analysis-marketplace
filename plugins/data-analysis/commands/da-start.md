---
description: データ分析プロジェクトの開始ウィザードです。CRISP-DM拡張フレームワークに基づき、プロジェクトの目的・データ・進め方を対話的に確認し、最適なスキルを案内します。
---

# データ分析プラグイン v2 — 開始ウィザード

## ステップ1: analysis_context.md の確認

まず `analysis_context.md` が存在するか確認してください。

- **ファイルがない場合**: `templates/analysis_context.md` をプロジェクトルートにコピーし、`/data-analysis:data-context` でヒアリングを開始してください。
- **ファイルがある場合**: 内容を読み込んでから以下のステップへ進んでください。

## ステップ2: プロジェクト状況の確認

以下を順番に確認します（AskUserQuestion を使用）:

1. **分析テーマ**: 何を分析したいですか？
2. **データの状況**: データはすでにある / これから収集する
3. **目的の種別**: 現状把握 / 予測 / 因果推論 / 異常検知 / クラスタリング
4. **制約**: 期限・使える技術スタック・データ量

## ステップ3: 推奨フローを案内する

### 新規プロジェクト（推奨順）

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

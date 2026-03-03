
## ── commands/da-start.md ─────────────────────────────────

---
description: データ分析プロジェクトの開始ウィザードです。プロジェクトの目的・データ・進め方を対話的に確認し、最適なスキルを案内します。
---

# データ分析プラグイン — 開始ウィザード

以下を順番に確認してください。

## ステップ1: プロジェクト確認

1. **分析テーマ**: 何を分析したいですか？
2. **データの状況**: データはすでにある / これから収集する
3. **目的**: 現状把握 / 予測 / 因果推論 / 異常検知
4. **制約**: 期限・使える技術スタック

## ステップ2: 推奨フェーズを案内する

① /data-analysis:da-requirements  — 要件定義・KPI設計
② /data-analysis:da-eda           — データ前処理・EDA
③ /data-analysis:da-features      — 特徴量エンジニアリング
④ /data-analysis:da-modeling      — モデル選定・交差検証
⑤ /data-analysis:da-visualize     — 可視化・レポート
⑥ /data-analysis:da-evaluate      — 評価・改善
⑦ /data-analysis:da-ops           — 管理・運用

このプラグインは恣意的分析禁止ガイドライン（GL-1〜GL-10）に基づいて動作します。

$ARGUMENTS
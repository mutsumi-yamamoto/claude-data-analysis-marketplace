# データ分析プロジェクト

## 最重要ルール

**どのスキル・フェーズを実行する場合も、必ず最初に `analysis_context.md` を読み込むこと。**

- ファイルが存在しない → `analysis_context.md` の作成をユーザーに促す
- 内容が空・不十分 → `AskUserQuestion` でヒアリングしてから分析に進む
- 内容が十分 → そのまま指定されたスキルを実行する

分析で得た知見・方針変更は `analysis_context.md` の「分析経過メモ」に随時追記する。

---

## ディレクトリ構造

```
analysis_context.md    # 分析コンテキスト（SSOT）← 必ず最初に読む
data/                  # CSVデータファイル・分析スクリプト
data/docs/             # 分析レポート・可視化の出力先
```

---

## 技術スタック

- **言語**: Python 3（`python3` で実行）
- **主要ライブラリ**: pandas, numpy, scipy, scikit-learn, matplotlib, seaborn, statsmodels, xgboost, lightgbm, shap
- **エンコーディング**: UTF-8（CSVは `encoding='utf-8'` または `encoding='cp932'` を明示）
- **日本語フォント**: `plt.rcParams['font.family'] = 'Hiragino Sans'`
- **乱数シード**: `random_state=42` で固定

---

## 利用可能なスキル（スラッシュコマンド）

### 準備・定義フェーズ
| コマンド | 役割 |
|---------|------|
| `/data-analysis:data-context` | コンテキストヒアリング・更新（**最初に実行**） |
| `/data-analysis:data-define` | ビジネス課題・KPI・Issue Tree の設計 |
| `/data-analysis:data-collect` | データ要件定義・収集・カタログ化 |
| `/data-analysis:data-integrate` | 複数ソース統合・整合性検証 |

### 分析フェーズ
| コマンド | 役割 |
|---------|------|
| `/data-analysis:data-analyze` | Phase 0〜8 フルサイクル一括実行 |
| `/data-analysis:data-explore` | Phase 1: データ理解・EDA |
| `/data-analysis:data-clean` | Phase 2: データ品質評価・クレンジング |
| `/data-analysis:data-feature` | Phase 3-5: 特徴量エンジニアリング |
| `/data-analysis:data-model` | Phase 6: 分析・モデリング |
| `/data-analysis:data-interpret` | Phase 7-8: 結果解釈・レポート作成 |

### 推奨実行順（新規プロジェクト）
```
/data-context → /data-define → /data-collect → /data-integrate
→ /data-explore → /data-clean → /data-feature → /data-model → /data-interpret
```

---

## 恣意的分析禁止ガイドライン（GL-1〜GL-10）

| # | ルール |
|---|--------|
| GL-1 | p値 < 0.05 のみ有意差。境界値は「限界的有意」と明記 |
| GL-2 | 効果量を必ず報告。Cohen's d < 0.2 は「実務的に無意味」と明記 |
| GL-3 | 多重検定時は Bonferroni/FDR 補正を適用 |
| GL-4 | 点推定値には必ず 95% 信頼区間を付ける |
| GL-5 | 分析前に検定力分析で必要サンプル数を明示（1-β = 0.80） |
| GL-6 | 外れ値除外は IQR×1.5 / 3σ 等の客観的基準のみ |
| GL-7 | 仮説を支持・否定する証拠を同等に提示 |
| GL-8 | グラフ軸は 0 始まりを原則。変更時は明記 |
| GL-9 | 相関関係を因果関係として解釈しない |
| GL-10 | 分析計画はデータ確認前に確定 |

---

## レポーティング規約

- 分析レポートは `data/docs/` に番号付きで保存（例: `07_executive_summary.md`）
- レポートには必ず含める: 分析目的・使用データ・手法と根拠・結果・限界・Next Step
- Next Step は「誰が / 何を / いつまでに」の 3 要素を含める

## 出力フォーマット（ピラミッドストラクチャー）

```
【Issue】解くべき問い
【結論】端的な回答（1-2文）
【根拠】
 1. Key Message A — Supporting Fact
 2. Key Message B — Supporting Fact
【リスク・前提】結論が覆る条件
【Next Step】誰が / 何を / いつまでに
```

## バイアスチェックリスト（全フェーズ共通）

- [ ] サバイバーシップバイアス: 脱落データの有無を確認
- [ ] 選択バイアス: サンプルが母集団を代表しているか
- [ ] 確証バイアス: 仮説と矛盾する証拠を積極的に探す
- [ ] シンプソンのパラドックス: サブグループ分析を必ず実施
- [ ] 相関 ≠ 因果: 時間的前後関係・メカニズム・交絡因子の統制
- [ ] ターゲットリーケージ: 時間的整合性の厳密な確認
- [ ] 多重比較問題: Bonferroni 補正・FDR 制御
- [ ] オーバーフィッティング: 交差検証・正則化

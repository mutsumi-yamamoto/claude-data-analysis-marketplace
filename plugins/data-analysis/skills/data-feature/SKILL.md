---
name: data-feature
description: 特徴量設計・エンジニアリングを行います（Phase 3-5）。「特徴量を作りたい」「変数を選びたい」「特徴量エンジニアリングして」と言われたら使用してください。
---

# Phase 3-5: 特徴量設計・エンジニアリングスキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## Phase 3: 特徴量設計・構想

### 1. ドメイン知識に基づく仮説リスト

`analysis_context.md` の仮説リストを参照し、各仮説に対応する特徴量候補を洗い出す:

| 仮説 | 特徴量候補 | データソース | 作成方法 |
|------|-----------|-------------|---------|
| ○○が高い顧客は解約しやすい | 最終購入からの経過日数 | 購買履歴 | 日付差分 |

### 2. 分析粒度（Grain）の確定

1レコード = 何かを再確認する（Phase 4 での結合計画と整合させる）

### 3. ターゲットリーケージの防止【GL-10】

**最重要チェック**: 予測時に使えない情報が混入していないか

- 時系列データ: 予測時点より「未来」の情報を使っていないか
- ターゲット変数と直接関連する変数（例: 解約後のコールセンター利用回数）を除外
- 実際の運用フローを確認: 「予測時点でこの特徴量は取得できるか？」

```python
# リーケージ確認: 目的変数との異常に高い相関を確認
target = 'target_col'
correlations = df.drop(columns=[target]).corrwith(df[target]).abs()
high_corr = correlations[correlations > 0.9]
if len(high_corr) > 0:
    print("⚠️ 目的変数との相関が異常に高い（リーケージ疑い）:")
    print(high_corr)
```

---

## Phase 4: 分析テーブル構築

```python
import pandas as pd
import numpy as np

# 分析単位を確認
print(f"分析単位: 1レコード = ○○")

# 必要な特徴量の結合（Fan-out防止）
n_before = len(df_base)
df_analysis = df_base.merge(df_features, on='key', how='left')
n_after = len(df_analysis)

print(f"結合前: {n_before:,} / 結合後: {n_after:,}")
if n_after > n_before:
    print("⚠️ Fan-out検知: 結合キーの一意性を確認してください")
```

---

## Phase 5: 特徴量エンジニアリング

### 1. 数値変換

```python
# 対数変換（右歪み分布に有効）
skewed_cols = df.select_dtypes(include='number').apply(
    lambda x: abs(x.skew()) > 1
).index.tolist()

for col in skewed_cols:
    if df[col].min() > 0:
        df[f'log_{col}'] = np.log1p(df[col])
        print(f"対数変換: {col} → log_{col}")

# ビニング（連続値のカテゴリ化）
df['age_bin'] = pd.cut(df['age'], bins=[0, 25, 40, 60, 100],
                        labels=['若年', '中年', 'シニア', '高齢'])
```

### 2. カテゴリエンコーディング

```python
# One-Hot Encoding（カーディナリティ < 20 推奨）
low_card_cols = [col for col in df.select_dtypes(include='object').columns
                 if df[col].nunique() < 20]
df = pd.get_dummies(df, columns=low_card_cols, drop_first=True)

# Target Encoding（高カーディナリティ用）
from category_encoders import TargetEncoder
# ※ 必ずTrain/Testを分割してからfitする（リーケージ防止）
```

### 3. 時系列特徴量（該当する場合）

```python
df['date'] = pd.to_datetime(df['date'])
df = df.sort_values('date')

# ラグ特徴量
for lag in [1, 7, 30]:
    df[f'lag_{lag}'] = df['value'].shift(lag)

# 移動平均
for window in [7, 30]:
    df[f'rolling_mean_{window}'] = df['value'].rolling(window=window).mean()

# 差分（トレンド除去）
df['diff_1'] = df['value'].diff(1)

# 季節性（曜日・月）
df['day_of_week'] = df['date'].dt.dayofweek
df['month'] = df['date'].dt.month
```

### 4. 交互作用特徴量

```python
# ドメイン知識に基づく交互作用
df['price_per_unit'] = df['total_price'] / df['quantity'].replace(0, np.nan)
df['recency_x_frequency'] = df['recency'] * df['frequency']
```

### 5. スケーリング（Train/Testを必ず分割してから実施）【GL-10】

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler
from sklearn.model_selection import train_test_split

# Train/Test分割（スケーリング前に必ず実施）
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y  # 分類の場合はstratify
)

# スケーラーはTrainのみでfit（TestにはTransformのみ）
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)   # Trainでfit + transform
X_test_scaled = scaler.transform(X_test)          # TestはTransformのみ
```

スケーラー選択の基準:
- 正規分布に近い → `StandardScaler`
- 値域を [0, 1] に揃えたい → `MinMaxScaler`
- 外れ値がある → `RobustScaler`（中央値・IQRベース）

### 6. 多重共線性チェック【GL-9】

```python
from statsmodels.stats.outliers_influence import variance_inflation_factor

def calc_vif(df):
    vif = pd.DataFrame()
    vif['Feature'] = df.columns
    vif['VIF'] = [variance_inflation_factor(df.values, i)
                  for i in range(df.shape[1])]
    return vif.sort_values('VIF', ascending=False)

vif_df = calc_vif(X_train)
print(vif_df)
print("\nVIF > 10 の特徴量（多重共線性の疑い）:")
print(vif_df[vif_df['VIF'] > 10])
```

### 7. 特徴量重要度の評価（暫定）

```python
from sklearn.ensemble import RandomForestClassifier  # or Regressor
import shap

# 暫定的な重要度確認（本格的な評価はモデリング後）
rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X_train_scaled, y_train)

# SHAP値（解釈可能な重要度）
explainer = shap.TreeExplainer(rf)
shap_values = explainer.shap_values(X_test_scaled)
shap.summary_plot(shap_values, X_test, feature_names=X_train.columns)
```

**注意**: Gini重要度はバイアスがあるため、SHAP値またはPermutation Importanceを優先する。

## 8. 発見事項のサマリーと Next Step

```
【特徴量エンジニアリングサマリー】
 - 元の特徴量数: ○○個
 - 作成した特徴量: ○○個（内訳: 数値変換○個、カテゴリ○個、時系列○個）
 - リーケージリスクで除外: ○○個（理由: ○○）
 - 多重共線性で除外: ○○個（VIF > 10）
 - 最終特徴量数: ○○個
【Next Step】
 → /data-analysis:data-model で分析・モデリングに進む
```

---

## 📝 実行ログの記録（必須）

`data/docs/05_feature_engineering.md` に保存し、`analysis_context.md` の「12. 実行ログ」末尾に以下のテンプレートを埋めて追記すること。

```
### YYYY-MM-DD HH:MM | data-feature
| 項目 | 内容 |
|------|------|
| ステータス | 完了 / 一部完了 / 中断 |
| 実施内容 | [数値変換 / カテゴリエンコード / 時系列特徴量 / 交互作用項 / 次元削減] |
| 元の特徴量数 | [件数] |
| 作成した特徴量 | [件数]（数値変換: [件]、カテゴリ: [件]、時系列: [件]） |
| リーケージ除外 | [件数]（除外理由: [内容]） |
| VIF除外 | [件数]（VIF > 10 の列名） |
| 最終特徴量数 | [件数] |
| 重要特徴量候補 | [モデリング前の予測特徴量トップ3] |
| 問題・懸念 | [ターゲットリーケージ疑い・VIF問題等] |
| 申し送り | [モデリングで使用する特徴量セット・注意点] |
| 生成ファイル | `data/docs/05_feature_engineering.md` |
```

---

## ✅ 完了後: 次の推奨アクション（必須）

上記の実行が完了したら、**必ず**以下をユーザーに提示すること。

**✅ data-feature が完了しました**
📁 生成ファイル: `data/docs/05_feature_engineering.md`
📋 `analysis_context.md` の「分析経過メモ」を更新しました

**▶ 次の推奨ステップ（標準フロー）:**
```
/data-analysis:data-model
```
設計した特徴量を用いてモデリング・統計分析を実施します

**現在の推奨フロー:**
```
data-context → data-define → data-explore → data-clean → data-feature ✅ → data-model → data-interpret
```

$ARGUMENTS

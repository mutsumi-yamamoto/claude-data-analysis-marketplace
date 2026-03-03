---
name: data-clean
description: データ品質評価・クレンジングを行います（Phase 2: データ品質評価・クレンジング）。「データをきれいにしたい」「前処理したい」「データクレンジングして」と言われたら使用してください。
---

# Phase 2: データ品質評価・クレンジングスキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 1. データ品質6次元の評価

| 次元 | 定義 | 評価方法 | 評価結果 |
|------|------|---------|---------|
| 正確性 | 実世界の事実を正確に表しているか | ドメイン知識との照合 | |
| 完全性 | 必要なデータが欠損なく揃っているか | 欠損率の確認 | |
| 一貫性 | 複数箇所で矛盾がないか | クロスチェック | |
| 一意性 | 同じ情報が重複して記録されていないか | 重複検出 | |
| 妥当性 | 値が許容範囲・フォーマット内か | 値域・型チェック | |
| 適時性 | データの鮮度が分析目的に合っているか | タイムスタンプ確認 | |

## 2. クレンジング順序（必ずこの順で実施）

### Step 1: 重複削除

```python
import pandas as pd

print(f"クレンジング前: {len(df):,} 件")

# 完全重複の削除
n_before = len(df)
df = df.drop_duplicates()
print(f"重複削除後: {len(df):,} 件（{n_before - len(df):,} 件削除）")

# ビジネスキーに基づく重複確認
dup_check = df.duplicated(subset=['primary_key'], keep=False)
print(f"主キー重複: {dup_check.sum():,} 件")
```

### Step 2: 欠損値処理

```python
missing_rate = df.isnull().mean()

# 欠損率に応じた処理方針
for col in df.columns:
    rate = missing_rate[col]
    if rate == 0:
        continue
    elif rate < 0.05:
        print(f"{col}: {rate:.1%} → 平均/中央値補完を検討")
    elif rate < 0.20:
        print(f"{col}: {rate:.1%} → 多重代入法(MICE)を推奨")
    else:
        print(f"{col}: {rate:.1%} → 変数の除外またはモデルベース補完を検討")
```

**MCAR（完全ランダム欠損）の場合**:
```python
# リストワイズ削除
df_clean = df.dropna(subset=['target_col'])
```

**MAR（ランダム欠損）の場合**:
```python
# 多重代入法（scikit-learn IterativeImputer = MICE相当）
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer

imputer = IterativeImputer(random_state=42, max_iter=10)
df_imputed = pd.DataFrame(
    imputer.fit_transform(df.select_dtypes(include='number')),
    columns=df.select_dtypes(include='number').columns
)
```

**MNAR（非ランダム欠損）の場合**:
- 欠損フラグ変数を追加（欠損自体が情報）
- ドメイン専門家と協議して処理方針を決定

### Step 3: 外れ値処理【GL-6】

**客観的基準のみ使用（恣意的除外禁止）**:

```python
# IQR法
def detect_outliers_iqr(series):
    q1, q3 = series.quantile([0.25, 0.75])
    iqr = q3 - q1
    lower = q1 - 1.5 * iqr
    upper = q3 + 1.5 * iqr
    return (series < lower) | (series > upper)

# 3σ法
def detect_outliers_zscore(series):
    z_scores = (series - series.mean()) / series.std()
    return z_scores.abs() > 3

for col in df.select_dtypes(include='number').columns:
    outliers_iqr = detect_outliers_iqr(df[col])
    outliers_z = detect_outliers_zscore(df[col])
    print(f"{col}: IQR法={outliers_iqr.sum()}件, 3σ法={outliers_z.sum()}件")
```

**重要**: 外れ値を機械的に除外することを禁止する。ドメイン知識で「真の異常値か、データ入力エラーか、稀少な正常値か」を判断してから処理方針を決定する。

## 3. クレンジング前後の比較【GL-7】

```python
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams['font.family'] = 'Hiragino Sans'

for col in df_clean.select_dtypes(include='number').columns:
    fig, axes = plt.subplots(1, 2, figsize=(12, 4))

    df[col].hist(bins=30, ax=axes[0], alpha=0.7, edgecolor='black')
    axes[0].set_title(f'{col} — クレンジング前')

    df_clean[col].hist(bins=30, ax=axes[1], alpha=0.7, edgecolor='black',
                       color='orange')
    axes[1].set_title(f'{col} — クレンジング後')

    plt.tight_layout()
    plt.savefig(f'data/docs/02_clean_{col}.png', dpi=150)
    plt.close()

# 統計量の変化を確認
print("クレンジング前後の統計量比較:")
print(pd.concat([
    df.describe().add_prefix('前_'),
    df_clean.describe().add_prefix('後_')
], axis=1))
```

## 4. 値域チェック（妥当性検証）

```python
# 定義上ありえない値の検出
validity_rules = {
    '年齢': (0, 120),
    '価格': (0, None),  # 負の値は不正
    '売上数量': (0, None),
}

for col, (min_val, max_val) in validity_rules.items():
    if col not in df.columns:
        continue
    if min_val is not None:
        invalid = df[col] < min_val
        print(f"{col} < {min_val}: {invalid.sum()} 件")
    if max_val is not None:
        invalid = df[col] > max_val
        print(f"{col} > {max_val}: {invalid.sum()} 件")
```

## 5. クレンジング結果サマリー

```python
print("=== クレンジングサマリー ===")
print(f"クレンジング前: {n_original:,} 件")
print(f"クレンジング後: {len(df_clean):,} 件")
print(f"削除率: {(1 - len(df_clean)/n_original):.1%}")

# 保存
df_clean.to_csv('data/cleaned_data.csv', index=False, encoding='utf-8')
```

## 6. 発見事項のサマリーと Next Step

```
【クレンジングサマリー】
 - 重複削除: ○○件
 - 欠損処理: ○○カラム（手法: ○○）
 - 外れ値処置: ○○件（判断根拠: ○○）
 - 最終レコード数: ○○件（元データの○○%）
【注意事項・限界】
 - ○○については「不確実性: ○○」の判断が残っている
【Next Step】
 → /data-analysis:data-feature で特徴量エンジニアリングに進む
```

---

クレンジング結果を `analysis_context.md` の「分析経過メモ」に追記し、`data/docs/02_cleaning_report.md` にレポートを保存すること。

$ARGUMENTS

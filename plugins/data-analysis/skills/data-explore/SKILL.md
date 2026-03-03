---
name: data-explore
description: データ理解・概要把握を行います（Phase 1: データ理解）。「データを調べたい」「EDAしたい」「データの概要を把握したい」と言われたら使用してください。
---

# Phase 1: データ理解・概要把握スキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 1. データソースの把握

- データの出所・収集方法・収集頻度を確認する
- 収集・記録時のバイアスリスクを明示する
- **落とし穴**: サバイバーシップバイアス（記録されていないデータがないか）

## 2. スキーマ確認

```python
import pandas as pd
import numpy as np

df = pd.read_csv('data/○○.csv', encoding='utf-8')

print("=== スキーマ確認 ===")
print(f"レコード数: {len(df):,}")
print(f"カラム数: {len(df.columns)}")
print("\nデータ型:")
print(df.dtypes)
print("\nサンプル:")
print(df.head())
```

主キー・外部キーを特定し、各カラムのビジネス的意味をコメントで記録する。

## 3. 基本統計量【GL-2, GL-4】

```python
print("=== 基本統計量 ===")
print(df.describe(include='all'))

# 追加統計量
for col in df.select_dtypes(include=[np.number]).columns:
    q1, q3 = df[col].quantile([0.25, 0.75])
    iqr = q3 - q1
    skewness = df[col].skew()
    kurtosis = df[col].kurt()
    print(f"{col}: 歪度={skewness:.2f}, 尖度={kurtosis:.2f}, IQR={iqr:.2f}")
```

## 4. 分布の可視化【GL-8】

```python
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams['font.family'] = 'Hiragino Sans'

# ヒストグラム + KDE
fig, axes = plt.subplots(2, 3, figsize=(15, 8))
for i, col in enumerate(df.select_dtypes(include=[np.number]).columns[:6]):
    ax = axes[i//3][i%3]
    df[col].hist(bins=30, ax=ax, edgecolor='black', alpha=0.7)
    ax.set_title(col)
    ax.set_xlabel('値')  # 軸は0始まりを原則【GL-8】
plt.tight_layout()
plt.savefig('data/docs/01_distributions.png', dpi=150)
plt.show()
```

## 5. 欠損値パターンの分析

```python
print("=== 欠損値分析 ===")
missing = pd.DataFrame({
    '欠損数': df.isnull().sum(),
    '欠損率': df.isnull().mean().map('{:.1%}'.format)
}).sort_values('欠損数', ascending=False)
print(missing[missing['欠損数'] > 0])
```

欠損パターンの分類:
- **MCAR（完全ランダム欠損）**: 欠損が他変数と無関係
- **MAR（ランダム欠損）**: 欠損が他変数で説明可能
- **MNAR（非ランダム欠損）**: 欠損値自体に情報がある（最も危険）

```python
# MCAR/MARの確認（他変数との相関）
import missingno as msno
msno.matrix(df)
plt.savefig('data/docs/01_missing_pattern.png', dpi=150)
```

## 6. カーディナリティ確認

```python
print("=== カーディナリティ ===")
for col in df.select_dtypes(include=['object', 'category']).columns:
    n_unique = df[col].nunique()
    print(f"{col}: {n_unique:,} ユニーク値")
    if n_unique <= 20:
        print(df[col].value_counts().head(10))
    print()
```

## 7. 時系列の確認（該当する場合）

```python
# 時系列カラムの特定と確認
date_col = '日付カラム名'
df[date_col] = pd.to_datetime(df[date_col])
print(f"期間: {df[date_col].min()} 〜 {df[date_col].max()}")
print(f"データ鮮度: {(pd.Timestamp.now() - df[date_col].max()).days} 日前")

# 時系列の連続性確認
date_range = pd.date_range(df[date_col].min(), df[date_col].max())
missing_dates = date_range.difference(df[date_col])
print(f"欠落日数: {len(missing_dates)}")
```

## 8. 変数間の関係【GL-9】

```python
# 相関分析（相関 ≠ 因果）
corr_matrix = df.select_dtypes(include=[np.number]).corr()

plt.figure(figsize=(12, 10))
sns.heatmap(corr_matrix, annot=True, fmt='.2f', cmap='coolwarm',
            center=0, vmin=-1, vmax=1)
plt.title('相関行列（相関は因果を示さない）【GL-9】')  # 注釈を付ける
plt.savefig('data/docs/01_correlation.png', dpi=150)
```

**必ず明記**: 「相関が観察された」止まりにする。因果の主張は別途因果推論分析が必要。

## 9. 主要な発見事項のサマリー

探索的分析の結果を以下の形式で整理する:

```
【Issue】Phase 0 で定義した問い（再掲）
【発見事項】
 1. ○○の分布は右歪み（中央値: ○○、平均: ○○）
 2. ○○と○○の間に強い正相関（r=○○）— ただし因果は未検証
 3. ○○の欠損率が○○%（MARと推定: ○○との相関あり）
【注意事項・限界】
 - ○○のデータは○○期間のみ（直近○ヶ月のデータなし）
 - サバイバーシップバイアスの可能性: ○○
【Next Step】
 → /data-analysis:data-clean でデータクレンジングに進む
```

---

分析結果を `analysis_context.md` の「分析経過メモ」に追記し、`data/docs/01_eda_report.md` にレポートを保存すること。

$ARGUMENTS

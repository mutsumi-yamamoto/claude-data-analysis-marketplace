---
name: data-explore
description: シニアデータサイエンティストレベルのEDA（探索的データ分析）を行います（Phase 1）。「データを調べたい」「EDAしたい」「データの概要を把握したい」「時系列を分析したい」と言われたら使用してください。
---

# Phase 1: データ理解・探索的分析スキル（Senior DS レベル）

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 0. 自動プロファイリング（最初に実行）

```python
import pandas as pd
import numpy as np
import warnings
warnings.filterwarnings('ignore')

# ─────────────────────────────────────────
# 0-1. ydata-profiling による自動レポート生成
# ─────────────────────────────────────────
try:
    from ydata_profiling import ProfileReport
    profile = ProfileReport(df, title="EDA Auto Profile", explorative=True,
                            correlations={"pearson": {"calculate": True},
                                          "spearman": {"calculate": True},
                                          "kendall": {"calculate": True}})
    profile.to_file("data/docs/01_auto_profile.html")
    print("自動プロファイリングレポート生成完了: data/docs/01_auto_profile.html")
except ImportError:
    print("ydata-profiling 未インストール。pip install ydata-profiling で追加可能。")
    print("手動EDAに進みます。")
```

---

## 1. データソース・生成過程の理解

- データの出所・収集方法・収集頻度を確認する
- **サバイバーシップバイアス**: 記録されていないデータ（品切れ・クレーム未記録等）がないか
- **選択バイアス**: サンプルが母集団を代表しているか
- **測定誤差**: 計測方法・単位・定義の変更がないか

```python
# ─────────────────────────────────────────
# 1-1. スキーマ確認
# ─────────────────────────────────────────
print("=== スキーマ確認 ===")
print(f"レコード数: {len(df):,} | カラム数: {len(df.columns)}")
print("\nデータ型:")
print(df.dtypes.to_string())
print("\nサンプル:")
print(df.head(3).to_string())
print("\n末尾:")
print(df.tail(3).to_string())

# 主キー候補の特定
for col in df.columns:
    if df[col].nunique() == len(df):
        print(f"主キー候補: {col} (全ユニーク)")
```

---

## 2. 基本統計量（拡張版）【GL-2, GL-4】

```python
# ─────────────────────────────────────────
# 2-1. 拡張統計量
# ─────────────────────────────────────────
from scipy import stats

print("=== 基本統計量（拡張） ===")
stats_df = df.describe(percentiles=[.01, .05, .25, .5, .75, .95, .99]).T
stats_df['skewness'] = df.select_dtypes(include='number').skew()
stats_df['kurtosis'] = df.select_dtypes(include='number').kurt()
stats_df['cv'] = stats_df['std'] / stats_df['mean'].abs()  # 変動係数
print(stats_df.to_string())

# ─────────────────────────────────────────
# 2-2. 正規性検定【GL-1】
# ─────────────────────────────────────────
print("\n=== 正規性検定（Shapiro-Wilk / D'Agostino-Pearson） ===")
for col in df.select_dtypes(include='number').columns:
    series = df[col].dropna()
    if len(series) < 5000:  # Shapiro-Wilk: n < 5000 推奨
        stat, p = stats.shapiro(series.sample(min(len(series), 5000), random_state=42))
        test_name = "Shapiro-Wilk"
    else:                   # 大サンプルはD'Agostino-Pearson
        stat, p = stats.normaltest(series)
        test_name = "D'Agostino-Pearson"

    if p > 0.05:
        result = "正規分布を棄却できず"
    elif 0.01 < p <= 0.05:
        result = "正規性疑わしい（p < 0.05）"
    else:
        result = "非正規分布（p < 0.01）"
    print(f"  {col}: {test_name} p={p:.4f} → {result}")
```

---

## 3. 分布の可視化（Senior DS 版）【GL-8】

```python
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats as scipy_stats

plt.rcParams['font.family'] = 'Hiragino Sans'

# ─────────────────────────────────────────
# 3-1. ヒストグラム + KDE + QQプロット
# ─────────────────────────────────────────
num_cols = df.select_dtypes(include='number').columns.tolist()
n_cols = min(len(num_cols), 6)

fig, axes = plt.subplots(n_cols, 2, figsize=(14, 4 * n_cols))
for i, col in enumerate(num_cols[:n_cols]):
    series = df[col].dropna()

    # ヒストグラム + KDE
    axes[i][0].hist(series, bins='auto', density=True, alpha=0.6, edgecolor='k')
    kde_x = np.linspace(series.min(), series.max(), 200)
    kde = scipy_stats.gaussian_kde(series)
    axes[i][0].plot(kde_x, kde(kde_x), 'r-', lw=2, label='KDE')
    axes[i][0].set_title(f'{col}\n歪度={series.skew():.2f}, 尖度={series.kurt():.2f}')
    axes[i][0].set_xlabel('')
    axes[i][0].legend()

    # QQプロット
    scipy_stats.probplot(series, dist="norm", plot=axes[i][1])
    axes[i][1].set_title(f'{col} — QQプロット（正規性確認）')

plt.tight_layout()
plt.savefig('data/docs/01_distributions.png', dpi=150)
plt.close()
print("分布可視化を保存: data/docs/01_distributions.png")

# ─────────────────────────────────────────
# 3-2. 外れ値可視化（箱ひげ図 + Violin）
# ─────────────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(14, max(4, n_cols * 1.2)))
df[num_cols[:n_cols]].plot.box(ax=axes[0], vert=False)
axes[0].set_title('箱ひげ図（外れ値候補: ●）')
axes[0].set_xlabel('')

if n_cols <= 4:
    import seaborn as sns
    df_melt = df[num_cols[:n_cols]].melt(var_name='変数', value_name='値')
    sns.violinplot(data=df_melt, x='値', y='変数', ax=axes[1], inner='box', cut=0)
    axes[1].set_title('Violin プロット（分布形状）')
else:
    axes[1].axis('off')

plt.tight_layout()
plt.savefig('data/docs/01_outlier_boxplot.png', dpi=150)
plt.close()
```

---

## 4. 欠損値の高度分析

```python
import missingno as msno

# ─────────────────────────────────────────
# 4-1. 欠損パターンの分類（MCAR / MAR / MNAR）
# ─────────────────────────────────────────
print("=== 欠損値分析 ===")
missing = pd.DataFrame({
    '欠損数': df.isnull().sum(),
    '欠損率(%)': (df.isnull().mean() * 100).round(2)
}).sort_values('欠損率(%)', ascending=False)
print(missing[missing['欠損数'] > 0].to_string())

# MCAR/MARチェック: 欠損フラグと他変数の相関
for col in df.columns[df.isnull().any()]:
    missing_flag = df[col].isnull().astype(int)
    for other_col in df.select_dtypes(include='number').columns:
        if other_col != col:
            corr = missing_flag.corr(df[other_col])
            if abs(corr) > 0.2:
                print(f"  MAR可能性: {col}の欠損 ↔ {other_col} (r={corr:.3f})")

# ─────────────────────────────────────────
# 4-2. 欠損パターン可視化
# ─────────────────────────────────────────
try:
    fig = msno.matrix(df, figsize=(14, 6), sparkline=False)
    plt.title('欠損値パターン行列（白=欠損）')
    plt.savefig('data/docs/01_missing_pattern.png', dpi=150, bbox_inches='tight')
    plt.close()

    msno.heatmap(df, figsize=(10, 8))
    plt.title('欠損値の共起ヒートマップ')
    plt.savefig('data/docs/01_missing_heatmap.png', dpi=150, bbox_inches='tight')
    plt.close()
except ImportError:
    print("missingno 未インストール。pip install missingno で追加可能。")
```

---

## 5. 変数間の関係分析【GL-9】

```python
# ─────────────────────────────────────────
# 5-1. ピアソン + スピアマン相関
# ─────────────────────────────────────────
num_df = df.select_dtypes(include='number')
pearson = num_df.corr(method='pearson')
spearman = num_df.corr(method='spearman')

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(18, 7))
sns.heatmap(pearson, annot=True, fmt='.2f', cmap='coolwarm',
            center=0, vmin=-1, vmax=1, ax=ax1)
ax1.set_title('ピアソン相関（線形・正規分布前提）\n※相関は因果を示さない【GL-9】')

sns.heatmap(spearman, annot=True, fmt='.2f', cmap='coolwarm',
            center=0, vmin=-1, vmax=1, ax=ax2)
ax2.set_title('スピアマン相関（順位・非線形対応）\n※相関は因果を示さない【GL-9】')

plt.tight_layout()
plt.savefig('data/docs/01_correlation.png', dpi=150)
plt.close()

# ─────────────────────────────────────────
# 5-2. 相互情報量（非線形依存関係の検出）
# ─────────────────────────────────────────
from sklearn.feature_selection import mutual_info_regression, mutual_info_classif

# 目的変数が指定されている場合
if 'target' in df.columns:
    target = df['target']
    features = num_df.drop(columns=['target'], errors='ignore')

    if target.nunique() <= 20:  # 分類
        mi = mutual_info_classif(features.fillna(features.median()), target, random_state=42)
    else:  # 回帰
        mi = mutual_info_regression(features.fillna(features.median()), target, random_state=42)

    mi_df = pd.Series(mi, index=features.columns).sort_values(ascending=False)
    print("\n=== 相互情報量（目的変数との非線形依存度） ===")
    print(mi_df.to_string())
    print("※ 0に近い = ほぼ独立、値が大きい = 強い依存関係")
    print("注意: 相互情報量は相関係数では検出できない非線形関係も捉える")

# ─────────────────────────────────────────
# 5-3. カテゴリ変数間の関係: Cramér's V
# ─────────────────────────────────────────
from scipy.stats import chi2_contingency

def cramers_v(x, y):
    confusion_matrix = pd.crosstab(x, y)
    chi2 = chi2_contingency(confusion_matrix)[0]
    n = confusion_matrix.sum().sum()
    phi2 = chi2 / n
    r, k = confusion_matrix.shape
    phi2corr = max(0, phi2 - ((k-1)*(r-1))/(n-1))
    rcorr = r - ((r-1)**2)/(n-1)
    kcorr = k - ((k-1)**2)/(n-1)
    return np.sqrt(phi2corr / min((kcorr-1), (rcorr-1)))

cat_cols = df.select_dtypes(include=['object', 'category']).columns.tolist()
if len(cat_cols) >= 2:
    cramers_matrix = pd.DataFrame(index=cat_cols, columns=cat_cols, dtype=float)
    for c1 in cat_cols:
        for c2 in cat_cols:
            mask = df[c1].notna() & df[c2].notna()
            cramers_matrix.loc[c1, c2] = cramers_v(df.loc[mask, c1], df.loc[mask, c2])
    print("\n=== Cramér's V（カテゴリ変数間の関連強度） ===")
    print(cramers_matrix.to_string())
    print("0: 独立, 1: 完全関連（0.1未満: 弱, 0.3以上: 強）")
```

---

## 6. カーディナリティ・カテゴリ分析

```python
print("=== カーディナリティ分析 ===")
for col in df.select_dtypes(include=['object', 'category']).columns:
    n_unique = df[col].nunique()
    missing_rate = df[col].isna().mean()
    print(f"\n{col}: {n_unique:,} ユニーク値 | 欠損率={missing_rate:.1%}")

    if n_unique <= 30:
        vc = df[col].value_counts(normalize=True)
        for val, pct in vc.head(10).items():
            print(f"  {val}: {pct:.1%}")

        # 稀少カテゴリの検出（0.5%未満）
        rare = vc[vc < 0.005]
        if len(rare) > 0:
            print(f"  ⚠️ 稀少カテゴリ {len(rare)}個（< 0.5%）: {rare.index.tolist()[:5]}...")
    else:
        print(f"  高カーディナリティ: Target Encoding / Embedding 推奨")
```

---

## 7. 時系列EDA（時系列データの場合）

```python
# ─────────────────────────────────────────
# 7-1. 時系列の基本確認
# ─────────────────────────────────────────
date_col = None
for col in df.columns:
    try:
        sample = pd.to_datetime(df[col].dropna().head(10))
        if (sample.max() - sample.min()).days > 0:
            date_col = col
            break
    except Exception:
        continue

if date_col:
    df[date_col] = pd.to_datetime(df[date_col])
    df_ts = df.sort_values(date_col)

    print(f"時系列カラム: {date_col}")
    print(f"期間: {df_ts[date_col].min()} 〜 {df_ts[date_col].max()}")
    print(f"データ鮮度: {(pd.Timestamp.now() - df_ts[date_col].max()).days}日前")

    # 欠落日の確認
    freq = pd.infer_freq(df_ts[date_col].drop_duplicates().sort_values())
    print(f"推定頻度: {freq}")

    # ─────────────────────────────────────────
    # 7-2. STL 分解（トレンド / 季節性 / 残差）
    # ─────────────────────────────────────────
    from statsmodels.tsa.seasonal import STL
    from statsmodels.tsa.stattools import adfuller, kpss

    # 数値列ごとに STL 分解
    value_cols = df.select_dtypes(include='number').columns.tolist()
    for vcol in value_cols[:3]:  # 最大3列
        ts_series = df_ts.set_index(date_col)[vcol].dropna()

        if len(ts_series) >= 14:
            try:
                # 期間を自動推定
                if freq and 'D' in str(freq):
                    period = 7   # 日次 → 週次季節性
                elif freq and 'M' in str(freq):
                    period = 12  # 月次 → 年次季節性
                else:
                    period = min(12, len(ts_series) // 2)

                stl = STL(ts_series, period=period, robust=True)
                result = stl.fit()

                fig, axes = plt.subplots(4, 1, figsize=(14, 10))
                axes[0].plot(ts_series.index, ts_series.values, 'b-')
                axes[0].set_title(f'{vcol} — 観測値')
                axes[1].plot(ts_series.index, result.trend, 'r-')
                axes[1].set_title('トレンド成分')
                axes[2].plot(ts_series.index, result.seasonal, 'g-')
                axes[2].set_title(f'季節性成分 (period={period})')
                axes[3].plot(ts_series.index, result.resid, 'k-', alpha=0.7)
                axes[3].axhline(0, color='red', linestyle='--')
                axes[3].set_title('残差（異常値候補を確認）')
                plt.suptitle(f'STL 分解: {vcol}', fontsize=13)
                plt.tight_layout()
                plt.savefig(f'data/docs/01_stl_{vcol}.png', dpi=150)
                plt.close()

                # 季節性強度・トレンド強度
                var_resid = result.resid.var()
                trend_strength = max(0, 1 - var_resid / (result.trend + result.resid).var())
                seasonal_strength = max(0, 1 - var_resid / (result.seasonal + result.resid).var())
                print(f"\n{vcol} STL結果:")
                print(f"  トレンド強度: {trend_strength:.3f} (1に近いほど強いトレンド)")
                print(f"  季節性強度: {seasonal_strength:.3f} (1に近いほど強い季節性)")

            except Exception as e:
                print(f"STL分解エラー ({vcol}): {e}")

    # ─────────────────────────────────────────
    # 7-3. 定常性検定 (ADF + KPSS)
    # ─────────────────────────────────────────
    print("\n=== 定常性検定 ===")
    for vcol in value_cols[:3]:
        ts_series = df_ts.set_index(date_col)[vcol].dropna()
        if len(ts_series) < 10:
            continue

        # ADF: H0=単位根あり(非定常), H1=定常
        adf_stat, adf_p, _, _, adf_crit, _ = adfuller(ts_series, autolag='AIC')

        # KPSS: H0=定常, H1=単位根(非定常)
        try:
            kpss_stat, kpss_p, _, kpss_crit = kpss(ts_series, regression='c', nlags='auto')
            kpss_result = "定常" if kpss_p > 0.05 else "非定常(差分化検討)"
        except Exception:
            kpss_result = "計算不可"

        adf_result = "定常" if adf_p < 0.05 else "非定常(差分化検討)"
        print(f"\n{vcol}:")
        print(f"  ADF: p={adf_p:.4f} → {adf_result}")
        print(f"  KPSS: → {kpss_result}")
        print(f"  判定: ADF定常 & KPSS定常 → 問題なし")
        print(f"        いずれかが非定常 → 差分変換・対数変換を検討")

    # ─────────────────────────────────────────
    # 7-4. ACF/PACF プロット
    # ─────────────────────────────────────────
    from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

    for vcol in value_cols[:2]:
        ts_series = df_ts.set_index(date_col)[vcol].dropna()
        if len(ts_series) < 20:
            continue

        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 4))
        lags = min(40, len(ts_series) // 3)
        plot_acf(ts_series, lags=lags, ax=ax1, alpha=0.05)
        ax1.set_title(f'ACF: {vcol}\nラグ相関（青帯 = 95%CI）')
        plot_pacf(ts_series, lags=lags, ax=ax2, alpha=0.05, method='ywmle')
        ax2.set_title(f'PACF: {vcol}\n偏自己相関（ARモデルの次数確認）')
        plt.tight_layout()
        plt.savefig(f'data/docs/01_acf_{vcol}.png', dpi=150)
        plt.close()
        print(f"ACF/PACF保存: data/docs/01_acf_{vcol}.png")
```

---

## 8. 高度な外れ値検出（統計 + 機械学習）【GL-6】

```python
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.preprocessing import StandardScaler

# ─────────────────────────────────────────
# 8-1. Isolation Forest（多変量）
# ─────────────────────────────────────────
num_df_clean = df.select_dtypes(include='number').dropna()
scaler = StandardScaler()
X_scaled = scaler.fit_transform(num_df_clean)

iso_forest = IsolationForest(contamination=0.05, random_state=42, n_jobs=-1)
outlier_labels_if = iso_forest.fit_predict(X_scaled)
n_outliers_if = (outlier_labels_if == -1).sum()
print(f"Isolation Forest: {n_outliers_if}件の外れ値候補 ({n_outliers_if/len(num_df_clean):.1%})")

# ─────────────────────────────────────────
# 8-2. LOF（局所外れ値検出、密度ベース）
# ─────────────────────────────────────────
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.05, n_jobs=-1)
outlier_labels_lof = lof.fit_predict(X_scaled)
n_outliers_lof = (outlier_labels_lof == -1).sum()
print(f"LOF:              {n_outliers_lof}件の外れ値候補 ({n_outliers_lof/len(num_df_clean):.1%})")

# ─────────────────────────────────────────
# 8-3. 両手法で外れ値と判定されたレコードのみ報告
# ─────────────────────────────────────────
consensus_outliers = (outlier_labels_if == -1) & (outlier_labels_lof == -1)
print(f"\n両手法で外れ値: {consensus_outliers.sum()}件 (IF∩LOF)")
print("→ これらを優先的にドメイン知識で確認すること【GL-6】")
print("→ 'データ入力エラー'か '真の異常値'か '稀少な正常値'かを判断してから処理")

# ─────────────────────────────────────────
# 8-4. 単変量 IQR 法
# ─────────────────────────────────────────
print("\n=== 単変量外れ値（IQR法）===")
for col in num_df_clean.columns:
    q1, q3 = df[col].quantile([0.25, 0.75])
    iqr = q3 - q1
    lower, upper = q1 - 1.5 * iqr, q3 + 1.5 * iqr
    n_out = ((df[col] < lower) | (df[col] > upper)).sum()
    if n_out > 0:
        print(f"  {col}: {n_out}件 ({lower:.2f}未満 or {upper:.2f}超)")
```

---

## 9. 目的変数分析（ターゲット分析）

```python
target_col = None  # analysis_context.md から取得するか、ユーザーに確認

if target_col and target_col in df.columns:
    target = df[target_col]

    # ─────────────────────────────────────────
    # 9-1. 二値分類の場合: クラスバランス
    # ─────────────────────────────────────────
    if target.nunique() <= 20:
        print("=== ターゲット分布（分類）===")
        vc = target.value_counts(normalize=True)
        print(vc.to_string())
        if vc.min() < 0.1:
            print(f"⚠️ クラス不均衡 (少数クラス {vc.min():.1%}) → SMOTE/重み付けを検討")

        # 数値特徴量ごとの目的変数別分布
        fig, axes = plt.subplots(1, min(4, len(num_df_clean.columns)), figsize=(16, 5))
        for i, col in enumerate(num_df_clean.columns[:4]):
            for cls in target.unique():
                mask = target == cls
                axes[i].hist(df.loc[mask, col].dropna(), bins=30, alpha=0.5, label=str(cls))
            axes[i].set_title(col)
            axes[i].legend()
        plt.suptitle('特徴量別ターゲット分布')
        plt.tight_layout()
        plt.savefig('data/docs/01_target_distribution.png', dpi=150)
        plt.close()

    # ─────────────────────────────────────────
    # 9-2. 回帰の場合: ターゲットの分布
    # ─────────────────────────────────────────
    else:
        print("=== ターゲット分布（回帰） ===")
        print(target.describe().to_string())
        print(f"歪度: {target.skew():.3f} | 尖度: {target.kurt():.3f}")
        if abs(target.skew()) > 1:
            print("⚠️ 右/左歪み → 対数変換 or Box-Cox変換を検討")
```

---

## 10. EDA サマリーと Next Step

```
【Issue】（analysis_context.md より再掲）

【EDA 発見事項トップ5】
1. データ品質: ○○ → [クレンジング方針]
2. 分布特性: ○○ → [前処理方針]
3. 外れ値: ○○件 → [処理方針]
4. 相関: ○○ → [特徴量選定への示唆]（因果未検証【GL-9】）
5. 時系列: ○○ → [モデリング方針]

【バイアスチェック】
- [ ] サバイバーシップバイアス: [有無・内容]
- [ ] 選択バイアス: [有無・内容]
- [ ] データ生成過程の変化点: [有無・内容]

【不確実性・要確認事項】
- 不確実性: [内容]（確度: 高/中/低）

【Next Step】
→ /data-analysis:data-clean でデータクレンジングに進む
```

---

## 📝 実行ログの記録（必須）

`data/docs/01_eda_report.md` にレポートを保存し、`analysis_context.md` の「12. 実行ログ」末尾に以下のテンプレートを埋めて追記すること。

```
### YYYY-MM-DD HH:MM | data-explore
| 項目 | 内容 |
|------|------|
| ステータス | 完了 / 一部完了 / 中断 |
| 実施内容 | [基本統計 / 分布確認 / 欠損分析 / 相関分析 / 外れ値検出 / 時系列分析] |
| データ規模 | [行数] 件 × [列数] 列 |
| 欠損状況 | [最大欠損率・欠損が多い列名] |
| 外れ値 | [検出件数・対象列・Isolation Forest/LOFスコア] |
| 分布の特徴 | [歪み・正規性検定結果・変換が必要な列] |
| 主要相関 | [相関係数上位ペア・多重共線性の懸念] |
| バイアスチェック | [サバイバーシップ / 選択バイアスの有無] |
| クレンジング優先対応 | [要対応の列名と処理方針] |
| 問題・懸念 | [解決できなかった問題・ドメイン専門家確認が必要な事項] |
| 申し送り | [クレンジングフェーズで特に注意すべき点] |
| 生成ファイル | `data/docs/01_eda_report.md` |
```

---

## ✅ 完了後: 次の推奨アクション（必須）

上記の実行が完了したら、**必ず**以下をユーザーに提示すること。

**✅ data-explore が完了しました**
📁 生成ファイル: `data/docs/01_eda_report.md`
📋 `analysis_context.md` の「分析経過メモ」を更新しました

**▶ 次の推奨ステップ（標準フロー）:**
```
/data-analysis:data-clean
```
EDAで発見した欠損・外れ値・型不整合・重複を体系的に処理します

**現在の推奨フロー:**
```
data-context → data-define → data-explore ✅ → data-clean → data-feature → data-model → data-interpret
```

$ARGUMENTS

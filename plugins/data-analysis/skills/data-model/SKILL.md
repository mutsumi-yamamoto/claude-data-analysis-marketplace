---
name: data-model
description: シニアデータサイエンティストレベルの分析・モデリングを実行します（Phase 6）。「モデルを作りたい」「機械学習を使いたい」「統計分析して」「時系列予測して」「需要予測して」「クラスタリングして」「因果推論して」と言われたら使用してください。
---

# Phase 6: 分析・モデリングスキル（Senior DS レベル）

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 0. 分析手法の選択フレームワーク

`analysis_context.md` の目的を確認し、適切な手法を選択する:

```
分析の4段階:
1. 記述的分析: 何が起きたか（現状把握・KPI分析）
2. 診断的分析: なぜ起きたか（原因特定・仮説検証）
3. 予測的分析: 何が起きるか（需要予測・リスク予測）
4. 処方的分析: 何をすべきか（最適化・シミュレーション）

目的変数がある？
├── Yes → 教師あり学習
│   ├── 連続値（回帰） → Ridge/Lasso/XGBoost/LightGBM
│   ├── 二値分類 → LogReg/RF/XGBoost（+クラス不均衡対応）
│   ├── 多クラス → XGBoost/LightGBM
│   └── 時系列 → ARIMA/SARIMA/Prophet/指数平滑化
└── No → 教師なし学習
    ├── グルーピング → KMeans/DBSCAN/階層クラスタリング
    ├── 次元削減 → PCA/UMAP/t-SNE
    └── 異常検知 → Isolation Forest/LOF/Autoencoder

因果推論が必要？
├── 実験可能 → A/Bテスト（RCT）
└── 観察データ → DID/RDD/傾向スコアマッチング/CausalImpact
```

---

## 1. 記述的分析（何が起きたか）

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams['font.family'] = 'Hiragino Sans'

# ─────────────────────────────────────────
# 1-1. KPI分析・クロス集計
# ─────────────────────────────────────────
pivot = pd.crosstab(df['category'], df['status'],
                    values=df['amount'], aggfunc='sum', margins=True, normalize='index')
print("クロス集計（行%）:")
print(pivot.to_string())

# ─────────────────────────────────────────
# 1-2. コホート分析
# ─────────────────────────────────────────
# df['cohort'] = df.groupby('customer_id')['date'].transform('min').dt.to_period('M')
# cohort_table = df.groupby(['cohort', df['date'].dt.to_period('M')])['customer_id'].nunique().unstack()
# print("コホート分析（リテンション率）:")
# print((cohort_table.div(cohort_table.iloc[:, 0], axis=0) * 100).round(1).to_string())
```

---

## 2. 診断的分析（なぜ起きたか）

```python
from scipy import stats

# ─────────────────────────────────────────
# 2-1. 正規性確認 → 適切な検定の選択
# ─────────────────────────────────────────
group_a = df[df['group'] == 'A']['metric'].dropna()
group_b = df[df['group'] == 'B']['metric'].dropna()

shapiro_p_a = stats.shapiro(group_a.sample(min(len(group_a), 5000), random_state=42))[1]
shapiro_p_b = stats.shapiro(group_b.sample(min(len(group_b), 5000), random_state=42))[1]

if shapiro_p_a > 0.05 and shapiro_p_b > 0.05:
    stat, p_value = stats.ttest_ind(group_a, group_b, equal_var=False)  # Welch's t-test
    test_name = "Welch's t検定"
else:
    stat, p_value = stats.mannwhitneyu(group_a, group_b, alternative='two-sided')
    test_name = "Mann-Whitney U検定（非正規）"

# 効果量【GL-2】
pooled_std = np.sqrt(((len(group_a)-1)*group_a.std()**2 + (len(group_b)-1)*group_b.std()**2) /
                     (len(group_a) + len(group_b) - 2))
cohen_d = (group_a.mean() - group_b.mean()) / pooled_std

# 信頼区間【GL-4】
diff = group_a.mean() - group_b.mean()
se = np.sqrt(group_a.var()/len(group_a) + group_b.var()/len(group_b))
ci_lower, ci_upper = diff - 1.96*se, diff + 1.96*se

print(f"{test_name}: stat={stat:.4f}, p={p_value:.4f}")
print(f"効果量 Cohen's d = {cohen_d:.3f}")
print(f"  d < 0.2: 実務的に無意味な差【GL-2】")
print(f"  d 0.2〜0.5: 小〜中程度、0.8以上: 大きな差")
print(f"95%信頼区間: [{ci_lower:.3f}, {ci_upper:.3f}]【GL-4】")

if p_value < 0.05:
    print(f"→ 統計的有意差あり (p={p_value:.4f} < 0.05)【GL-1】")
elif 0.05 <= p_value < 0.06:
    print(f"→ 限界的有意 (p={p_value:.4f}) — 注意深く解釈【GL-1】")
else:
    print(f"→ 統計的有意差なし (p={p_value:.4f} ≥ 0.05)【GL-1】")

# ─────────────────────────────────────────
# 2-2. 多重検定の補正【GL-3】
# ─────────────────────────────────────────
from statsmodels.stats.multitest import multipletests

# 複数のp値をまとめて補正
# p_values = [0.03, 0.04, 0.02, 0.06, 0.001]
# reject, p_corrected, _, _ = multipletests(p_values, alpha=0.05, method='fdr_bh')  # Benjamini-Hochberg
# print("多重比較補正（BH-FDR）:")
# for i, (orig, corr, rej) in enumerate(zip(p_values, p_corrected, reject)):
#     print(f"  検定{i+1}: p_orig={orig:.3f} → p_corrected={corr:.3f} → {'棄却' if rej else '保留'}")
```

---

## 3. 予測的分析（機械学習）

### 3-1. クラス不均衡の対応

```python
from imblearn.over_sampling import SMOTE
from imblearn.combine import SMOTETomek
from collections import Counter

print(f"クラス分布: {Counter(y_train)}")
imbalance_ratio = min(Counter(y_train).values()) / max(Counter(y_train).values())
print(f"不均衡比率: {imbalance_ratio:.3f}")

if imbalance_ratio < 0.2:
    print("⚠️ 強いクラス不均衡 → SMOTE+Tomek Links 推奨")
    smote_tomek = SMOTETomek(random_state=42)
    X_train_resampled, y_train_resampled = smote_tomek.fit_resample(X_train_scaled, y_train)
    print(f"SMOTE後: {Counter(y_train_resampled)}")
elif imbalance_ratio < 0.5:
    print("⚠️ クラス不均衡あり → class_weight='balanced' を推奨")
    X_train_resampled, y_train_resampled = X_train_scaled, y_train
else:
    print("クラスバランス良好")
    X_train_resampled, y_train_resampled = X_train_scaled, y_train
```

### 3-2. ベースライン設定【GL-7】

```python
from sklearn.dummy import DummyClassifier, DummyRegressor
from sklearn.metrics import f1_score, mean_absolute_error, r2_score

# 分類: 最頻クラス予測
dummy_clf = DummyClassifier(strategy='most_frequent', random_state=42)
dummy_clf.fit(X_train_resampled, y_train_resampled)
baseline_f1 = f1_score(y_test, dummy_clf.predict(X_test_scaled), average='weighted')
print(f"ベースライン F1: {baseline_f1:.4f}（これを超えないモデルは意味なし）")

# 回帰: 平均値予測
# dummy_reg = DummyRegressor(strategy='mean')
# baseline_mae = mean_absolute_error(y_test, dummy_reg.fit(X_train, y_train).predict(X_test))
# print(f"ベースライン MAE: {baseline_mae:.4f}")
```

### 3-3. 候補モデルの比較（交差検証）

```python
from sklearn.linear_model import LogisticRegression, Ridge, Lasso
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.model_selection import StratifiedKFold, cross_validate
import xgboost as xgb
import lightgbm as lgb

models = {
    'Logistic Regression (L2)': LogisticRegression(C=1.0, random_state=42, max_iter=1000),
    'Random Forest': RandomForestClassifier(n_estimators=200, random_state=42, n_jobs=-1),
    'XGBoost': xgb.XGBClassifier(random_state=42, eval_metric='logloss', verbosity=0),
    'LightGBM': lgb.LGBMClassifier(random_state=42, verbose=-1, n_jobs=-1),
}

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
results = {}

for name, model in models.items():
    cv_results = cross_validate(
        model, X_train_resampled, y_train_resampled, cv=cv,
        scoring=['f1_weighted', 'roc_auc', 'accuracy'],
        return_train_score=True,
        n_jobs=-1
    )
    results[name] = {
        'val_f1': cv_results['test_f1_weighted'].mean(),
        'val_f1_std': cv_results['test_f1_weighted'].std(),
        'val_auc': cv_results['test_roc_auc'].mean(),
        'train_f1': cv_results['train_f1_weighted'].mean(),
        'overfit_gap': cv_results['train_f1_weighted'].mean() - cv_results['test_f1_weighted'].mean()
    }
    print(f"{name:30s}: val_F1={results[name]['val_f1']:.4f} ± {results[name]['val_f1_std']:.4f} "
          f"| AUC={results[name]['val_auc']:.4f} | 過学習Gap={results[name]['overfit_gap']:.3f}")
    if results[name]['overfit_gap'] > 0.1:
        print(f"  ⚠️ 過学習の疑い → 正則化強化・深さ制限・Dropoutを検討")
```

### 3-4. ハイパーパラメータ最適化（Optuna）

```python
import optuna
optuna.logging.set_verbosity(optuna.logging.WARNING)

def objective(trial):
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 50, 500),
        'max_depth': trial.suggest_int('max_depth', 3, 10),
        'learning_rate': trial.suggest_float('learning_rate', 1e-3, 0.3, log=True),
        'subsample': trial.suggest_float('subsample', 0.5, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.5, 1.0),
        'min_child_weight': trial.suggest_int('min_child_weight', 1, 10),
        'reg_alpha': trial.suggest_float('reg_alpha', 1e-8, 1.0, log=True),
        'reg_lambda': trial.suggest_float('reg_lambda', 1e-8, 1.0, log=True),
        'random_state': 42, 'eval_metric': 'logloss', 'verbosity': 0
    }
    model = xgb.XGBClassifier(**params)
    cv_scores = cross_validate(model, X_train_resampled, y_train_resampled,
                                cv=StratifiedKFold(n_splits=5, shuffle=True, random_state=42),
                                scoring='f1_weighted', n_jobs=-1)
    return cv_scores['test_score'].mean()

study = optuna.create_study(direction='maximize',
                             pruner=optuna.pruners.MedianPruner())
study.optimize(objective, n_trials=50, timeout=300)  # 50試行 or 5分

print(f"\nOptuna最適化完了: Best F1 = {study.best_value:.4f}")
print(f"最適パラメータ: {study.best_params}")

# 最良モデルを構築
best_model = xgb.XGBClassifier(**study.best_params, random_state=42)
best_model.fit(X_train_resampled, y_train_resampled)
```

### 3-5. 最終評価（テストデータで一度のみ）

```python
from sklearn.metrics import (classification_report, confusion_matrix,
                              roc_auc_score, average_precision_score,
                              brier_score_loss)

y_pred = best_model.predict(X_test_scaled)
y_prob = best_model.predict_proba(X_test_scaled)[:, 1]

print("=== 最終評価（テストデータ）===")
print(classification_report(y_test, y_pred))
print(f"AUC-ROC:   {roc_auc_score(y_test, y_prob):.4f}")
print(f"AP (PR曲線): {average_precision_score(y_test, y_prob):.4f}")
print(f"Brier Score: {brier_score_loss(y_test, y_prob):.4f} (低いほど良い)")

# ─────────────────────────────────────────
# 確率キャリブレーション（重要）
# ─────────────────────────────────────────
from sklearn.calibration import calibration_curve, CalibratedClassifierCV

fraction_of_positives, mean_predicted_value = calibration_curve(y_test, y_prob, n_bins=10)

plt.figure(figsize=(8, 6))
plt.plot(mean_predicted_value, fraction_of_positives, 's-', label='モデル')
plt.plot([0, 1], [0, 1], 'k--', label='完全キャリブレーション')
plt.xlabel('予測確率')
plt.ylabel('実際の正例率')
plt.title('キャリブレーション曲線\n（対角線に近いほど確率が信頼できる）')
plt.legend()
plt.savefig('data/docs/06_calibration.png', dpi=150)
plt.close()

# キャリブレーションが悪い場合は補正
# calibrated_model = CalibratedClassifierCV(best_model, cv='prefit', method='isotonic')
# calibrated_model.fit(X_val_scaled, y_val)

# 混同行列
cm = confusion_matrix(y_test, y_pred)
plt.figure(figsize=(7, 6))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues')
plt.title('混同行列')
plt.ylabel('実際')
plt.xlabel('予測')
plt.savefig('data/docs/06_confusion_matrix.png', dpi=150)
plt.close()
```

---

## 4. 時系列予測モデル

```python
# ─────────────────────────────────────────
# 4-1. Prophet（Meta）: トレンド + 季節性
# ─────────────────────────────────────────
try:
    from prophet import Prophet
    from prophet.diagnostics import cross_validation, performance_metrics

    # データ準備（Prophetの入力形式: ds, y）
    # df_prophet = df[['date', 'value']].rename(columns={'date': 'ds', 'value': 'y'})

    # モデル設定
    model = Prophet(
        seasonality_mode='multiplicative',    # 季節性を乗法モデルで（変動が増加傾向の場合）
        yearly_seasonality=True,
        weekly_seasonality=True,
        daily_seasonality=False,
        interval_width=0.95,                  # 95%信頼区間【GL-4】
        changepoint_prior_scale=0.05,         # トレンド変化点の感度
    )

    # カスタム季節性の追加（例: 四半期）
    # model.add_seasonality(name='quarterly', period=91.25, fourier_order=5)

    # 日本の祝日を追加（該当する場合）
    # model.add_country_holidays(country_name='JP')

    model.fit(df_prophet)

    # 予測（90日先）
    future = model.make_future_dataframe(periods=90)
    forecast = model.predict(future)

    print("Prophet予測完了:")
    print(forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].tail(10).to_string())

    # 交差検証
    df_cv = cross_validation(
        model,
        initial='365 days',  # 初期学習期間
        period='30 days',    # CV間隔
        horizon='90 days',   # 予測期間
        parallel='processes'
    )
    perf = performance_metrics(df_cv)
    print(f"\nProphet交差検証:")
    print(f"  MAPE:  {perf['mape'].mean():.3f} ({perf['mape'].mean()*100:.1f}%)")
    print(f"  RMSE:  {perf['rmse'].mean():.3f}")
    print(f"  MAE:   {perf['mae'].mean():.3f}")

    # 可視化
    fig = model.plot(forecast)
    plt.title('Prophet予測（実績値 + 将来予測 + 95%CI）')
    plt.savefig('data/docs/06_prophet_forecast.png', dpi=150)
    plt.close()

    fig2 = model.plot_components(forecast)
    plt.savefig('data/docs/06_prophet_components.png', dpi=150)
    plt.close()

except ImportError:
    print("Prophet 未インストール。pip install prophet で追加可能。")

# ─────────────────────────────────────────
# 4-2. SARIMA: 統計的時系列モデル
# ─────────────────────────────────────────
import pmdarima as pm
from statsmodels.tsa.statespace.sarimax import SARIMAX

# auto_arima でパラメータを自動探索
try:
    auto_model = pm.auto_arima(
        ts_series,
        start_p=0, start_q=0,
        max_p=3, max_q=3,
        m=7,             # 季節周期（週次の場合7）
        seasonal=True,
        d=None,          # 差分次数を自動決定
        D=None,          # 季節差分次数を自動決定
        stepwise=True,
        information_criterion='aic',
        trace=True,
        error_action='ignore',
        suppress_warnings=True,
        random_state=42
    )
    print(f"\nSARIMA最良モデル: {auto_model.order} × {auto_model.seasonal_order}")
    print(f"AIC: {auto_model.aic():.2f}")

    # 予測
    forecast_sarima, conf_int = auto_model.predict(n_periods=30, return_conf_int=True)
    print("SARIMA予測（30期先）:")
    for i, (yhat, ci) in enumerate(zip(forecast_sarima, conf_int)):
        print(f"  t+{i+1}: {yhat:.2f} [{ci[0]:.2f}, {ci[1]:.2f}]")

except ImportError:
    print("pmdarima 未インストール。pip install pmdarima で追加可能。")

# ─────────────────────────────────────────
# 4-3. 指数平滑化（Holt-Winters）
# ─────────────────────────────────────────
from statsmodels.tsa.holtwinters import ExponentialSmoothing

hw_model = ExponentialSmoothing(
    ts_series,
    trend='add',
    seasonal='add',
    seasonal_periods=7,  # 季節周期
).fit(optimized=True)

forecast_hw = hw_model.forecast(30)
print(f"\nHolt-Winters予測完了: 30期先")

# ─────────────────────────────────────────
# 4-4. 時系列モデル比較
# ─────────────────────────────────────────
from sklearn.metrics import mean_absolute_percentage_error as mape

# Train/Test分割（時系列は時間順分割）
split = int(len(ts_series) * 0.8)
ts_train, ts_test = ts_series[:split], ts_series[split:]

print("\n=== 時系列モデル比較（テスト期間） ===")
print(f"評価期間: {ts_test.index[0]} 〜 {ts_test.index[-1]} ({len(ts_test)}点)")
# 各モデルのMAPE・RMSEを比較表にまとめる
```

---

## 5. 教師なし学習

```python
from sklearn.cluster import KMeans, DBSCAN
from sklearn.metrics import silhouette_score, davies_bouldin_score
from sklearn.decomposition import PCA
import umap

# ─────────────────────────────────────────
# 5-1. 最適クラスター数の探索（Elbow + Silhouette）
# ─────────────────────────────────────────
inertia_list, silhouette_list = [], []
K_range = range(2, 12)

for k in K_range:
    km = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = km.fit_predict(X_scaled)
    inertia_list.append(km.inertia_)
    silhouette_list.append(silhouette_score(X_scaled, labels))
    print(f"k={k}: Inertia={km.inertia_:.0f}, Silhouette={silhouette_list[-1]:.4f}")

optimal_k = K_range[silhouette_list.index(max(silhouette_list))]
print(f"\n推奨クラスター数 k={optimal_k} (Silhouette最大: {max(silhouette_list):.4f})")
print("注意: 統計指標と合わせてビジネス解釈可能性も考慮すること")

# ─────────────────────────────────────────
# 5-2. UMAP による可視化（t-SNEより保持的）
# ─────────────────────────────────────────
try:
    reducer = umap.UMAP(n_components=2, random_state=42, n_neighbors=15, min_dist=0.1)
    embedding = reducer.fit_transform(X_scaled)

    plt.figure(figsize=(10, 8))
    scatter = plt.scatter(embedding[:, 0], embedding[:, 1],
                          c=km.labels_, cmap='tab10', alpha=0.7, s=20)
    plt.colorbar(scatter)
    plt.title(f'UMAP 2D投影（k={optimal_k}クラスタ）')
    plt.savefig('data/docs/06_umap_clusters.png', dpi=150)
    plt.close()
except ImportError:
    print("umap-learn 未インストール。pip install umap-learn で追加可能。")
```

---

## 6. 因果推論（観察データ）

```python
# ─────────────────────────────────────────
# 6-1. 傾向スコアマッチング（PSM）
# ─────────────────────────────────────────
# 目的: 選択バイアスを除去して平均処置効果（ATE）を推定
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler

# Step 1: 傾向スコアの推定
# 介入(treatment=1/0)を被説明変数にロジスティック回帰
# ps_model = LogisticRegression(random_state=42)
# ps_model.fit(X_confounders, treatment)
# propensity_scores = ps_model.predict_proba(X_confounders)[:, 1]

# Step 2: 重複サポートの確認（positivity assumption）
# 介入群と対照群の傾向スコア分布が重なっているか確認

# Step 3: マッチング or IPW (逆確率重み付け)
# from causalinference import CausalModel
# model = CausalModel(outcome, treatment, covariates)
# model.est_via_matching()
# print(model.estimates)

print("因果推論手法の選択指針:")
print("  実験データ（RCT） → 単純平均比較")
print("  観察データ + 選択バイアス → PSM/IPW")
print("  政策変化・外生的ショック → DID")
print("  不連続な閾値 → RDD")
print("  操作変数が存在 → IV推定")
print("\n注意: 相関から因果を主張するには上記の設計が必要【GL-9】")
```

---

## 7. モデル解釈性（SHAP）

```python
import shap

# ─────────────────────────────────────────
# 7-1. SHAP グローバル解釈（全体的な特徴量重要度）
# ─────────────────────────────────────────
explainer = shap.TreeExplainer(best_model)
shap_values = explainer(X_test_scaled)  # 新しいAPI（Explanation オブジェクト）

# Summary Plot（Gini重要度より信頼性が高い）
shap.summary_plot(shap_values, X_test, feature_names=feature_names,
                  show=False, max_display=20)
plt.title("SHAP Summary Plot（グローバル特徴量重要度）")
plt.tight_layout()
plt.savefig('data/docs/06_shap_summary.png', dpi=150, bbox_inches='tight')
plt.close()

# ─────────────────────────────────────────
# 7-2. SHAP Waterfall Plot（個別予測の説明）
# ─────────────────────────────────────────
# 最も外れた予測を選んで解釈
most_confident_idx = abs(shap_values.values).sum(axis=1).argmax()
shap.waterfall_plot(shap_values[most_confident_idx], show=False, max_display=15)
plt.title("SHAP Waterfall Plot（代表的な予測の説明）")
plt.tight_layout()
plt.savefig('data/docs/06_shap_waterfall.png', dpi=150, bbox_inches='tight')
plt.close()

# ─────────────────────────────────────────
# 7-3. SHAP Dependence Plot（特徴量と予測値の関係）
# ─────────────────────────────────────────
# top3特徴量の依存性プロット
top_features_idx = abs(shap_values.values).mean(axis=0).argsort()[::-1][:3]
fig, axes = plt.subplots(1, 3, figsize=(18, 5))
for i, feat_idx in enumerate(top_features_idx):
    shap.dependence_plot(feat_idx, shap_values.values, X_test,
                         feature_names=feature_names, ax=axes[i], show=False)
    axes[i].set_title(f'SHAP Dependence: {feature_names[feat_idx]}')
plt.suptitle("SHAP Dependence Plots（特徴量値と貢献度の関係）")
plt.tight_layout()
plt.savefig('data/docs/06_shap_dependence.png', dpi=150)
plt.close()
print("SHAP分析完了: data/docs/06_shap_*.png")
```

---

## 8. 分析結果サマリー

```
【Issue】（analysis_context.md より再掲）

【結論】（1〜2文で端的に）

【根拠】
 1. Key Finding A — Statistical Evidence（p=○○, d=○○, 95%CI[○○, ○○]）【GL-1, GL-2, GL-4】
 2. Key Finding B — 時系列トレンド（MAPE=○○%, STL分解より）
 3. Key Finding C — SHAP重要度上位（特徴量: ○○, ○○, ○○）

【バイアスチェック】【GL-7, GL-9】
 - 選択した証拠に対し、仮説を否定する証拠も確認した: □
 - 相関と因果を混同していない: □
 - 多重比較補正（BH-FDR）を適用した: □

【リスク・前提】
 - 確度高: ○○
 - 蓋然性あり: ○○（前提: ○○）
 - 仮説段階: ○○
 - 不確実性: ○○（確度: 低）

【Next Step】
→ /data-analysis:data-interpret で結果解釈・レポート作成に進む
```

---

分析結果を `analysis_context.md` の「分析経過メモ」に追記し、`data/docs/06_modeling_report.md` にレポートを保存すること。

$ARGUMENTS

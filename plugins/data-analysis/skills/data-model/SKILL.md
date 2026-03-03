---
name: data-model
description: 分析・モデリングを実行します（Phase 6）。「モデルを作りたい」「機械学習を使いたい」「統計分析して」「クラスタリングして」と言われたら使用してください。
---

# Phase 6: 分析・モデリングスキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 0. 分析手法の選択

`analysis_context.md` の目的を確認し、適切な分析段階・手法を選択する:

```
分析の4段階:
1. 記述的分析: 何が起きたか（現状把握）
2. 診断的分析: なぜ起きたか（原因特定）
3. 予測的分析: 何が起きるか（将来予測）
4. 処方的分析: 何をすべきか（最適化）

手法選択フロー:
目的変数がある？
├── Yes → 教師あり学習
│   ├── 連続値 → 回帰
│   ├── 二値  → 分類
│   └── 多クラス → 多クラス分類
└── No → 教師なし学習
    ├── グルーピング → クラスタリング
    ├── 次元削減 → PCA/t-SNE/UMAP
    └── 異常検知 → Isolation Forest/LOF

因果推論が必要？
├── 実験可能 → A/Bテスト
└── 観察データ → DID/RDD/傾向スコアマッチング
```

---

## 1. 記述的分析（何が起きたか）

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams['font.family'] = 'Hiragino Sans'

# クロス集計
pivot = pd.crosstab(df['category'], df['status'],
                     values=df['amount'], aggfunc='sum', margins=True)
print(pivot)

# コホート分析（時系列の場合）
df['cohort'] = df.groupby('customer_id')['date'].transform('min').dt.to_period('M')
cohort_data = df.groupby(['cohort', df['date'].dt.to_period('M')])['customer_id'].nunique()
```

## 2. 診断的分析（なぜ起きたか）

```python
from scipy import stats

# 二群間の検定【GL-1, GL-3】
group_a = df[df['group'] == 'A']['metric']
group_b = df[df['group'] == 'B']['metric']

# 正規性確認
stat_a, p_a = stats.shapiro(group_a)
stat_b, p_b = stats.shapiro(group_b)

if p_a > 0.05 and p_b > 0.05:
    # 正規分布が仮定できる場合: t検定
    stat, p_value = stats.ttest_ind(group_a, group_b)
    test_name = 't検定'
else:
    # 非正規の場合: Mann-Whitney U検定
    stat, p_value = stats.mannwhitneyu(group_a, group_b, alternative='two-sided')
    test_name = 'Mann-Whitney U検定'

# 効果量【GL-2】
cohen_d = (group_a.mean() - group_b.mean()) / np.sqrt(
    ((len(group_a)-1)*group_a.std()**2 + (len(group_b)-1)*group_b.std()**2) /
    (len(group_a) + len(group_b) - 2)
)

# 信頼区間【GL-4】
ci = stats.t.interval(0.95, df=len(group_a)+len(group_b)-2,
                       loc=group_a.mean()-group_b.mean(),
                       scale=stats.sem(np.concatenate([group_a, group_b])))

print(f"{test_name}: stat={stat:.4f}, p={p_value:.4f}")
print(f"効果量 Cohen's d = {cohen_d:.3f}")
print(f"95%信頼区間: [{ci[0]:.3f}, {ci[1]:.3f}]")

# p値の解釈【GL-1】
if p_value < 0.05:
    print("→ 統計的有意差あり（p < 0.05）")
elif 0.05 <= p_value < 0.06:
    print("→ 限界的有意（0.05 ≤ p < 0.06）【GL-1】")
else:
    print("→ 統計的有意差なし（p ≥ 0.05）")
```

## 3. 予測的分析（回帰・分類）

### 3.1 ベースライン設定【GL-7】

```python
# 必ずベースラインを先に設定する
from sklearn.dummy import DummyClassifier, DummyRegressor
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score
from sklearn.metrics import rmse_score, mean_absolute_error, r2_score

# 分類のベースライン（最頻クラス予測）
dummy = DummyClassifier(strategy='most_frequent', random_state=42)
dummy.fit(X_train, y_train)
baseline_score = f1_score(y_test, dummy.predict(X_test), average='weighted')
print(f"ベースライン F1: {baseline_score:.4f}")
```

### 3.2 候補モデルの比較

```python
from sklearn.linear_model import LogisticRegression, Ridge, Lasso
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import StratifiedKFold, cross_validate
import xgboost as xgb
import lightgbm as lgb

models = {
    'Logistic Regression': LogisticRegression(random_state=42, max_iter=1000),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
    'XGBoost': xgb.XGBClassifier(random_state=42, eval_metric='logloss'),
    'LightGBM': lgb.LGBMClassifier(random_state=42, verbose=-1),
}

# 交差検証
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
results = {}

for name, model in models.items():
    cv_results = cross_validate(
        model, X_train_scaled, y_train, cv=cv,
        scoring=['f1_weighted', 'roc_auc'],
        return_train_score=True
    )
    results[name] = {
        'val_f1': cv_results['test_f1_weighted'].mean(),
        'val_f1_std': cv_results['test_f1_weighted'].std(),
        'train_f1': cv_results['train_f1_weighted'].mean(),
    }
    print(f"{name}: val_F1={results[name]['val_f1']:.4f} ± {results[name]['val_f1_std']:.4f} "
          f"(train={results[name]['train_f1']:.4f})")
```

### 3.3 多重比較補正【GL-3】

```python
# Bonferroni補正
n_models = len(models)
alpha_adjusted = 0.05 / n_models
print(f"Bonferroni補正後の有意水準: {alpha_adjusted:.4f} ({n_models}モデル比較)")
```

### 3.4 過学習・未学習の診断

```python
for name, r in results.items():
    gap = r['train_f1'] - r['val_f1']
    if gap > 0.1:
        print(f"⚠️ {name}: 過学習の疑い（gap={gap:.3f}）→ 正則化・Dropout検討")
    elif r['val_f1'] < baseline_score + 0.05:
        print(f"⚠️ {name}: 未学習の疑い → モデル複雑化・特徴量追加検討")
    else:
        print(f"✓ {name}: 良好なバランス")
```

### 3.5 最良モデルの最終評価

```python
# 最良モデルでテストデータを評価（一度のみ）
best_model = models['XGBoost']
best_model.fit(X_train_scaled, y_train)
y_pred = best_model.predict(X_test_scaled)

print("=== テストデータでの最終評価 ===")
print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
print(f"F1 (weighted): {f1_score(y_test, y_pred, average='weighted'):.4f}")
print(f"AUC-ROC: {roc_auc_score(y_test, best_model.predict_proba(X_test_scaled)[:,1]):.4f}")
```

## 4. 教師なし分析

### クラスタリング

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score

# 最適クラスター数の探索
silhouette_scores = []
for k in range(2, 11):
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(X_scaled)
    score = silhouette_score(X_scaled, labels)
    silhouette_scores.append(score)
    print(f"k={k}: Silhouette={score:.4f}")
```

## 5. 因果推論（観察データの場合）

```python
# 傾向スコアマッチング
# DID（差分の差分法）
# RDD（回帰不連続デザイン）
# 使用する手法と理由を必ず明記すること【GL-9】
print("因果推論手法: ○○ を使用")
print("理由: ○○")
print("前提条件: ○○")
print("限界: ○○")
```

## 6. 分析結果サマリー

```
【Issue】（analysis_context.md より再掲）
【結論】端的な回答（1-2文）
【根拠】
 1. Key Finding A — Statistical Evidence【GL-1, GL-2, GL-4】
 2. Key Finding B — Statistical Evidence
 3. Key Finding C — Statistical Evidence
【リスク・前提】
 - 確度高: ○○
 - 蓋然性あり: ○○
 - 仮説段階: ○○
 - 不確実性: ○○（必要に応じて）
【Next Step】
 → /data-analysis:data-interpret で結果解釈・レポート作成に進む
```

---

分析結果を `analysis_context.md` の「分析経過メモ」に追記し、`data/docs/06_modeling_report.md` にレポートを保存すること。

$ARGUMENTS

---
name: data-interpret
description: 分析結果の解釈・示唆出し・検証・レポート作成を行います（Phase 7-8）。「結果を解釈して」「レポートを作りたい」「示唆を出して」「検証して」と言われたら使用してください。
---

# Phase 7-8: 結果解釈・検証・レポートスキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## Phase 7: 結果解釈・示唆出し

### 1. Fact → Insight → Action → Impact フレームワーク

分析結果を以下の構造で整理する（ファクトの羅列で終わることを禁止）:

| 層 | 内容 | 例 |
|----|------|-----|
| **Fact** | 観察された事実（数値） | 「解約率が前年比+5%pt」 |
| **Insight** | ファクトから導かれる洞察（So What?） | 「30代男性セグメントが解約を牽引している」 |
| **Action** | 具体的な打ち手（誰が/何を/いつまでに） | 「○月までにリテンションキャンペーンを実施」 |
| **Impact** | 期待される効果（定量的）【GL-2, GL-4】 | 「解約率を-2%pt改善、年間○○万円の収益改善」 |

### 2. 禁止事項の確認【GL-7, GL-9】

- [ ] 相関と因果を混同していないか
- [ ] 都合の良い証拠だけを使っていないか（チェリーピッキング）
- [ ] 仮説を否定する証拠も同等に提示したか
- [ ] p-hacking（結果が出るまで検定を繰り返す）を行っていないか
- [ ] シンプソンのパラドックスを確認したか（サブグループ分析の実施）

```python
# シンプソンのパラドックスの確認
# 全体と各サブグループで傾向が一致しているか確認
overall = df.groupby('group')['metric'].mean()
by_subgroup = df.groupby(['group', 'subgroup'])['metric'].mean()
print("全体:", overall)
print("サブグループ別:", by_subgroup)
```

### 3. 限界・前提条件の明記

```
【限界・前提条件】
1. データの制約: ○○期間のデータのみ（外挿には注意）
2. 因果の問題: 観察データのため因果推論には限界がある
3. 外的妥当性: ○○地域・○○セグメントのデータのみ
4. モデルの前提: ○○の仮定が成立する場合のみ有効
5. 不確実性: ○○（確度: 仮説段階）
```

### 4. 反事実の検討

「もし○○だったら」の問いに答える:
- A/Bテスト実施前: 「介入なしの場合の推計結果」
- 感度分析: 「主要な前提が変わった場合の結果の変化」

---

## Phase 8: 検証・バリデーション

### 1. 再現性の確認

```python
# 乱数シードの固定確認
import random, numpy as np, os

random.seed(42)
np.random.seed(42)
os.environ['PYTHONHASHSEED'] = '42'

# スクリプトを再実行して同じ結果が得られるか確認
```

### 2. 頑健性の検証

```python
# パラメータ変動に対する安定性確認
results_sensitivity = {}
for n_estimators in [50, 100, 200]:
    model = RandomForestClassifier(n_estimators=n_estimators, random_state=42)
    score = cross_val_score(model, X_train, y_train, cv=5, scoring='f1_weighted').mean()
    results_sensitivity[n_estimators] = score
    print(f"n_estimators={n_estimators}: F1={score:.4f}")

variation = max(results_sensitivity.values()) - min(results_sensitivity.values())
print(f"パラメータ変動による結果の変動幅: {variation:.4f}")
if variation > 0.05:
    print("⚠️ 結果が不安定: モデルの頑健性に注意が必要")
```

### 3. 交差検証の最終確認

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score

# K-Fold交差検証
cv = StratifiedKFold(n_splits=10, shuffle=True, random_state=42)
scores = cross_val_score(best_model, X, y, cv=cv, scoring='f1_weighted')
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")
print(f"95%CI: [{scores.mean()-1.96*scores.std():.4f}, {scores.mean()+1.96*scores.std():.4f}]")
```

### 4. 常識チェック（ドメイン知識との整合）

```python
# 予測結果がビジネス的に妥当かを確認
print("=== 常識チェック ===")
# 例: 解約予測モデルの場合
print(f"予測解約率: {y_pred.mean():.1%}")
print(f"実際の解約率: {y_test.mean():.1%}")
print(f"乖離: {abs(y_pred.mean() - y_test.mean()):.1%}")
# ビジネス知識と照合して「あり得ない数字」でないか確認
```

---

## レポート作成

### エグゼクティブサマリー（非技術者向け）

`data/docs/07_executive_summary.md` に以下を出力:

```markdown
# エグゼクティブサマリー

## 分析目的
[1〜2文で分析の目的を記述]

## 主要な発見（3点以内）
1. **[最重要発見]**: [具体的な数値と意味]
2. **[2番目の発見]**: [具体的な数値と意味]
3. **[3番目の発見]**: [具体的な数値と意味]

## 推奨アクション
| 優先度 | アクション | 期待効果 | 担当 | 期限 |
|--------|-----------|---------|------|------|
| 高 | ○○を実施する | ○○%改善 | ○○部門 | ○月 |

## 限界・注意事項
- [結論が覆りうる条件]
- [外挿・一般化の限界]

## 次のステップ（Next Step）
誰が: ○○
何を: ○○
いつまでに: ○○
```

### 技術的詳細レポート（データサイエンティスト向け）

`data/docs/08_technical_report.md` に以下を出力:

```markdown
# 技術的詳細レポート

## 使用データとスコープ
- データソース: ○○
- 期間: ○○〜○○（○件）
- 除外基準: ○○

## 手法と選択根拠
- 分析手法: ○○（選択理由: ○○）
- 交差検証: ○○（理由: ○○）
- 評価指標: ○○（選択理由: ○○）

## 評価指標の全数値
| 指標 | Train | Validation | Test |
|------|-------|-----------|------|
| F1 | | | |
| AUC | | | |

## 限界・前提条件
（詳細版）

## 分析スクリプト
`data/XX_analysis.py` を参照
```

### 可視化【GL-8, GL-4】

```python
import matplotlib.pyplot as plt
import seaborn as sns

plt.rcParams['font.family'] = 'Hiragino Sans'

# 1. ROC曲線（分類の場合）
from sklearn.metrics import RocCurveDisplay
RocCurveDisplay.from_predictions(y_test, y_prob, name='最良モデル')
plt.title('ROC曲線（AUC付き）')
# 軸は0始まり【GL-8】
plt.savefig('data/docs/07_roc_curve.png', dpi=150)

# 2. SHAP値（特徴量重要度）
import shap
explainer = shap.TreeExplainer(best_model)
shap_values = explainer.shap_values(X_test)
shap.summary_plot(shap_values, X_test, feature_names=feature_names,
                   show=False)
plt.savefig('data/docs/07_shap_importance.png', dpi=150)

# 3. 残差プロット（回帰の場合）
residuals = y_test - y_pred
plt.figure(figsize=(8, 6))
plt.scatter(y_pred, residuals, alpha=0.5)
plt.axhline(y=0, color='r', linestyle='--')
plt.xlabel('予測値')
plt.ylabel('残差')
plt.title('残差プロット')
plt.savefig('data/docs/07_residuals.png', dpi=150)
```

## Next Step の最終提示

```
【Issue】（最終回答）
【結論】
【根拠】（3点以内に絞る）
【リスク・前提】
【Next Step】
 誰が: [具体的な担当者・部門]
 何を: [具体的なアクション（優先度 High → Medium → Low の順）]
 いつまでに: [具体的な期限]
 期待効果: [定量的な成果目標]【GL-2, GL-4】
```

---

完成したレポートを `analysis_context.md` の「分析経過メモ」に記録し、成果物一覧を更新すること。

$ARGUMENTS

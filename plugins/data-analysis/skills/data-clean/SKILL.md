---
name: data-clean
description: シニアデータエンジニアレベルのデータ品質評価・クレンジング・構造化を行います（Phase 2）。「データをきれいにしたい」「前処理したい」「名寄せしたい」「日本語の表記揺れを修正したい」「データ構造を整えたい」と言われたら使用してください。
---

# Phase 2: データ品質評価・クレンジング・構造化スキル（Senior DE レベル）

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 0. クレンジング戦略の策定（最初に実行）

分析目的に応じてクレンジングの優先順位を決める。**過剰なクレンジングは情報損失につながる。**

```
クレンジング設計の原則:
1. 元データは必ずバックアップして変更しない（rawデータの不変性）
2. すべてのクレンジング操作を記録（データリネージ）
3. 各操作の前後でレコード数・統計量を比較【GL-7】
4. 操作の根拠をコメントに明記（「なぜその閾値か」）【GL-6】
5. 不確かな判断は「不確実性」として記録し、ドメイン専門家に確認
```

```python
import pandas as pd
import numpy as np
import warnings
warnings.filterwarnings('ignore')

# バックアップ（原則）
df_raw = df.copy()
print(f"元データ: {len(df_raw):,} 件 × {len(df_raw.columns)} 列")
print("バックアップ完了: df_raw")

# クレンジングログ
cleaning_log = []
def log_step(step, before, after, method, notes=""):
    diff = before - after
    cleaning_log.append({
        'Step': step,
        '処理前': before,
        '処理後': after,
        '削除/変更数': diff,
        '削除率(%)': f"{diff/before*100:.2f}%" if before > 0 else "0%",
        '手法': method,
        '備考': notes
    })
    print(f"[{step}] {method}: {before:,}→{after:,} ({diff:,}件削除)")
```

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

---

## 2. 型推論と型強制（シニアDE必須）

```python
# ─────────────────────────────────────────
# 2-1. データ型の自動診断と修正提案
# ─────────────────────────────────────────
import re

type_issues = []

for col in df.columns:
    current_type = str(df[col].dtype)
    sample = df[col].dropna().head(100)

    # object型なのに数値が格納されているケース
    if current_type == 'object':
        # 数値として解析できるか
        numeric_rate = pd.to_numeric(sample, errors='coerce').notna().mean()
        if numeric_rate > 0.8:
            type_issues.append({'列': col, '現在': 'object', '推奨': 'numeric', '変換率': f"{numeric_rate:.1%}"})

        # 日付として解析できるか
        try:
            date_rate = pd.to_datetime(sample, errors='coerce', infer_datetime_format=True).notna().mean()
            if date_rate > 0.8:
                type_issues.append({'列': col, '現在': 'object', '推奨': 'datetime', '変換率': f"{date_rate:.1%}"})
        except Exception:
            pass

    # int64なのに bool 的な値（0/1のみ）
    if current_type in ('int64', 'float64'):
        uniq = df[col].dropna().unique()
        if set(uniq).issubset({0, 1, True, False}):
            type_issues.append({'列': col, '現在': current_type, '推奨': 'bool', '変換率': '100%'})

if type_issues:
    print("=== 型不整合の検出 ===")
    print(pd.DataFrame(type_issues).to_string())
    print("→ 以下でユーザーに確認の上、型変換を実施する")

# ─────────────────────────────────────────
# 2-2. 型変換の実行
# ─────────────────────────────────────────
df_typed = df.copy()

for col in df_typed.columns:
    if df_typed[col].dtype == 'object':
        # 数値変換試行
        converted = pd.to_numeric(df_typed[col], errors='coerce')
        if converted.notna().mean() > 0.8:
            df_typed[col] = converted
            print(f"型変換: {col} → numeric (変換率 {converted.notna().mean():.1%})")
            continue
        # 日付変換試行
        try:
            converted = pd.to_datetime(df_typed[col], errors='coerce', infer_datetime_format=True)
            if converted.notna().mean() > 0.8:
                df_typed[col] = converted
                print(f"型変換: {col} → datetime (変換率 {converted.notna().mean():.1%})")
        except Exception:
            pass

df = df_typed
```

---

## 3. 日本語文字列の正規化（シニアDE必須）

```python
import unicodedata

# ─────────────────────────────────────────
# 3-1. NFKC正規化（全角/半角統一・特殊文字処理）
# ─────────────────────────────────────────
def normalize_japanese(text):
    """
    日本語文字列の標準的な正規化処理
    - NFKC: 全角数字/英字 → 半角、半角カナ → 全角カナ
    - 前後の空白除去
    - 重複空白の除去
    """
    if pd.isna(text) or not isinstance(text, str):
        return text
    # NFKC正規化: ① → 1, Ａ → A, ｱ → ア, ﾆ → ニ など
    text = unicodedata.normalize('NFKC', text)
    text = text.strip()
    text = re.sub(r'\s+', ' ', text)  # 重複空白を単一に
    return text

str_cols = df.select_dtypes(include='object').columns.tolist()
for col in str_cols:
    before_unique = df[col].nunique()
    df[col] = df[col].apply(normalize_japanese)
    after_unique = df[col].nunique()
    if before_unique != after_unique:
        print(f"NFKC正規化 {col}: {before_unique} → {after_unique} ユニーク値 ({before_unique-after_unique}件統合)")

# ─────────────────────────────────────────
# 3-2. 表記揺れの統一（ファジーマッチング）
# ─────────────────────────────────────────
try:
    from rapidfuzz import fuzz, process

    def find_variants(series, threshold=85):
        """類似文字列のグループを検出する"""
        unique_vals = series.dropna().unique().tolist()
        groups = []
        visited = set()

        for val in unique_vals:
            if val in visited:
                continue
            matches = process.extract(val, unique_vals, scorer=fuzz.ratio,
                                      limit=None, score_cutoff=threshold)
            group = [m[0] for m in matches if m[0] != val]
            if group:
                groups.append({'代表値': val, '表記揺れ候補': group})
                visited.update(group)
        return groups

    print("\n=== 表記揺れ候補（要ドメイン確認）===")
    for col in str_cols:
        if df[col].nunique() <= 500:
            variants = find_variants(df[col])
            if variants:
                print(f"\n{col}:")
                for g in variants[:5]:
                    print(f"  「{g['代表値']}」の揺れ: {g['表記揺れ候補'][:3]}")
                print("  → 上記をドメイン専門家と確認し、必要に応じてマッピング辞書を作成")

except ImportError:
    print("rapidfuzz 未インストール。pip install rapidfuzz で名寄せ機能を有効化できます。")
```

---

## 4. クレンジング順序（必ずこの順で実施）

### Step 1: 重複削除

```python
n_before = len(df)
df = df.drop_duplicates()
log_step("重複削除（完全一致）", n_before, len(df), "drop_duplicates()")

# ビジネスキーに基づく重複確認
pk_col = None  # 主キー列名を指定
if pk_col and pk_col in df.columns:
    pk_dups = df.duplicated(subset=[pk_col], keep=False)
    if pk_dups.sum() > 0:
        print(f"\n⚠️ 主キー重複: {pk_dups.sum()}件")
        print("→ 最新レコードを残す / 集約するかをドメイン専門家と判断")
        # 例: 最新タイムスタンプを残す
        # df = df.sort_values('updated_at').drop_duplicates(subset=[pk_col], keep='last')
```

### Step 2: 欠損値処理

```python
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer, KNNImputer

missing_rate = df.isnull().mean()
print("\n=== 欠損値処理方針 ===")

for col in df.columns[df.isnull().any()]:
    rate = missing_rate[col]
    dtype = str(df[col].dtype)

    if rate == 0:
        continue
    elif rate < 0.01:
        strategy = "中央値/最頻値補完（欠損率1%未満）"
    elif rate < 0.05:
        strategy = "KNN補完推奨（欠損率5%未満）"
    elif rate < 0.20:
        strategy = "MICE（多重代入法）推奨（欠損率5〜20%）"
    elif rate < 0.50:
        strategy = "欠損フラグ変数を追加 + モデルベース補完"
    else:
        strategy = f"変数の除外を検討（欠損率{rate:.0%} — 情報不足）"

    print(f"  {col} ({dtype}): {rate:.1%} → {strategy}")

# MCAR → 中央値/最頻値補完（軽度欠損の数値列）
low_missing_num = [c for c in df.select_dtypes(include='number').columns
                   if 0 < df[c].isna().mean() < 0.05]
if low_missing_num:
    for col in low_missing_num:
        df[col] = df[col].fillna(df[col].median())
        print(f"中央値補完: {col}")

# MAR → MICE（IterativeImputer = scikit-learnのMICE実装）
mid_missing_num = [c for c in df.select_dtypes(include='number').columns
                   if 0.05 <= df[c].isna().mean() < 0.20]
if mid_missing_num:
    imputer = IterativeImputer(random_state=42, max_iter=10, verbose=0)
    before_na = df[mid_missing_num].isna().sum().sum()
    imputed_values = imputer.fit_transform(df[mid_missing_num])
    df[mid_missing_num] = imputed_values
    print(f"MICE補完: {mid_missing_num} ({before_na}件補完)")

# MNAR → 欠損フラグ変数を追加
high_missing = [c for c in df.columns if 0.20 <= df[c].isna().mean() < 0.50]
for col in high_missing:
    df[f'{col}_is_missing'] = df[col].isna().astype(int)
    print(f"欠損フラグ追加: {col}_is_missing")
```

### Step 3: 外れ値処理【GL-6】

```python
# ─────────────────────────────────────────
# 客観的基準のみ使用（恣意的除外禁止）
# ─────────────────────────────────────────

def detect_outliers_iqr(series, k=1.5):
    q1, q3 = series.quantile([0.25, 0.75])
    iqr = q3 - q1
    return (series < q1 - k * iqr) | (series > q3 + k * iqr)

def detect_outliers_zscore(series, threshold=3):
    z_scores = (series - series.mean()) / series.std()
    return z_scores.abs() > threshold

print("\n=== 外れ値検出（ドメイン確認必須）===")
for col in df.select_dtypes(include='number').columns:
    iqr_mask = detect_outliers_iqr(df[col].dropna())
    z_mask = detect_outliers_zscore(df[col].dropna())
    consensus = iqr_mask & z_mask
    if consensus.sum() > 0:
        print(f"  {col}: IQR法={iqr_mask.sum()}件, 3σ法={z_mask.sum()}件, 両方={consensus.sum()}件")

print("\n→ 外れ値の処理方針は必ずドメイン専門家と協議すること【GL-6】")
print("   判断基準: 1)入力エラー → 削除/修正  2)真の外れ値 → Winsorize  3)稀少な正常値 → 保持")

# Winsorization（上下○%ile でクリップ）
# df[col] = df[col].clip(lower=df[col].quantile(0.01), upper=df[col].quantile(0.99))
```

---

## 5. 日付・時刻の標準化（シニアDE必須）

```python
# ─────────────────────────────────────────
# 5-1. 様々な日付フォーマットの統一
# ─────────────────────────────────────────
date_formats = [
    '%Y-%m-%d', '%Y/%m/%d', '%Y%m%d',
    '%d/%m/%Y', '%m/%d/%Y',
    '%Y年%m月%d日', '%Y年%m月%d日 %H時%M分',
    '%Y-%m-%dT%H:%M:%S', '%Y-%m-%d %H:%M:%S'
]

def parse_date_flexible(val):
    if pd.isna(val):
        return pd.NaT
    val_str = str(val).strip()
    for fmt in date_formats:
        try:
            return pd.to_datetime(val_str, format=fmt)
        except (ValueError, TypeError):
            continue
    return pd.to_datetime(val_str, errors='coerce', infer_datetime_format=True)

date_cols = df.select_dtypes(include=['datetime64']).columns.tolist()
# object型の日付列を検出
for col in df.select_dtypes(include='object').columns:
    sample_converted = pd.to_datetime(df[col].dropna().head(50), errors='coerce', infer_datetime_format=True)
    if sample_converted.notna().mean() > 0.8:
        date_cols.append(col)

for col in date_cols:
    df[col] = df[col].apply(parse_date_flexible)
    print(f"日付標準化: {col} → datetime64")
    print(f"  期間: {df[col].min()} 〜 {df[col].max()}")
    print(f"  未来日付: {(df[col] > pd.Timestamp.now()).sum()}件 → 要確認")
    print(f"  異常に古い日付（1900年以前）: {(df[col] < pd.Timestamp('1900-01-01')).sum()}件 → 要確認")
```

---

## 6. エンティティ解決・名寄せ（シニアDE必須）

```python
# ─────────────────────────────────────────
# 6-1. 同一エンティティの名寄せ（マッピング辞書方式）
# ─────────────────────────────────────────
# 例: 企業名の名寄せ（事前にドメイン専門家と作成）
# entity_mapping = {
#     'サンゲツ': 'サンゲツ株式会社',
#     '（株）サンゲツ': 'サンゲツ株式会社',
#     'SANGETSU': 'サンゲツ株式会社',
# }
# df['company_name'] = df['company_name'].replace(entity_mapping)

# ─────────────────────────────────────────
# 6-2. 重複レコードの統合（ファジーマッチ）
# ─────────────────────────────────────────
try:
    from rapidfuzz import fuzz

    def deduplicate_fuzzy(df, col, threshold=90):
        """
        文字列列に基づいてファジーマッチで重複を検出する
        Returns: 重複グループのDataFrame
        """
        unique_vals = df[col].dropna().unique()
        groups = {}
        processed = set()

        for val in unique_vals:
            if val in processed:
                continue
            group = [val]
            for other in unique_vals:
                if other != val and other not in processed:
                    if fuzz.ratio(str(val), str(other)) >= threshold:
                        group.append(other)
                        processed.add(other)
            if len(group) > 1:
                canonical = max(group, key=len)  # 最長の文字列を代表値
                groups[canonical] = group
            processed.add(val)
        return groups

    # 適用例（company_name 列で名寄せ）
    # if 'company_name' in df.columns:
    #     dupes = deduplicate_fuzzy(df, 'company_name', threshold=90)
    #     print(f"名寄せ候補: {len(dupes)}グループ")
    #     for canonical, variants in list(dupes.items())[:5]:
    #         print(f"  代表値「{canonical}」: {variants}")

except ImportError:
    print("rapidfuzz 未インストール。名寄せ機能には pip install rapidfuzz が必要です。")
```

---

## 7. Tidy データへの変換（データ構造化）

```python
# ─────────────────────────────────────────
# Tidy Data の3原則:
# 1. 各変数が1列
# 2. 各観測値が1行
# 3. 各観測単位が1テーブル
# ─────────────────────────────────────────

# Wide → Long 変換
# 例: 月別売上列（2024_01, 2024_02...）が横持ちの場合
# df_long = df.melt(
#     id_vars=['product_id', 'store_id'],  # キー列
#     value_vars=[c for c in df.columns if c.startswith('2024')],  # 値列
#     var_name='month',
#     value_name='sales'
# )
# print(f"Wide→Long変換: {len(df):,}行 × {len(df.columns)}列 → {len(df_long):,}行 × {len(df_long.columns)}列")

# Long → Wide 変換
# df_wide = df_long.pivot_table(
#     index=['product_id', 'store_id'],
#     columns='month',
#     values='sales',
#     aggfunc='sum'
# ).reset_index()

# ─────────────────────────────────────────
# 分析単位（Grain）の確認
# ─────────────────────────────────────────
print("\n=== Grain（分析単位）確認 ===")
print("現在のGrain（推定）: 1行 = ?")
for col in df.columns:
    if df[col].nunique() == len(df):
        print(f"  主キー候補: {col} → 1行 = 1{col}")
print("→ 分析目的に合ったGrainになっているか確認する")
```

---

## 8. メモリ最適化（シニアDE必須）

```python
# ─────────────────────────────────────────
# データ型のダウンキャスト（大規模データ向け）
# ─────────────────────────────────────────
def optimize_memory(df):
    """数値型をダウンキャストしてメモリ使用量を削減する"""
    before_mem = df.memory_usage(deep=True).sum() / 1024 ** 2

    for col in df.select_dtypes(include=['int']).columns:
        df[col] = pd.to_numeric(df[col], downcast='integer')

    for col in df.select_dtypes(include=['float']).columns:
        df[col] = pd.to_numeric(df[col], downcast='float')

    # カーディナリティが低い object は category に
    for col in df.select_dtypes(include='object').columns:
        if df[col].nunique() / len(df) < 0.5:
            df[col] = df[col].astype('category')

    after_mem = df.memory_usage(deep=True).sum() / 1024 ** 2
    print(f"メモリ最適化: {before_mem:.1f} MB → {after_mem:.1f} MB ({(1-after_mem/before_mem)*100:.0f}%削減)")
    return df

df = optimize_memory(df)
```

---

## 9. 妥当性チェック（値域・ビジネスルール）

```python
# ─────────────────────────────────────────
# 9-1. 定義上ありえない値の検出
# ─────────────────────────────────────────
print("\n=== 妥当性チェック ===")

validity_rules = {}
# ドメイン知識に基づいてルールを定義する（例）
# validity_rules = {
#     '年齢': (0, 120),
#     '価格': (0, None),      # 負の値は不正
#     '数量': (0, None),
#     '在庫数': (0, 1_000_000),
# }

for col, (min_val, max_val) in validity_rules.items():
    if col not in df.columns:
        continue
    if min_val is not None:
        n_invalid = (df[col] < min_val).sum()
        if n_invalid > 0:
            print(f"⚠️ {col} < {min_val}: {n_invalid}件")
    if max_val is not None:
        n_invalid = (df[col] > max_val).sum()
        if n_invalid > 0:
            print(f"⚠️ {col} > {max_val}: {n_invalid}件")

# ─────────────────────────────────────────
# 9-2. 一貫性チェック（列間の論理的整合）
# ─────────────────────────────────────────
# 例: 開始日 ≤ 終了日
# if 'start_date' in df.columns and 'end_date' in df.columns:
#     invalid = df['start_date'] > df['end_date']
#     print(f"開始日 > 終了日: {invalid.sum()}件 → 要確認")
```

---

## 10. クレンジング前後の比較と記録【GL-7】

```python
# ─────────────────────────────────────────
# 10-1. 統計量の変化確認
# ─────────────────────────────────────────
print("\n=== クレンジング前後の比較 ===")
print(pd.concat([
    df_raw.describe().add_suffix('_before'),
    df.describe().add_suffix('_after')
], axis=1).to_string())

# ─────────────────────────────────────────
# 10-2. クレンジングログの出力
# ─────────────────────────────────────────
log_df = pd.DataFrame(cleaning_log)
if not log_df.empty:
    print("\n=== クレンジング操作ログ ===")
    print(log_df.to_string(index=False))

log_step("クレンジング完了", len(df_raw), len(df), "全ステップ合計",
         f"元データの{len(df)/len(df_raw):.1%}を保持")

# ─────────────────────────────────────────
# 10-3. クレンジング済みデータの保存
# ─────────────────────────────────────────
df.to_csv('data/cleaned_data.csv', index=False, encoding='utf-8-sig')
print(f"\n保存完了: data/cleaned_data.csv ({len(df):,}件 × {len(df.columns)}列)")
```

---

## 11. データリネージの記録（シニアDE必須）

```python
# ─────────────────────────────────────────
# すべての変換操作を記録する（監査・再現性のため）
# ─────────────────────────────────────────
lineage = f"""
## データリネージ

| ステップ | 入力 | 操作 | 出力 | 担当者 | 日時 |
|---------|------|------|------|--------|------|
| 1 | {df_raw.shape} | 重複削除 | {df.shape} | [担当者] | {pd.Timestamp.now().strftime('%Y-%m-%d')} |
| 2 | — | 欠損値補完 | — | [担当者] | — |
| 3 | — | 型変換 | — | [担当者] | — |
| 4 | — | 日本語正規化 | — | [担当者] | — |

## 不確実性・判断保留事項
- [不確実な判断: 内容] → [確認方法]
"""
print(lineage)
```

---

## 12. クレンジング結果サマリーと Next Step

```
【クレンジングサマリー】
 - 処理前: ○○件 × ○○列
 - 処理後: ○○件 × ○○列（元の○○%保持）
 - 重複削除: ○○件
 - 型変換: ○○列（数値/日付）
 - 日本語正規化: ○○列（表記揺れ統合: ○○件）
 - 欠損処理: ○○列（中央値/MICE/フラグ）
 - 外れ値確認: ○○件（処理方針: ○○）
 - メモリ削減: ○○%
 - 新規追加列: ○○（欠損フラグ、正規化列など）

【注意事項・不確実性】
 - 不確実性: [内容]（確度: 高/中/低）
 - 未解決: [ドメイン専門家確認待ち事項]

【Next Step】
→ /data-analysis:data-feature で特徴量エンジニアリングに進む
```

---

クレンジング結果を `analysis_context.md` の「分析経過メモ」に追記し、`data/docs/02_cleaning_report.md` にレポートを保存すること。

$ARGUMENTS

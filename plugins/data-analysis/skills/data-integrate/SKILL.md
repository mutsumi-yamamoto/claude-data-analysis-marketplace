---
name: data-integrate
description: 複数データソースの統合・整合性検証・データリネージ記録を行います。「データを結合したい」「複数テーブルを統合したい」「データの整合性を確認したい」と言われたら使用してください。
---

# データ統合・蓄積スキル

## 前提: analysis_context.md を必ず読み込んでから実行すること【最重要】

---

## 1. 分析単位（Grain）の定義

**最重要**: 1レコード = 何か を明確にする

例:
- 1レコード = 1顧客の1日の行動
- 1レコード = 1注文
- 1レコード = 1商品 × 1週間の販売実績

分析単位が不明確なまま結合することを禁止する。

## 2. 結合計画の設計

```python
# 結合前のレコード数を必ず確認する
print(f"テーブルA件数: {len(df_a):,}")
print(f"テーブルB件数: {len(df_b):,}")

# 結合キーのNULL確認
print(f"テーブルA キーNULL率: {df_a['key'].isna().mean():.2%}")
print(f"テーブルB キーNULL率: {df_b['key'].isna().mean():.2%}")

# 結合キーのカーディナリティ確認
print(f"テーブルA ユニークキー数: {df_a['key'].nunique():,}")
print(f"テーブルB ユニークキー数: {df_b['key'].nunique():,}")
```

## 3. Fan-out（重複）問題の防止

1:N結合では意図しない重複が発生しないか確認する:

```python
# 結合前後のレコード数を比較
df_merged = df_a.merge(df_b, on='key', how='left')

print(f"結合前: {len(df_a):,}")
print(f"結合後: {len(df_merged):,}")
print(f"増加率: {len(df_merged)/len(df_a):.2f}倍")

# 1倍を超える場合はFan-outが発生している
if len(df_merged) > len(df_a):
    print("⚠️ Fan-out検知: 結合キーの一意性を確認してください")
    # 重複キーの確認
    dup_keys = df_b[df_b.duplicated(subset='key', keep=False)]['key'].unique()
    print(f"重複キー数: {len(dup_keys)}")
```

## 4. 時間軸の整合性確認

時系列データの結合では時間的整合性を必ず確認する:
- 結合キーに加えて時間軸の対応関係を確認する
- 未来の情報が混入していないかチェックする（ターゲットリーケージ防止）【GL-10】

```python
# 時間軸の確認
print(f"テーブルA 期間: {df_a['date'].min()} 〜 {df_a['date'].max()}")
print(f"テーブルB 期間: {df_b['date'].min()} 〜 {df_b['date'].max()}")
```

## 5. 整合性検証チェックリスト

結合後のテーブルに対して以下を確認する:

```python
# 結合後の確認
df = df_merged.copy()

print("=== 整合性チェック ===")
print(f"総レコード数: {len(df):,}")
print(f"総カラム数: {len(df.columns)}")
print(f"\n欠損値チェック:")
missing = df.isnull().sum()
print(missing[missing > 0])

print(f"\n重複レコードチェック:")
print(f"重複行数: {df.duplicated().sum():,}")

print(f"\n主キーの一意性:")
print(f"ユニーク件数: {df['primary_key'].nunique():,}")
```

## 6. データリネージの記録

以下をドキュメントとして `data/docs/` に記録する:

```markdown
## データリネージ

| 結果テーブル | ソーステーブルA | ソーステーブルB | 結合キー | 結合種別 | 作成日 |
|-------------|----------------|----------------|---------|---------|--------|
| analysis_table | ○○.csv | ○○.csv | customer_id | LEFT JOIN | YYYY-MM-DD |
```

## 7. 分析テーブルの保存

```python
# 分析テーブルを保存
df.to_csv('data/analysis_table.csv', index=False, encoding='utf-8')
print(f"保存完了: {len(df):,} 件")
```

## 8. Next Step の提示

```
統合テーブルが完成したら → /data-analysis:data-explore（データ探索）
```

---

## 📝 実行ログの記録（必須）

`analysis_context.md` の「データ統合情報」セクションを更新し、「12. 実行ログ」末尾に以下のテンプレートを埋めて追記すること。

```
### YYYY-MM-DD HH:MM | data-integrate
| 項目 | 内容 |
|------|------|
| ステータス | 完了 / 一部完了 / 中断 |
| 実施内容 | [結合処理・整合性検証・Fan-out確認] |
| 結合ソース数 | [件数] |
| 結合前件数 | [各テーブルの件数] |
| 結合後件数 | [件数（Fan-out有無を明記）] |
| 結合キー | [結合に使用したキー列名] |
| 整合性問題 | [不一致・欠損・重複等の件数と対処] |
| 問題・懸念 | [解決できなかった問題] |
| 申し送り | [EDAで特に確認すべき統合起因の問題] |
| 生成ファイル | `data/analysis_table.csv` |
```

---

## ✅ 完了後: 次の推奨アクション（必須）

上記の実行が完了したら、**必ず**以下をユーザーに提示すること。

**✅ data-integrate が完了しました**
📁 生成ファイル: `data/analysis_table.csv`
📋 `analysis_context.md` の「データ統合情報」を更新しました

**▶ 次の推奨ステップ（標準フロー）:**
```
/data-analysis:data-explore
```
統合済みデータの全体像・品質・分布・相関をEDAで把握します

**現在の推奨フロー:**
```
data-context → data-define → data-collect → data-integrate ✅ → data-explore → data-clean → data-feature → data-model → data-interpret
```

$ARGUMENTS

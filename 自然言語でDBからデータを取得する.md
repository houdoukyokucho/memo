# 自然言語でDBからデータを取得する

調査中。。。

## 処理全体フロー

| アクション | 担当 | インプット | アウトプット |
|---|---|---|---|
| 自然言語で質問 | ユーザー | - | 自然言語 |
| 自然言語を構造化クエリに変換 | LLM（Gemini） | 自然言語 | 構造化クエリ（JSON） |
| 構造化クエリからDBクエリを組み立て | コード | 構造化クエリ（JSON） | DBクエリ（SQL + ベクトル検索条件） |
| DBから検索 | DB (psql) | DBクエリ | 検索結果（行データ） |
| JSONに変換 | コード | 検索結果（行データ） | APIレスポンスJSON |

## 自然言語で質問

ユーザーが、希望条件を自然言語で入力する。
条件は曖昧でもよく、自然な日本語のままでよい。

【アウトプット例】
```
蔵前付近で、2LDK以上、ペット可能で、庭付きの物件を探したい
```

## 自然言語を構造化クエリに変換

自然言語の質問を解析し、意味検索に使う条件と、数値・属性条件として使うものを分離する。この処理は LLM（Gemini）が担当する。

【アウトプット例】
```json
{
  "semantic_query": "庭付き 住居",
  "filters": {
    "min_layout": "2LDK",
    "pet_allowed": true,
    "area": "蔵前"
  }
}
```

## 構造化クエリからDBクエリを組み立て

構造化クエリをもとに、
データベースが実行できるDBクエリ（SQL + ベクトル検索条件）を組み立てる。
この処理は決定論的なアプリケーションコードで行う。

【アウトプット例】
```
SELECT
  id,
  name,
  description,
  1 - (embedding <=> :query_embedding) AS score
FROM properties
WHERE layout_rank >= :min_layout_rank
  AND pet_allowed = true
  AND area = '蔵前'
ORDER BY embedding <=> :query_embedding
LIMIT 10;
```

## DBから検索

組み立てられたDBクエリを実行し、条件に合致し、かつ意味的に近い物件を取得する。この時点ではデータは行データとして扱われる。

【アウトプット例】
### APIレスポンス（JSON相当の内容）

| id | name | score | description |
|---|---|---:|---|
| property-001 | 蔵前ガーデンレジデンス | 0.91 | 専用庭付きの2LDK。ペット飼育可。 |
| property-014 | リバーサイド蔵前 | 0.86 | 1階庭付き住戸。小型犬可。 |


## JSONに変換
DBから取得した行データを、APIレスポンスとして返すJSON形式に変換する。自然言語での要約や説明は行わない。

【アウトプット例】
```json
{
  "results": [
    {
      "id": "property-001",
      "name": "蔵前ガーデンレジデンス",
      "score": 0.91,
      "description": "専用庭付きの2LDK。ペット飼育可。"
    },
    {
      "id": "property-014",
      "name": "リバーサイド蔵前",
      "score": 0.86,
      "description": "1階庭付き住戸。小型犬可。"
    }
  ]
}
```

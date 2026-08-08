# OpenAPI Generator DTO 重複生成問題 解決ガイド

これまでのセッションにおける `openapi-generator` の **DTO（モデルクラス）重複問題（末尾に番号がつく原因）** と、**Spring Boot (Kotlin) での `openapi-normalizer` の設定方法** についてのまとめです。

---

## 1. DTOの末尾に番号（`1`, `2`など）がつく原因と解決策

`openapi-generator` が「構造は似ているが、別のクラスとして扱わなければならない」と判断したときに、衝突を避けるために自動的に番号が振られます。主な原因は以下の4点です。

### 原因①：インラインオブジェクト（共通化漏れ）
複数のエンドポイントの `requestBody` や `responses` で、まったく同じ構造のオブジェクトを個別に直接記述しているケース。

* **NG（重複の原因）**:
  ```yaml
  /users:
    post:
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                name: { type: string }
  ```
* **OK（共通化による解決）**:
  必ず `components/schemas` に定義し、`$ref` で参照します。
  ```yaml
  /users:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserRequest'
  ```

### 原因②：`$ref` の横に `description` などを書いている (OpenAPI 3.1 バグ)
OpenAPI 3.1 では `$ref` と同階層の属性（シブリングアノテーション）が許容されていますが、ジェネレーターがこれを「別物のカスタムスキーマ」と誤認して複製バグを起こすことがあります。

* **NG（重複の原因）**:
  ```yaml
  properties:
    author:
      $ref: '#/components/schemas/User'
      description: "投稿者情報" # $refの横にあるとUser1が作られる
  ```
* **OK（`allOf` による階層分離）**:
  ```yaml
  properties:
    author:
      description: "投稿者情報"
      allOf:
        - $ref: '#/components/schemas/User'
  ```

### 原因③：インライン配列（`items: { type: object }`）
`type: array` の中身（`items`）を、`$ref` を使わずにインラインで直接オブジェクト定義した場合。

* **NG（重複の原因）**:
  ```yaml
  properties:
    users:
      type: array
      items:
        type: object
        properties:
          id: { type: string }
  ```
* **OK（スキーマ参照化）**:
  ```yaml
  properties:
    users:
      type: array
      items:
        $ref: '#/components/schemas/UserSummary'
  ```

### 原因④：同一YAML内での `title` タグの重複
`components/schemas` 以下などで `title: User` という記述が複数ある場合、ジェネレーターがクラス決定時に `title` を優先し、`User1` などを生成することがあります。

* **解決策**:
  YAML全体から重複する `title: XXX` を検索し、削除するか一意な名前に変更します（基本的にはスキーマのキー名がクラス名になるため `title` は不要です）。

---

## 2. Spring Boot (Kotlin) での `openapi-normalizer` 設定

定義ファイルの揺れや複雑な構造（`allOf` など）をクレンジングして重複を防ぐために、Gradle（Kotlin DSL）環境で **`openapi-normalizer`** を有効化します。

### build.gradle.kts への設定例

`openApiGenerate` タスク内で `openApiNormalizer.set(mapOf(...))` を使用してルールを指定します。

```kotlin
openApiGenerate {
    generatorName.set("kotlin-spring")
    
    // --- openapi-normalizer の設定 ---
    openApiNormalizer.set(
        mapOf(
            "REFACTOR_ALLOF_WITH_PROPERTIES_ONLY" to "true", // allOf周辺の重複整理
            "REF_AS_PARENT_IN_ALLOF" to "true",             // 親クラスの適切な認識
            "SIMPLIFY_ONEOF_ANYOF" to "true"                // 複雑な定義の単純化
        )
    )
}
```

### 推奨ルールの効果
1. **`REFACTOR_ALLOF_WITH_PROPERTIES_ONLY`**: `$ref` なしのプロパティを持つ `allOf` をインライン化し、不要な中間オブジェクト生成を抑制。
2. **`REF_AS_PARENT_IN_ALLOF`**: `$ref` を使用した親クラスを適切に認識させ、子クラスでの重複定義を防ぐ。

---

## 3. その他の回避策

* **外部DTOのインポート（生成のスキップ）**:
  すでに別プロジェクト等で定義済みのクラスがある場合は、`--import-mappings` を使って既存クラスを使い回すように強制できます。
* **ジェネレーターのバージョン変更**:
  特定の言語・バージョンにおけるパース処理のバグである可能性があるため、最新の安定版へアップデートするか、過去の安定版へダウングレードします。

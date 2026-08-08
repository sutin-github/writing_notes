# openapi-generator DTO 重複生成問題に関するやりとり

## 質問1: openapi generator dto 重複
`openapi-generator` で DTO（モデルクラス）が `User`, `User2` のように**重複して自動生成される問題**は、多くの場合 **OpenAPI（Swagger）定義ファイルの書き方**や**ジェネレーターの仕様（バグ含む）**が原因です。

原因に応じた具体的な解決策を以下にまとめました。

### 1. インラインオブジェクト（共通化漏れ）を解決する
最も多い原因は、複数のエンドポイントの `requestBody` や `responses` で**共通の構造を共通のコンポーネント（`$ref`）として定義せず、インラインで直接記述している**ケースです。ジェネレーターは構造が同じでも定義場所が異なると別物とみなし、末尾に数字をつけた重複クラスを作ります。

* **NGな例（重複の原因）**:
  ```yaml
  /users:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object  # インライン定義
                properties:
                  id: { type: string }
  ```
* **OKな例（共通化による解決）**:
  必ず `components/schemas` に定義し、`$ref` で参照してください。
  ```yaml
  /users:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserDto' # 参照化
  ```

### 2. OpenAPI 3.1 系の `$ref` の仕様バグを回避する
OpenAPI 3.1 を使用しており、**`$ref` と同じ階層に `description` などの他のプロパティ（シブリングアノテーション）を書いている場合**、重複モデル（`User` と `User1` など）が生成されるバグが報告されています。

* **NGな例（重複の原因）**:
  ```yaml
  properties:
    author:
      $ref: '#/components/schemas/UserDto'
      description: "投稿したユーザーの情報" # $refの横にdescriptionを書くと重複する
  ```
* **解決策**: 以下のように `allOf` を噛ませて回避します。
  ```yaml
  properties:
    author:
      description: "投稿したユーザーの情報"
      allOf:
        - $ref: '#/components/schemas/UserDto'
  ```

### 3. `openapi-normalizer` 機能を有効化する
`openapi-generator` には、定義ファイルの揺れや複雑な構造を自動的にクレンジングして重複を防ぐ **`openapi-normalizer`** という機能が備わっています。

* **CLI コマンド例**:
  ```bash
  openapi-generator generate -g java -i openapi.yaml -o ./out \
    --openapi-normalizer REFACTOR_ALLOF_WITH_PROPERTIES_ONLY=true,SIMPLIFY_ONEOF_ANYOF=true
  ```

### 4. 外部の既存 DTO をインポートして再利用する（生成のスキップ）
* **設定例（CLI）**:
  ```bash
  openapi-generator generate -g java -i openapi.yaml -o ./out \
    --import-mappings UserDto=com.example.shared.dto.UserDto
  ```

### 5. ジェネレーターのバージョンを変更する
特定の言語において、アレイ型（配列）のネストや継承（`allOf`）のパース処理にバグがあり、一時的に重複クラスが生成されるバグが定期的に発生しています。最新の安定版（Stable）にするか、過去のパッチバージョンへのダウングレードを検討します。

---

## 質問2: springboot kotlin openapi-normalizer
Spring Boot (Kotlin) の環境において、`openapi-generator` Gradle プラグインを利用して **DTO の重複生成（`User`, `User_allOf` など）** を解決するために `openapi-normalizer` を設定する方法を解説します。

### build.gradle.kts への設定方法
`openApiGenerate` タスク内の **`openApiNormalizer.set(mapOf(...))`** を使用して指定します。

```kotlin
openApiGenerate {
    openApiNormalizer.set(
        mapOf(
            "REFACTOR_ALLOF_WITH_PROPERTIES_ONLY" to "true",
            "REF_AS_PARENT_IN_ALLOF" to "true"
        )
    )
}
```

### DTO 重複の解消に強力な 2 つの推奨ルール
1. **`REFACTOR_ALLOF_WITH_PROPERTIES_ONLY` (`true`)**: `$ref` なしのプロパティを持つ `allOf` をインライン化し、不要な中間オブジェクト生成を抑制します。
2. **`REF_AS_PARENT_IN_ALLOF` (`true`)**: `$ref` を使用した親クラスを適切に認識し、子クラスごとのプロパティ再定義（重複）を防ぎます。

---

## 質問3: DTO の重複生成 末尾に番号がつく
`openapi-generator` で生成された DTO の**末尾に `1`, `2` などの番号がつく現象**は、ジェネレーターが「これらは構造が似ているけれど、別のクラスとして扱わなければならない」と判断したときに発生します。

### 原因1：インライン定義（`type: object` の直接記述）
別々の場所でまったく同じ構造のオブジェクトを個別にインライン定義すると、衝突を避けるために番号が振られます。
* **対策**: 必ず `components/schemas` に記述し、**すべて `$ref` で参照**するように統一します。

### 原因2：`$ref` の横に `description` などを書いている (OpenAPI 3.1 バグ)
* **対策**: `$ref` の横にある `description` などを削除するか、**`allOf` を噛ませて階層を分ける**ことで回避します。

### 原因3：インライン配列（`items: { type: object }`）
`type: array` の中身（`items`）を `$ref` を使わずにインラインで直接オブジェクト定義した場合も、番号が生成されます。
* **対策**: 配列の中身も必ずコンポーネント化して `$ref` で指定します。

### 原因4：同一YAML内で `title` タグの重複がある
ジェネレーターがクラス名を決定する際に `title` を優先し、結果として `User1`, `User2` を生成することがあります。
* **対策**: 不要であれば各スキーマ内の `title` プロパティを削除します。
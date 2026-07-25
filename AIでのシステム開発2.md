# AIでのシステム開発（まとめ）

システム開発の実装をAIで自動化できる・できない箇所とそれぞれの方法と具体的な手順を各レイヤー・工程ごとに分かり易く整理して教えてください。

- システム種類：Web API
- プログラミング言語：Kotlin 2.x
- 環境：Spring Boot 4, OpenAPI, Spring Data JPA, Blaze-Persistence, MapStruct
- 条件：クリーンアーキテクチャにて実装する
- 実装上の注意：Blaze-Persistenceを用いた複雑なクエリが必要
- AI：GitHub Copilot（Claude Opus 4.6 を活用）

補足条件：
- DB設計にてテーブル定義のDDLがあるため、DBマイグレーションは行わない
- OpenAPI の yaml は設計にて作成済み。openapi-generator によりコード自動生成済み
- Kotlin の @Entity クラスは IntelliJ の JPA Buddy を使用して生成する
- @Entity はドメインモデルと完全に分離する
- クリーンアーキテクチャの層は以下：
  - コントローラー: controller
  - ドメインモデル: domain
  - データアクセス: entity
  - サービス: service
  - リポジトリ: repository
- 各層の変換には MapStruct を使用

---

## 要約（結論）
ボイラープレート（定型）コードの生成や単純なマッピング、基本的なロジックの実装は 90% 以上自動化可能です。ただし、Blaze-Persistence を用いた高度でドメイン固有の複雑クエリについては、人間による設計・検証・修正が不可欠です。

---

## 全体アーキテクチャと AI の自動化カバー率

各レイヤー・工程における AI の自動化可否のサマリーです。

| レイヤー / 工程                           | 自動化の可否                     | AI の役割 / 手法 |
|-------------------------------------------|----------------------------------|------------------|
| 0. 前準備 (OpenAPI / Entity)              | 完全自動化                       | openapi-generator と JPA Buddy で 100% 自動生成 |
| 1. プレゼンテーション層 (Controller)      | 高確率で自動化可                 | 自動生成された Interface の実装クラスを Copilot で生成 |
| 2. ドメイン層 (Domain Model)              | 一部自動化可（人間が主導）       | 値オブジェクトや不変オブジェクトは自動生成可。ビジネス設計は人がレビュー |
| 3. データアクセス層 (Repository / Entity) | 一部自動化可（要精密プロンプト） | Blaze-Persistence の複雑なクエリは AI 単体での精度が足りないため手動修正必須 |
| 4. マッピング層 (MapStruct)               | ほぼ完全自動化                   | @Mapper インターフェースの生成、カスタムマッパー補助 |

---

## 各レイヤー・工程ごとの詳細手順

### 0. 設計・前準備工程（自動生成）
この工程は指定ツールでほぼ 100% 自動化できます。

- できること
  - OpenAPI yaml → Controller の Interface、Request/Response DTO を openapi-generator で自動生成
  - DDL → @Entity クラス（@Id、リレーションなど）を IntelliJ JPA Buddy で自動生成
- できないこと
  - 特になし（ただし生成物の命名規約・パッケージ構成は要確認）

---

### 1. プレゼンテーション層（Controller）
openapi-generator が生成した Interface を実装する Controller クラスを作成します。

- 自動化できる箇所
  - Interface を満たすメソッドのオーバーライド
  - 引数 DTO → ドメインモデル への MapStruct 呼び出し
  - Service 層の呼び出し、レスポンス DTO への変換
- 自動化できない箇所
  - HTTP ステータスコードの微調整やポリシー設計（例外ハンドリングの全体方針など）
- 具体的な AI を使った手順（例）
  1. 空の Controller クラスを用意する
  2. Copilot に以下を指示して生成
     - クラス注釈：@RestController
     - implements XxxApi（openapi-generator により生成された Interface）
     - コンストラクタインジェクションで XxxService と XxxMapper を注入
     - 各エンドポイントで DTO → ドメイン → Service → DTO 変換の流れを実装
  3. Kotlin 2.x の記法に沿うこと

---

### 2. ドメイン層（Domain Model）
クリーンアーキテクチャの核心。外部に依存しない純粋な Kotlin コードで記述します。

- 自動化できる箇所
  - 値オブジェクト（Value Object）や不変な data class の生成
  - 単純なバリデーションロジック、ユーティリティ関数
- 自動化できない箇所
  - ビジネスロジックの適切なカプセル化や責務設計（AI はロジックを間違った層に置きがち）
- 具体的な AI を使った手順（例）
  1. 要件定義や仕様を Copilot（Claude 4.6 Opus）に読み込ませる
  2. プロンプト例：
     - 「以下の仕様を満たす純粋な Kotlin のドメインモデル（data class）を作成してください。Spring や JPA の依存は含めないこと。状態変更はメソッドを通じて新しいインスタンスを返す不変オブジェクトにしてください。」
  3. 生成後、人間が責務や不変性をレビューする

---

### 3. データアクセス層（Repository / Entity / Blaze-Persistence）
ドメインリポジトリのインターフェースを Blaze-Persistence と Spring Data JPA で実装する最難関工程です。

- 自動化できる箇所
  - 基本 CRUD（Spring Data JPA のリポジトリ定義）
  - Blaze-Persistence, CriteriaBuilderFactory などのボイラープレート DI コード
- 自動化できない箇所
  - Blaze-Persistence を用いた複雑クエリ（CTE、Window 関数、複雑な JOIN）の正確な生成
  - 理由：API の細かな仕様、ドメインと Entity の完全分離ルール、最適なクエリ設計はドメイン固有であり AI 単体では誤りを含む可能性が高い
- 部分自動化の具体手順（AI + 人手）
  1. Entity、ドメインモデル、SQL（やりたいこと）の3点を Copilot に提示
  2. プロンプト例：
     - 「以下の Entity とドメインモデルがあります。CriteriaBuilderFactory と EntityViewManager を使って、[やりたい集計・結合] を行うリポジトリ実装を作ってください。戻り値は Entity ではなくドメインモデルに変換して返すこと。」
  3. 生成されたコードは IntelliJ でコンパイルして生じるエラーを手動で修正（特に Blaze-Persistence の型・API 周り）

---

### 4. マッピング層（MapStruct）
Entity ⇄ Domain Model ⇄ DTO の相互変換を MapStruct で定義します。

- 自動化できる箇所
  - @Mapper インターフェースの作成
  - フィールド名が一致する単純なマッピングは自動生成可能
- 自動化できない箇所
  - 構造が大きく異なる変換（フラットな Entity をネストされた値オブジェクトへ変換する等）の複雑なマッピング設計
- 具体的な AI を使った手順
  1. 変換元（XxxEntity）と変換先（XxxDomain）を開く
  2. Copilot に MapStruct のマッパー作成を依頼
     - 条件：componentModel = "spring"
     - 必要なら @Mapping 指定でフィールド名の差異を明記（例：hoge_id → hogeId）

---

## AI 自動化を成功させるための注意点（Kotlin 2.x × Spring Boot 4）

1. コンテキストの共有  
   - Copilot チャットを使う際は、必ず関係する「OpenAPI yaml」「JPA Buddy が作った Entity」「ドメインモデル」などのファイルをすべて開いた状態でプロンプトを投げる。これにより AI の生成精度が向上する。

2. フレームワーク／言語バージョン差分への注意  
   - AI は古い Spring / Kotlin の記法を混ぜることがあるため、生成コードは常にコンパイルしてエラーを確認し、必要に応じてバージョンに合わせて修正する。

---

まずは最初のステップとして、すでに手元にある OpenAPI の yaml ファイル、または JPA Buddy で生成した Entity クラスを 1 つ教えてください。それらをベースに、Claude 4.6 Opus にそのまま入力できる「MapStruct マッパー」や「Controller クラス」を生成するための具体的なプロンプトを作成します。

AI システム開発 実装 自動化できる・できない箇所とそれぞれの方法と具体的な手順を各レイヤー・工程ごとに分かり易く整理して教えて。 システム種類：WebAPI。 プログラミング言語：kotlin 2.x。 環境：spring boot4, openApi, Spring Data JPA Blaze-Persistence、MapStruct。 条件：クリーンアーキテクチャにて実装する。 実装：Blaze-Persistenceを用いた複雑なクエリが必要になる。 AI：GitHub CopilotからClaude Opus 4.6を使用する。 DB設計にてテーブル定義のDDLがあるので、DBマイグレーションはしない。 OpenApiのyamlは設計にて作成済で、OpenAPIのyamlからコードを自動生成するopenapi-generatorも導入済。 Kotlinの@Entityクラスの生成は、IntelliJのJPA Buddyを使用する。 @Entityはドメインモデルと完全に分ける。 クリーンアーキテクチャは、 ・コントローラー:controller ・ドメインモデル:domain ・データアクセス:entity ・サービス:Service ・リポジトリ：Repository。 各層の変換にはMapStructを使用。

### AIシステム開発の実装工程において、ご提示の技術スタック（Kotlin 2.x, Spring Boot 4, OpenAPI, Blaze-Persistence, MapStruct, クリーンアーキテクチャ）で「自動化できる（コード生成・ボイラープレート）箇所」と「自動化できない（人間による設計・プロンプト誘導・ロジック構築が必要な）箇所」を整理しました。結論として、レイヤー間の繋ぎ（I/Oやマッパー）は「自動化可能」ですが、Blaze-Persistenceを用いた複雑なクエリや、ドメイン層のコアロジックは「自動化不可能（人の手によるプロンプト制御が必要）」となります。
------------------------------
## 全体マップと自動化比率

| レイヤー / 工程 | 自動化の可否 | 主要ツール / AIの役割 |
|---|---|---|
| 1. ルーティング層 (Controller) | 完全自動化可能 | OpenAPI Generator |
| 2. データモデル層 (Entity) | 完全自動化可能 | IntelliJ JPA Buddy |
| 3. 変換層 (MapStruct) | 一部自動化可能 | マッパー定義のインターフェース生成 |
| 4. データアクセス層 (Repository) | 自動化不可能（要プロンプト） | Claude 4.6 (Blaze-Persistence構築) |
| 5. 業務ロジック層 (Domain/Service) | 自動化不可能（要プロンプト） | Claude 4.6 (ロジック実装) |

------------------------------
## 各レイヤー・工程ごとの具体的手順## 1. コントローラー層 (Controller)

* 自動化できる箇所: APIのインターフェース、リクエスト/レスポンスDTOの生成。
* 自動化できない箇所: 自動生成されたインターフェースを実装（implements）する実コントローラークラスの作成。

* 【手順】:
1. openapi-generator-maven-plugin（またはGradleプラグイン）を実行し、定義済みのYAMLからKotlinコードを生成します。
2. 生成された ApiInterface を実装する Controller クラスを作成します。
3. クラス内の具体的な処理（Serviceの呼び出しとMapStructによる変換）は、GitHub Copilotに1行目を書かせて補完（Tabキー）させます。

## 2. データアクセス層：Entity (Entity)

* 自動化できる箇所: DDL（テーブル定義）からのKotlin @Entity クラスの自動生成。
* 自動化できない箇所: 複合主キーや、Blaze-Persistenceで利用する高度なリレーション定義（@ManyToOne のフェッチ設定など）の微調整。

* 【手順】:
1. IntelliJの JPA Buddy ツールウィンドウを開きます。
2. 「Entities from DB」またはDDLファイルを指定し、ターゲットとなるテーブルを選択してKotlinクラスを自動生成します。
3. 生成されたEntityに、クリーンアーキテクチャで必要なバリデーションやアノテーションが不足していないか目視で確認します。

## 3. 各層の変換層 (MapStruct)

* 自動化できる箇所: Entity ⇔ Domain、Domain ⇔ DTO 間のマッピングインターフェースの骨組み。
* 自動化できない箇所: フィールド名が異なる場合の @Mapping アノテーションによる明示的な対応付け、ネストされたオブジェクトのカスタム変換ロジック。

* 【手順】:
1. Claude 4.6 (Opus) に対し、ソース（例: Entity）とターゲット（例: Domain）の構造をコンテキストとして渡します。
2. 「これらを変換するMapStructのインターフェースを出力して」と指示し、アノテーション付きのコードを生成させます。
3. ビルドを一度実行し、MapStructの annotation processor が生成する実装クラス（~Impl.kt）のエラーの有無を確認します。

## 4. データアクセス層：クエリ (Repository / Blaze-Persistence)

* 自動化できる箇所: Spring Data JPAの標準的なCRUDメソッド（findById など）。
* 自動化できない箇所: Blaze-Persistence（CriteriaBuilder / CTE / 複雑なウインドウ関数など）を用いた複雑なクエリ実装。 AIはBlaze-Persistenceの最新仕様や複雑な文脈を誤解しやすいため、人間による厳密な指示が必要です。

* 【手順】:
1. 複雑な条件（集計、CTE、最適化フェッチなど）の要件を自然言語で書き出します。
2. Claude 4.6 (Opus) を使用し、対象の Entity 定義と、「Blaze-Persistenceの CriteriaBuilderFactory と EntityViewManager (使用する場合) を用いて、以下の要件を満たすクエリをKotlin 2.xで書いてください」とプロンプトを送ります。
3. 生成されたコードをRepositoryImplに貼り付け、型安全性が保たれているか（Kotlin 2.xのスマートキャストやコンパイルチェック）を確認しながら手動で修正します。 [1, 2] 

## 5. ドメイン・サービス層 (Domain / Service)

* 自動化できる箇所: インターフェース定義や、単純なCRUDを仲介するだけのユースケース。
* 自動化できない箇所: 外部仕様やビジネスルールに依存するコアロジックの組み立て。
* 【手順】:
1. ドメインモデル（Domain）のクリーンなオブジェクト（アノテーションを持たない純粋なKotlinの data class）をClaude 4.6を使って、要件定義から書き出させます。
2. Service レイヤーの実装では、GitHub Copilotのチャット機能（@workspace）を使い、「Repository からデータを取得し、Domain のルールに従って計算した結果を保存するロジックを書いて」と依頼します。
3. AIが生成したロジックに対して、エッジケース（Null安全、例外処理）を考慮するよう人間がレビュー・修正します。

------------------------------
## AI（GitHub Copilot / Claude 4.6）を活かす開発のコツ

* コンテキストの分離: クリーンアーキテクチャでは層が明確に分かれているため、AIに指示を出す際は「いまどの層の実装をしているか」を明確に伝えてください（例：「今は外部に依存しない domain 層のロジックを作っています」）。
* Blaze-Persistenceの補正: Copilotは一般的なJPA/Hibernateのコードを出力しがちです。Blaze-Persistenceのコードを書かせる際は、プロジェクト内にある他の正常なBlaze-Persistenceのコード（実装例）をIntelliJで開いた状態（Copilotの参照コンテキストに入る状態）にしてから生成させてください。

まずはどの工程から着手しますか？
もしよろしければ、最初のAPI（OpenAPIのyamlの一部）または対象テーブルのDDLを共有していただければ、それをベースにMapStructの定義やBlaze-PersistenceのRepository実装コードの具体的なプロンプト例・雛形を作成します。

[1] [https://kaopiz.com](https://kaopiz.com/ja-news-chatgpt-app-development-automation/)
[2] [https://relipasoft.com](https://relipasoft.com/blog/claude-code-and-n8n-integration-by-an-ai-system-development-company/)

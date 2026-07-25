システム開発の実装をAIで自動化できる・できない箇所とそれぞれの方法と具体的な手順を各レイヤー・工程ごとに分かり易く整理して教えて。

システム種類：WebAPI。
プログラミング言語：kotlin 2.x。
環境：spring boot4, openApi, Spring Data JPA Blaze-Persistence、MapStruct。
条件：クリーンアーキテクチャにて実装する。
実装：Blaze-Persistenceを用いた複雑なクエリが必要になる。
AI：GitHub CopilotからClaude Opus 4.6を使用する。

DB設計にてテーブル定義のDDLがあるので、DBマイグレーションはしない。
OpenApiのyamlは設計にて作成済で、OpenAPIのyamlからコードを自動生成するopenapi-generatorも導入済。
Kotlinの@Entityクラスの生成は、IntelliJのJPA Buddyを使用する。
@Entityはドメインモデルと完全に分ける。

クリーンアーキテクチャは、
・コントローラー:controller
・ドメインモデル:domain
・データアクセス:entity
・サービス:Service
・リポジトリ：Repository。

各層の変換にはMapStructを使用。

---

### GitHub Copilot（Claude 4.6 Opus搭載）を活用し、指定された技術スタック（Kotlin 2.x, Spring Boot 4, Blaze-Persistence, MapStruct）かつクリーンアーキテクチャの条件下で、WebAPIの実装を自動化する「できる・できない」の境界線と、具体的な手順を整理しました。
結論として、ボイラープレート（定型文）コードの生成や単純なマッピング、基本的なロジックの実装は90%以上自動化可能ですが、Blaze-Persistenceを用いた複雑なクエリの結合、クリーンアーキテクチャの境界定義、ドメイン層のコアイミュータブル設計はAIの補助が必要（自動化は不可）となります。
------------------------------
## 全体アーキテクチャとAIの自動化カバー率
各レイヤー・工程におけるAIの自動化可否のサマリーです。

| レイヤー / 工程 | 自動化の可否 | AIの役割 / 手法 |
|---|---|---|
| 0. 前準備 (OpenAPI / Entity) | 完全自動化 | openapi-generator と JPA Buddy で100%自動生成。 |
| 1. プレゼンテーション層 (Controller) | 高確率で自動化可 | 自動生成されたInterfaceの実装クラスをCopilotで生成。 |
| 2. ドメイン層 (Domain Model) | 一部自動化可 (人間が主導) | ビジネスロジックの記述は可能。不変性の設計は人間がレビュー。 |
| 3. データアクセス層 (Repository/Entity) | 一部自動化可 (要精密プロンプト) | Blaze-Persistenceの複雑なCTEや結合はAI単体ではバグが出るため手動修正必須。 |
| 4. マッピング層 (MapStruct) | ほぼ完全自動化 | インターフェースの定義と、カスタムマッパーの生成。 |

------------------------------
## 各レイヤー・工程ごとの詳細手順## 0. 設計・前準備工程（自動生成）
この工程はAI（Copilot）ではなく、指定されたツールで100%自動化します。

* できること:
   * OpenAPI yamlからControllerのインターフェース、Request/Response DTOの自動生成（openapi-generator）。
   * DDLからの@Entityクラス、@Idやリレーション（@ManyToOne等）の自動生成（IntelliJ JPA Buddy）。
* できないこと: 特になし（ツールが全て担保）。

------------------------------
## 1. プレゼンテーション層 (Controller)
openapi-generatorが生成したInterfaceを実装（implements）する具体的なControllerクラスを作成します。

* 自動化できる箇所:
  * Interfaceを満たすメソッドのオーバーライド。
     * 引数（DTO）からドメインモデルへのMapStructマッパーの呼び出し。
     * Service層のメソッド呼び出しと、レスポンスDTOへの変換処理。
* 自動化できない箇所:
  * HTTPステータスコードの微調整や、例外ハンドリング（@RestControllerAdvice）の設計判断。
  * 具体的なAI手順:
    1. 作成した空のControllerクラスを開く。
    2. Copilotのチャット（またはインライン生成）に以下を指示。
   
      @RestController
      クラス名 : XxxController : XxxApi（生成されたインターフェース）
   
   手順：
   1. XxxService と XxxMapper (MapStruct) をコンストラクタインジェクションする。
   2. 各エンドポイントで、リクエストDTOをMapperでドメインモデルに変換し、Serviceに渡す。
   3. Serviceの結果をMapperでレスポンスDTOに変換して返す。
   Kotlin 2.xの記法に従うこと。
   
------------------------------
## 2. ドメイン層 (Domain Model)
クリーンアーキテクチャの核心であり、外部（SpringやJPA）に依存しない純粋なKotlinコードで記述します。

* 自動化できる箇所:
* 値オブジェクト（Value Object）や不変（Immutable）なdata classの作成。
   * バリデーションロジックや、明確なビジネスルールの関数実装。
* 自動化できない箇所:
* ビジネスロジックのドメインモデルへの適切なカプセル化（AIは油断するとService層にロジックを逃がしがちになります）。
* 具体的なAI手順:
1. 要件定義や仕様書のテキストをCopilot（Claude 4.6 Opus）に読み込ませる。
   2. プロンプト例：
   
   以下の仕様を満たす純粋なKotlinのドメインモデル（data class）を作成してください。
   仕様：[ここに仕様を記述]
   条件：
   - SpringやJPAの依存は一切含めないこと。
   - 状態変更はメソッドを通じて新しいインスタンスを返す（不変オブジェクト）にすること。
   
   
------------------------------
## 3. データアクセス層 (Repository / Entity / Blaze-Persistence)
ドメインリポジトリのインターフェースを、Blaze-PersistenceとSpring Data JPAを組み合わせて実装する最難関の工程です。

* 自動化できる箇所:
* 基本的なCRUD処理。
   * BlazePersistenceRepositoryやCriteriaBuilderFactoryのボイラープレートコード（DI部分など）の記述。
* 自動化できない箇所:
* Blaze-Persistenceを用いた複雑なクエリ（CTE、Window関数、複雑なJOIN）の正確な生成。
   * 理由: Claude 4.6 Opusであっても、Blaze-PersistenceのKotlin DSLやJava APIの最新仕様、およびドメインモデルとEntityの完全分離ルールを完璧に把握しきれず、型エラーや存在しないメソッドを生成する確率が非常に高いためです。
* 具体的なAI手順（部分自動化）:
1. Entity、ドメインモデル、およびSQL（またはやりたいこと）の3つをCopilotに提示。
   2. プロンプト例：
   
   以下のEntityクラスとドメインモデルがあります。
   Blaze-Persistenceの `CriteriaBuilderFactory` と `EntityViewManager` (使用する場合) を使用して、[やりたい複雑な集計・結合処理] を行うリポジトリ実装クラスを作成してください。
   
   注意：
   - 戻り値はEntityではなく、必ずドメインモデル（またはそのリスト）に変換して返してください。
   - Blaze-Persistenceのクエリビルダの型安全性を意識してください。
   
   3. 人間による修正: 生成されたコードのコンパイルエラー（特にBlaze-Persistence特有の流れるようなインターフェースの型不一致）を、IntelliJのコンパイルエラーを見ながら人間が1行ずつ修正・補完します。

------------------------------
## 4. マッピング層 (MapStruct)
Entity ⇄ Domain Model ⇄ DTO の相互変換を定義します。

* 自動化できる箇所:
* MapStructの @Mapper インターフェースの定義。
   * フィールド名が一致している単純なマッピングの記述。
* 自動化できない箇所:
* Entityとドメインモデルで構造が大きく異なる場合（例：Entity側はフラットだがドメイン側は値オブジェクトでネストしているなど）のカスタムマッピング（@Mapping(target = "...", expression = "...")）の完全な自動化。
* 具体的なAI手順:
1. 変換元（例：XxxEntity）と変換先（例：XxxDomain）のコードを両方開く。
   2. Copilotにマッパーの作成を依頼。
   
   変換元クラスと変換先クラスを元に、MapStruct 1.6以降に対応したKotlinのマッパーインターフェースを作成してください。
   
   条件：
   - Springのコンポーネントモデル（componentModel = "spring"）にすること。
   - フィールド名が異なる以下の箇所は、適切な@Mappingアノテーションを付与して。
     - 変換元.hoge_id -> 変換先.hogeId
   
   
------------------------------
## AI自動化を成功させるための注意点（Kotlin 2.x × Spring Boot 4）

   1. コンテキスト（文脈）の共有:
   GitHub Copilotチャットを使用する際、必ず関係する「OpenAPIのyaml」「JPA Buddyが作ったEntity」「ドメインモデル」のファイルをすべて開いた状態（または #file で明示的に指定）で指示を出してください。これを怠ると、AIが架空のプロパティ名を捏造します。
   2. Spring Boot 4 / Kotlin 2.x の最新仕様への追従:
   AIは古いSpring Boot 2.xや3.xの記法、Kotlin 1.xの記法を混ぜて出力することがあります。コンパイルエラーが発生した場合は、エラーメッセージをそのままCopilotに貼り付けて 「Kotlin 2.x / Spring Boot 4 の仕様に合わせて修正して」 と指示してください。

まずは最初のステップとして、すでに手元にあるOpenAPIのyamlファイル、またはJPA Buddyで生成したEntityクラスを1つ教えていただけますか？
それらをベースに、Claude 4.6 Opusにそのまま入力できる「MapStructマッパー」や「Controllerクラス」を生成するための具体的なプロンプトを実際に作成してお見せします。


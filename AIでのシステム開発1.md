# AI システム開発 実装ガイド

自動化できる・できない箇所とそれぞれの方法と具体的な手順を各レイヤー・工程ごとに分かり易く整理しました。

技術スタック：**Kotlin 2.x**、**Spring Boot 4**、**OpenAPI**、**Blaze-Persistence**、**MapStruct**、**クリーンアーキテクチャ**

---

## 全体マップと自動化比率

| レイヤー / 工程 | 自動化の可否 | 主要ツール / AIの役割 |
| --- | --- | --- |
| 1. ルーティング層 (Controller) | 完全自動化可能 | OpenAPI Generator |
| 2. データモデル層 (Entity) | 完全自動化可能 | IntelliJ JPA Buddy |
| 3. 変換層 (MapStruct) | 一部自動化可能 | マッパー定義のインターフェース生成 |
| 4. データアクセス層 (Repository) | 自動化不可能（要プロンプト） | Claude 4.6 (Blaze-Persistence構築) |
| 5. 業務ロジック層 (Domain/Service) | 自動化不可能（要プロンプト） | Claude 4.6 (ロジック実装) |

---

## 各レイヤー・工程ごとの具体的手順

### 1. コントローラー層 (Controller)

#### 自動化できる箇所
- APIのインターフェース定義
- リクエスト/レスポンスDTOの生成

#### 自動化できない箇所
- 自動生成されたインターフェースを実装（implements）する実コントローラークラスの作成

#### 【手順】

1. `openapi-generator-maven-plugin`（またはGradleプラグイン）を実行し、定義済みのYAMLからKotlinコードを生成します
2. 生成された `ApiInterface` を実装する `Controller` クラスを作成します
3. クラス内の具体的な処理（Serviceの呼び出しとMapStructによる変換）は、GitHub Copilotに1行目を書かせて補完（Tabキー）させます

---

### 2. データアクセス層：Entity

#### 自動化できる箇所
- DDL（テーブル定義）からのKotlin `@Entity` クラスの自動生成

#### 自動化できない箇所
- 複合主キーの設定
- Blaze-Persistenceで利用する高度なリレーション定義（`@ManyToOne` のフェッチ設定など）の微調整

#### 【手順】

1. IntelliJの **JPA Buddy** ツールウィンドウを開きます
2. 「Entities from DB」またはDDLファイルを指定し、ターゲットとなるテーブルを選択してKotlinクラスを自動生成します
3. 生成されたEntityに、クリーンアーキテクチャで必要なバリデーションやアノテーションが不足していないか目視で確認します

---

### 3. 変換層 (MapStruct)

#### 自動化できる箇所
- `Entity ⇔ Domain`、`Domain ⇔ DTO` 間のマッピングインターフェースの骨組み

#### 自動化できない箇所
- フィールド名が異なる場合の `@Mapping` アノテーションによる明示的な対応付け
- ネストされたオブジェクトのカスタム変換ロジック

#### 【手順】

1. Claude 4.6 (Opus) に対し、ソース（例: Entity）とターゲット（例: Domain）の構造をコンテキストとして渡します
2. 「これらを変換するMapStructのインターフェースを出力して」と指示し、アノテーション付きのコードを生成させます
3. ビルドを一度実行し、MapStructの annotation processor が生成する実装クラス（`~Impl.kt`）のエラーの有無を確認します

---

### 4. データアクセス層：クエリ (Repository / Blaze-Persistence)

#### 自動化できる箇所
- Spring Data JPAの標準的なCRUDメソッド（`findById` など）

#### 自動化できない箇所
- Blaze-Persistence（CriteriaBuilder / CTE / 複雑なウインドウ関数など）を用いた複雑なクエリ実装
- AIはBlaze-Persistenceの最新仕様や複雑な最適化の詳細を常には把握していません

#### 【手順】

1. 複雑な条件（集計、CTE、最適化フェッチなど）の要件を自然言語で書き出します
2. Claude 4.6 (Opus) を使用し、対象の Entity 定義と共に以下の要件を提示します：
   - 「Blaze-Persistenceの `CriteriaBuilderFactory` と `EntityViewManager` （使用する場合）を用いて、以下の要件を満たすクエリを実装してください」
3. 生成されたコードをRepositoryImplに貼り付け、型安全性が保たれているか（Kotlin 2.xのスマートキャストやコンパイルチェック）を確認しながら手動で調整します

---

### 5. ドメイン・サービス層 (Domain / Service)

#### 自動化できる箇所
- インターフェース定義
- 単純なCRUDを仲介するだけのユースケース

#### 自動化できない箇所
- 外部仕様やビジネスルールに依存するコアロジックの組み立て

#### 【手順】

1. ドメインモデル（Domain）のクリーンなオブジェクト（アノテーションを持たない純粋なKotlinの `data class`）をClaude 4.6を使って、要件定義から書き出します
2. Service レイヤーの実装では、GitHub Copilotのチャット機能（`@workspace`）を使い、以下を指示します：
   - 「Repository からデータを取得し、Domain のルールに従って計算した結果を返してください」
3. AIが生成したロジックに対して、エッジケース（Null安全、例外処理）を考慮するよう人間がレビュー・修正します

---

## AI（GitHub Copilot / Claude 4.6）を活かす開発のコツ

### コンテキストの分離
クリーンアーキテクチャでは層が明確に分かれているため、AIに指示を出す際は「いまどの層の実装をしているか」を明確に伝えることが重要です。

### Blaze-Persistenceの補正
Copilotは一般的なJPA/Hibernateのコードを出力しがちです。Blaze-Persistenceのコードを書かせる際は、プロジェクト内にある他の正常に動作しているクエリ例を参照させることで、AI生成コードの精度が向上します。

---

## 次のステップ

まずはどの工程から着手しますか？

もしよろしければ、最初のAPI（OpenAPIのYAML）または対象テーブルのDDLを共有していただければ、それをベースにMapStructの定義やBlaze-Persistenceのクエリ例を提供できます。

---

## 参考資料

1. [ChatGPT App Development Automation - kaopiz](https://kaopiz.com/ja-news-chatgpt-app-development-automation/)
2. [Claude Code and N8N Integration - relipasoft](https://relipasoft.com/blog/claude-code-and-n8n-integration-by-an-ai-system-development-company/)

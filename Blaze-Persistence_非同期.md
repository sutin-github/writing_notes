Blaze-Persistence jOOQ 非同期

Blaze-Persistence と jOOQ はそれぞれ異なるデータベース操作ライブラリであり、直接的な非同期処理の連携機能や統合された非同期APIは用意されていません。それぞれ個別に非同期処理（R2DBCやCompletionStageなど）を組み合わせて利用する必要があります。 [1, 2] 

## 特徴と非同期の仕組み

* Blaze-Persistence: JPA/Hibernateを拡張するライブラリであり、基本は同期的なJPAのEntityManagerをベースに動作します。
* jOOQ: 型安全なSQL構築ツールであり、R2DBCを通じたリアクティブ/非同期実行をサポートしています。
* 非同期の適用: それぞれ独立した技術スタックであるため、両者を1つの非同期パイプラインで直接繋ぐことはできません。 [1, 2] 

もしよろしければ、以下について教えてください：

* どちらのライブラリ（または両方）をどのように組み合わせて使いたいですか？
* 実現したい非同期処理の具体的な要件（Spring WebFlux、R2DBC、CompletableFutureなど）は何ですか？

状況に合わせてさらに詳しくお答えします。

[1] [https://medium.com](https://medium.com/@kiarash.shamaii/dynamic-filtering-in-spring-boot-jpa-vs-jooq-vs-blaze-persistence-14246f9b4f12)
[2] [https://github.com](https://github.com/tuyucheng7/awesome-java)

Blaze-Persistence を非同期で

Blaze-Persistence は JPA（Java Persistence API）や Hibernate を拡張するライブラリであるため、基本的にはブロッキング（同期）I/Oを前提として設計されています。そのため、Spring WebFlux や R2DBC のようにネイティブでノンブロッキングな非同期処理を行うことはできません。
Blaze-Persistence を非同期環境で利用する場合は、別スレッド（専用のスレッドプール）に処理を委譲してノンブロッキングに見せるアプローチが一般的です。

------------------------------
## 1. 仮想スレッド（Java 21以降）を活用する
Java 21 以降を利用している場合、仮想スレッド（Virtual Threads）上で Blaze-Persistence を実行するのが最もクリーンで推奨される方法です。

* 仕組み: データベースのI/O待ちが発生した際、OSのスレッドをブロックせず、Java仮想マシン（JVM）レベルでスレッドが切り替わります。
* メリット: コードを非同期（リアクティブ）用に書き換える必要がなく、同期コードのままで高い並行性を実現できます。
* 実装例（Spring Boot 3.2以降）: application.properties に以下を追加するだけで、Tomcat やタスク実行が仮想スレッドに切り替わります。

```properties
spring.threads.virtual.enabled=true
```

------------------------------
## 2. CompletableFuture と専用プールを使う
Java 8 以降や、仮想スレッドが使えない環境では、CompletableFuture を使って特定のクエリ実行を別スレッドに切り離します。

* 注意点: データベースの接続数（HikariCPなど）を使い果たさないよう、必ず JPA専用にサイズ制限したスレッドプール（Executor） を用意してください。

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import jakarta.inject.Inject;

public class AsyncUserService {

    @Inject
    private CriteriaBuilderFactory cbf;

    @Inject
    private EntityManager em;

    @Inject
    private Executor jpaExecutor; // JPA専用に隔離されたスレッドプール

    public CompletableFuture<List<UserDto>> getUsersAsync(int age) {
        return CompletableFuture.supplyAsync(() -> {
            // Blaze-Persistence のクエリ構築と実行（ブロッキング）
            return cbf.create(em, UserDto.class)
                .from(User.class, "u")
                .where("u.age").gt(age)
                .getResultList();
        }, jpaExecutor);
    }
}
```

------------------------------
## 3. Spring Data JPA の @Async を併用する
Spring Data JPA と Blaze-Persistence の統合機能（Blaze-Persistence Spring Data Integration）を使っている場合、Spring の @Async アノテーションを利用して、リポジトリのメソッド呼び出し自体を非同期化できます。

```java
import org.springframework.scheduling.annotation.Async;
import java.util.concurrent.CompletableFuture;
import java.util.List;

public interface UserRepository extends EntityViewRepository<UserDto, Long> {

    @Async("jpaTaskExecutor") // 専用のタスクエグゼキューターを指定
    CompletableFuture<List<UserDto>> findByAgeGreaterThan(int age);
}
```

------------------------------
## ⚠️ Blaze-Persistence を非同期で扱う際の注意点

* セッションの生存期間: JPA の EntityManager（セッション）はスレッドセーフではありません。メインスレッドのセッションをそのまま非同期スレッド（CompletableFuture の中など）に持ち込むと、高確率で競合エラーやデータ破損が発生します。非同期スレッド側で個別にトランザクションとセッションを開くか、スコープを完全に分離してください。
* 遅延ローディング（Lazy Loading）の禁止: 非同期スレッドにエンティティを渡した後、スレッドの外やセッション終了後に getBooks() などの遅延コレクションにアクセスすると LazyInitializationException が発生します。Blaze-Persistence の Entity Views や fetch 結合を活用し、必要なデータをクエリ実行時にすべて一括取得（Eager Fetch）してください。

非同期処理を導入しようとしている全体のアーキテクチャについて教えていただけますか？

* アプリケーションの基盤: Spring Boot（WebMVC または WebFlux）、Jakarta EE、あるいは Quarkus などでしょうか？
* 非同期にしたい理由: 外部APIと並列でクエリを投げたい、リクエストスレッドを解放してスループットを上げたい、など具体的な目的を教えていただければ、最適なスレッドプールの設計やトランザクション管理についてご提案できます。

Spring Boot　リクエストスレッドを解放してスループットを上げたい

Spring Boot でリクエストスレッド（Tomcat 等の Servlet スレッド）を即座に解放し、アプリケーション全体の同時並行処理数（スループット）を最大化するには、Java 21以降の仮想スレッド（Virtual Threads）を有効化するのが最も効果的で現代的なアプローチです。 [1, 2] 
JPA と Blaze-Persistence を利用している環境において、スループットを向上させるための2つのアプローチとそれぞれの実装方法・注意点を解説します。

------------------------------
## アプローチ1：仮想スレッド（Virtual Threads）の有効化【最推奨】
Java 21 以降および Spring Boot 3.2 以降を使用している場合、プロパティを1行追加するだけで、Tomcat のリクエスト処理がすべて仮想スレッド上で実行されるようになります。 [2] 

* 仕組み: データベースアクセス（I/O待ち）が発生した際、OSスレッドをブロックせず、Java仮想マシン（JVM）レベルで別のリクエスト処理へ一瞬でスレッドを切り替えます。 [1] 
* メリット: コードを非同期（リアクティブや CompletableFuture）に書き換える必要が一切なく、JPA/Blaze-Persistence の同期的なコードそのままで数万リクエストを同時に捌けるようになります。 [1, 3] 

設定方法 (application.properties):

```properties
spring.threads.virtual.enabled=true
```

------------------------------
## アプローチ2：CompletableFuture を使った非同期リクエストハンドリング
Java 21 未満の環境や、特定のリクエストだけを完全にバックグラウンドで処理して Tomcat スレッドを解放したい場合は、コントローラーから CompletableFuture を返却します。 [4] 

* 仕組み: リクエストを受け取った Tomcat スレッドは、CompletableFuture を作成して即座にプールへ返却（解放）されます。その後、JPA専用に用意した別スレッドプールがデータを処理し、完了した段階でレスポンスがクライアントに送信されます。 [4, 5] 

## 1. 非同期処理用のスレッドプール（Executor）の設定
JPA（Blaze-Persistence）用のスレッドプールを明示的に作成します。

```java
@Configuration
@EnableAsyncpublic class AsyncConfig {

    @Bean(name = "jpaTaskExecutor")
    public Executor jpaTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);        // 常に維持するスレッド数
        executor.setMaxPoolSize(50);         // 最大スレッド数
        executor.setQueueCapacity(100);      // 待ち行列の許容量
        executor.setThreadNamePrefix("JPAAsync-");
        executor.initialize();
        return executor;
    }
}
```

## 2. コントローラーとサービスの実装
コントローラーの戻り値を CompletableFuture<T> にします。 [4] 

```java
@RestController
@RequestMapping("/api/users")public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public CompletableFuture<ResponseEntity<List<UserDto>>> getUsers(@RequestParam int age) {
        // この時点でTomcatスレッドは解放可能状態になります
        return userService.getUsersAsync(age)
                .thenApply(ResponseEntity::ok);
    }
}

@Servicepublic class UserService {

    @Autowired
    private CriteriaBuilderFactory cbf;
    @Autowired
    private EntityManager em;
    
    @Autowired
    @Qualifier("jpaTaskExecutor")
    private Executor jpaTaskExecutor;

    public CompletableFuture<List<UserDto>> getUsersAsync(int age) {
        return CompletableFuture.supplyAsync(() -> {
            // 別スレッドでBlaze-Persistenceのクエリを実行
            return cbf.create(em, UserDto.class)
                    .from(User.class, "u")
                    .where("u.age").gt(age)
                    .getResultList();
        }, jpaTaskExecutor);
    }
}
```

------------------------------
## ⚠️ リクエストスレッドを解放する際の重要な注意点（ボトルネックの移動）
JPA環境でスループットを上げようとすると、スレッド解放後に別の場所がボトルネック化します。以下の2点を必ずセットでチューニングしてください。

   1. DB接続プール（HikariCP）の枯渇に注意
   Tomcatスレッドをいくら解放して大量のリクエストを受け入れても、データベースへの接続数（spring.datasource.hikari.maximum-pool-size）が足りなければ、今度は「DB接続待ち」で処理が詰まります。
   * 仮想スレッドや非同期スレッドの最大数に合わせて、HikariCP の最大接続数やデータベース側の最大コネクション数（PostgreSQLの max_connections など）を適切に引き上げてください。 [6] 
   2. Entity Views（Blaze-Persistence機能）の徹底活用
   非同期処理（別スレッド）に処理が移った後、JPAエンティティの遅延ローディング（Lazy Initialization）を行うと、セッション切れでエラーになるか、追加のSQL（N+1問題）が裏で大量に走りDBがパンクします。
   * Blaze-Persistence の Entity Views 機能を使って、必要なデータを1回のクエリでDTO（インターフェース）にマッピングして取得し、エンティティのままスレッド間で受け渡さないようにしてください。 [3, 7] 
   
非同期化によるスループット向上のために、どちらの方法が適していそうでしょうか？

* Java 21以降を利用しており、アプリ全体をシンプルに高速化したい（アプローチ1）
* Java 17以前であるか、特定の重い参照クエリだけを制御したい（アプローチ2）

現在利用されている Javaのバージョン や、想定しているデータベースの接続数（プールサイズ） に不安があれば、具体的な設定値の計算などをサポートします。

[1] [https://jp.linkedin.com](https://jp.linkedin.com/pulse/spring-boot-meets-java-virtual-threads-high-lower-cost-patel-mfqrc?tl=ja)
[2] [https://springboot-123.mizucoffee.com](https://springboot-123.mizucoffee.com/ja/blog/spring-boot-virtual-threads-java21-guide/)
[3] [https://blog.devgenius.io](https://blog.devgenius.io/spring-boot-jpa-virtual-threads-java-21-avoid-n-1-osiv-pagination-pitfalls-0bde190da6bc)
[4] [https://alexanderobregon.substack.com](https://alexanderobregon.substack.com/p/async-request-processing-in-spring)
[5] [https://medium.com](https://medium.com/@AlexanderObregon/async-request-processing-in-spring-boot-with-completablefuture-13f00243bba7)
[6] [https://www.infoq.com](https://www.infoq.com/articles/virtual-threads-after-jdk24/)
[7] [https://www.baeldung.com](https://www.baeldung.com/blaze-persistence-tutorial)

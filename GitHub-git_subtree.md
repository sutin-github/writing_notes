## サブツリーのGitHubでの使い方

GitサブツリーはGitの標準機能（git subtree）として提供されているため、GitHub上での特別な設定は不要です。すべて自身のPC（ローカル）のターミナル操作で完結します。
以下に、GitサブツリーをGitHub連携リポジトリで使うための具体的な手順（追加・更新・変更の書き戻し）をまとめました。

------------------------------
## 1. サブツリーの追加（外部リポジトリを取り込む）
親リポジトリのローカル環境で以下のコマンドを実行し、GitHub上にある別リポジトリ（子）を指定したディレクトリ（パス）に結合します。

git subtree add --prefix=<配置するディレクトリ名> <子リポジトリのGitHub URL> <ブランチ名> --squash


* --prefix=: 取り込んだファイルを配置するフォルダ名（例: src/common や themes/my-theme）
* --squash: 子リポジトリのこれまでの全コミット履歴を1つにまとめて取り込みます（必須推奨。これを付けないと、子リポジトリの過去の履歴がすべて親の履歴に混ざってしまい見づらくなります）。

実行後、GitHub（親）へ通常通り git push すれば、GitHub上にもフォルダとファイルの実体がそのまま反映されます。

------------------------------

## 2. 外部（GitHub）の最新コードを追従する（Pull）
子リポジトリが別の開発者によってGitHub上で更新され、その最新状態を親リポジトリに取り込みたい場合は、以下のコマンドを実行します。

git subtree pull --prefix=<配置したディレクトリ名> <子リポジトリのGitHub URL> <ブランチ名> --squash

実行すると、最新の変更分が自動的にマージされます。マージ後は親リポジトリを git push してGitHubに同期させてください。

------------------------------

## 3. 親側で修正したコードを子（GitHub）に書き戻す（Push）
親リポジトリ内で、サブツリー内のファイルを直接編集してコミット（通常の git commit）することがあります。その変更を、元の子リポジトリ（GitHub）側にも反映させたい場合は以下を実行します。

git subtree push --prefix=<配置したディレクトリ名> <子リポジトリのGitHub URL> <ブランチ名>

Gitが親リポジトリのコミット履歴から「サブツリー内のフォルダに関する変更だけ」を自動的に抽出して、指定した子リポジトリのGitHubブランチへプッシュしてくれます。

------------------------------

## 💡 より運用を楽にするTips：リモート登録
毎回長いGitHubのURLを入力するのは大変なため、子リポジトリをリモート（別名）として登録しておくとコマンドが短くなり、打ち間違いを防げます。

### 最初の一度だけ子リポジトリを「child-repo」という名前で登録
git remote add child-repo <子リポジトリのGitHub URL>

### 追加
git subtree add --prefix=src/common child-repo main --squash

### 更新（Pull）
git subtree pull --prefix=src/common child-repo main --squash

### 書き戻し（Push）
git subtree push --prefix=src/common child-repo main

次に進むステップとして、どちらの内容が知りたいですか？

* 実際に試すための、具体的なファイル構成とコマンドの実行例
* サブツリーを操作する際に注意すべきコンフリクト（衝突）の防ぎ方

## JPAエンティティを共通リポジトリから取り込む方法

Java（Spring Bootなど）のJPAエンティティを共通リポジトリ（Gitリポジトリ）からGit Subtreeを使って取り込み、プロジェクトで利用できるようにする手順は以下の通りです。
JPAエンティティはJavaコードそのものであるため、Git Subtreeでソースコードを取り込んだ後、ビルドツール（GradleまたはMaven）側でそのソースフォルダを認識させる設定が必要になります。

------------------------------

## 全体手順（Gradleの場合）
共通リポジトリ（例: common-entities）からエンティティを取り込み、プロジェクトのビルド対象に含める具体的な流れです。

## 1. Git Subtreeでエンティティを取り込む
まず、メインプロジェクトのルートディレクトリで以下のコマンドを実行し、共通エンティティのコードを特定のフォルダ（例: src/main/java-common）に引き込みます。

### 短く管理するためにリモート登録

git remote add common-repo <共通エンティティリポジトリのGitHub URL>

### サブツリーとして追加（--squashで履歴を集約）

git subtree add --prefix=src/main/java-common common-repo main --squash

## 2. ビルドツール（build.gradle）にソースフォルダを追加する

標準の src/main/java 以外に、今取り込んだ src/main/java-common もJavaのソースコードとしてコンパイル対象に含めるよう、build.gradle に設定を追加します。

```kts
sourceSets {
    main {
        java {
            srcDirs = ['src/main/java', 'src/main/java-common']
        }
    }
}
```

※この設定をすることで、IDE（IntelliJやEclipse）やビルドツールが取り込んだJPAエンティティクラスを正しく認識し、インポートできるようになります。

## 3. Spring Bootのエンティティスキャン対象に加える
メインプロジェクトと共通エンティティでJavaのパッケージ名（com.example...）が異なる場合、Spring BootがJPAエンティティを自動検出できません。
メインのアプリケーション起動クラス（@SpringBootApplication）に、スキャン対象のパッケージを明示的に指定します。

```java
package com.example.mainapp;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@SpringBootApplication
@EntityScan(basePackages = {
    "com.example.mainapp",       // メインプロジェクトのパッケージ
    "com.example.common.entity"  // 取り込んだ共通JPAエンティティのパッケージ
})
@EnableJpaRepositories(basePackages = {
    "com.example.mainapp",
    "com.example.common.repository" // リポジトリクラスも共通化している場合
})public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

------------------------------

## 💡 Maven（pom.xml）の場合の設定
もしビルドツールに Maven を使用している場合は、手順2の代わりに pom.xml へ build-helper-maven-plugin を追加してソースディレクトリを拡張します。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
            <version>3.5.0</version> <!-- 2026年時点の安定版 -->
            <executions>
                <execution>
                    <id>add-source</id>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>add-source</goal>
                    </goals>
                    <configuration>
                        <sources>
                            <source>src/main/java-common</source>
                        </sources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

------------------------------

## ⚠️ JPAエンティティをGitで直接共有する際の注意点

* メタモデル（Querydsl等）の生成: もし Querydsl などを利用して QUser などのメタモデルを自動生成している場合、共通コード側ではなく、取り込み先のメインプロジェクト側で再度コード生成（ビルド）が走るように設定を確認してください。
* データベース（DB）スキーマの同期: 共通リポジトリのエンティティが変更（フィールドの追加など）された場合は、忘れずにメイン側で git subtree pull を行い、ローカルのDBマイグレーション（FlywayやLiquibaseなど）のバージョンと整合性を保つ必要があります。

現在の開発環境に合わせて調整します。以下について教えてください。

* ビルドツールは Gradle と Maven のどちらを使っていますか？
* 共通リポジトリ側には、JPAエンティティの他に Spring Data JPAの「Repository（インターフェース）」 も含まれていますか？

## Gradleで、共通リポジトリはGradleプロジェクトでjarを作成するプロジェクトになっている

共通リポジトリが独立したGradleプロジェクトとしてビルドされ、JAR（ライブラリ）を出力する構成になっている場合、Git Subtreeで直接ソースコードフォルダ（src/main/java）を取り込んでしまうと、共通リポジトリ側の build.gradle や依存関係（ライブラリ）の定義が無視されてしまい、コンパイルエラーの原因になります。
この場合のベストプラクティスは、メインプロジェクトのルート直下に共通プロジェクトを丸ごとGit Subtreeで配置し、Gradleのマルチプロジェクト構成（マルチモジュール）として組み込む方法です。

------------------------------
## 1. 全体像（ディレクトリ構成）
メインプロジェクトの配下に、共通プロジェクト（common-jar-project）をSubtreeとして配置します。

```txt
main-project/
├── build.gradle (メイン)
├── settings.gradle (★ここで共通プロジェクトを結合する)
├── src/main/java/...
└── common-jar-project/  (★Git Subtreeで丸ごと配置)
    ├── build.gradle (共通)
    └── src/main/java/com/example/entity/... (JPAエンティティなど)
```

------------------------------
## 2. 具体的な手順## ステップ1: 共通プロジェクトをGit Subtreeで追加する
メインプロジェクトのルートディレクトリで以下を実行します。共通リポジトリを丸ごと、ルート直下のフォルダとして取り込みます。

### リモート登録（最初の一度だけ）

```bash
git remote add common-project-repo <共通プロジェクトのGitHub URL>
```

### ルート直下の「common-jar-project」というフォルダ名で取り込む

```bash
git subtree add --prefix=common-jar-project common-project-repo main --squash
```

## ステップ2: settings.gradle にモジュールとして登録する
メインプロジェクトの settings.gradle を開き、取り込んだ共通プロジェクトをGradleのサブモジュールとして認識させます。

```kts
rootProject.name = 'main-project'
```

// 共通プロジェクトをマルチプロジェクトとして含める

```kts
include 'common-jar-project'
```

## ステップ3: メインの build.gradle で依存関係（JAR）として追加する
メインプロジェクト側の build.gradle の dependencies に、共通プロジェクトをプロジェクト参照として追加します。これにより、Gradleが自動的に共通プロジェクトのJARをビルドし、メインプロジェクトにクラスパスを通します。

```kts
dependencies {
    // 共通プロジェクトをJARとして依存関係に追加
    implementation project(':common-jar-project')

    // メインプロジェクトに必要なその他の依存関係
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    // ...
}
```

## ステップ4: Spring Bootのエンティティスキャンを通す
前述の通り、共通JAR内のJPAエンティティのパッケージ名がメイン側と異なる場合は、メインの起動クラス（@SpringBootApplication）に @EntityScan を付与して明示的にスキャンさせてください。

```kts
@SpringBootApplication
@EntityScan(basePackages = {"com.example.mainapp", "com.example.common.entity"})public class MainApplication { ... }
```

------------------------------
## 💡 この構成にするメリット

* 依存ライブラリの自動解決: 共通プロジェクトの build.gradle に書かれているJPAの依存関係やライブラリが、そのままメインプロジェクトにも引き継がれます。
* ローカルでの同時編集・デバッグ: メインプロジェクトの開発中、取り込んだ common-jar-project 内のJPAエンティティを直接修正しても、Gradleが自動で再ビルドしてJARを反映してくれるため、開発効率が落ちません。
* 共通リポジトリへの容易な還元: 修正した内容は、前述の git subtree push コマンドで簡単に共通リポジトリのGitHubへ書き戻せます。

このマルチプロジェクト構成を進めるにあたって：

* 共通プロジェクト（JAR側）で使っている Spring BootやJPA（Hibernate）のバージョン と、メインプロジェクトのバージョンは一致していますか？
* 共通プロジェクト内で Lombok（@Data など） や Querydsl を使用していますか？設定に補足が必要な場合はお伝えできます。

## GitHubでのgitサブツリーを追加する場合の注意点

GitHub上で管理されているリモートリポジトリを Git Subtree（サブツリー） として追加・運用する場合、GitHubならではの仕様や、Gitの特性に起因する注意点がいくつかあります。
運用をスムーズに行うために、以下の4つの注意点を必ず押さえておきましょう。

------------------------------

## 1. --squash オプションは「最初から最後まで」統一する
サブツリーを追加（add）するときと、後から更新（pull）するときで、--squash オプションの有無を必ず統一してください。基本的には「常に付ける」のが鉄則です。

* 付けるメリット: 子リポジトリの大量のコミット履歴が1つに集約されるため、親リポジトリのGitHub上のコミット履歴（Networkグラフやコミットログ）が汚れません。
* 混ぜるとどうなるか: 最初に --squash を付けたのに、次回 pull するときに付け忘れると、Gitが履歴の共通点を見失い、大量のコンフリクト（衝突）が発生して修復が極めて困難になります。

------------------------------

## 2. GitHubのPR（プルリクエスト）やレビューの運用ルール
GitHub上でコードレビューを行う際、サブツリー特有の挙動に注意が必要です。

* 親リポジトリ側での見え方: サブツリー内のファイルを修正して親リポジトリでPRを作ると、レビューアからは「ただの重いファイル変更」に見えます。これが「共通ライブラリの修正」なのか「メインアプリの修正」なのかが区別しづらくなります。
* 書き戻し（push）のタイミング: 親リポジトリ側でマージされた後に、git subtree push を使って子リポジトリ（GitHub）へ変更を書き戻す運用になりますが、これを忘れると「親リポジトリは最新なのに、元の子リポジトリに修正が反映されていない」という同期漏れが発生します。

------------------------------

## 3. GitHub上の「アクセス権限（リポジトリの秘匿性）」
親リポジトリと子リポジトリのGitHub上の公開設定（Public / Private）の組み合わせに注意してください。

* Private（親）の中に Private（子）を入れる場合:
開発者個人のローカルPCで git subtree push/pull を行う際、その開発者が両方のリポジトリに対してGitHubのアクセス権限（Read/Write）を持っている必要があります。
* 一度取り込むと、親側では制限できない:
サブツリーはコードの実体をそのままコピーして親に含めるため、親リポジトリの権限を持つ人は、子リポジトリの権限を持っていなくても中のコードをすべて閲覧・編集できてしまいます。コードを隠したい場合は不向きです。

------------------------------

## 4. CI/CD（GitHub Actionsなど）のビルド負荷
サブツリーを導入して「マルチプロジェクト構成（前述のJAR構成など）」にすると、GitHub ActionsなどのCIツールでの挙動が変わります。

* ビルド時間の増加: 毎回、親プロジェクトのビルド時に共通リポジトリ（JAR）のビルドも走るため、CIの実行時間が長くなり、GitHub Actionsの無料枠（Minutes）を多く消費するようになります。
* テストの依存: 共通リポジトリ側のテストコードも一緒に走るように設定されている場合、共通側のテストが落ちるとメインアプリのデプロイも巻き添えで止まってしまいます。

------------------------------

## 💡 トラブルを防ぐための運用ベストプラクティス
これらを防ぐために、チームで以下のルールを決めておくことを強くおすすめします。

   1. 基本は「子リポジトリ側」で修正して親は pull するだけにする（親リポジトリ側から直接ガリガリ編集して subtree push する運用は、コンフリクトの元なので最小限にする）。
   2. コマンドの間違い（--squash の付け忘れなど）を防ぐため、Makefile や shellスクリプト、または Gradleの独自タスクに対象コマンドを共通化して登録しておく。

もしチーム開発で導入される場合、

* ミスを防ぐためのスクリプト化（Makefileなど）の具体例
* 万が一、更新（pull）時にコンフリクトしてしまった場合の対処法

のどちらについて詳しく知りたいですか？

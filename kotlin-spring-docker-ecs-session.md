# Kotlin Spring Boot Docker & AWS ECS Fargate 構築セッション

このファイルには、Kotlin + Spring Boot アプリケーションを OpenAPI Generator、Amazon Corretto 21、AWS Secrets Manager、Amazon RDS (PostgreSQL)、および Amazon ECS (Fargate) と連携して Docker で動作させるための全セッションのやりとりが記録されています。

---

## 1. Dockerfile の作成 (初回基本構成)

**Amazon Corretto 21（JDK 21）** を使用して、Kotlin + Spring Boot アプリケーションをビルド・実行するためのマルチステージ `Dockerfile` です。ビルド環境（Gradlewを使用）と実行環境（AlpineベースのCorretto）を分離し、軽量で安全な本番環境用イメージを構築します。

### Dockerfile
```dockerfile
# ==============================================================================
# 1. ビルドステージ (Build Stage)
# ==============================================================================
FROM amazoncorretto:21 AS builder

WORKDIR /app

# 依存関係のキャッシュ効率化のため、まずビルドスクリプト一式をコピー
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .

# gradlew に実行権限を付与し、依存関係を事前ダウンロード (ソースコードの変更に影響されないようにする)
RUN chmod +x ./gradlew && ./gradlew dependencies --no-daemon

# ソースコードをコピーしてビルドを実行 (テストはスキップ)
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# ==============================================================================
# 2. 実行ステージ (Runtime Stage)
# ==============================================================================
# 本番環境の軽量化とセキュリティ向上のため、Alpine Linux ベースの Corretto を使用
FROM amazoncorretto:21-alpine

WORKDIR /app

# セキュリティ対策：ルート権限ではない一般ユーザーを作成して実行
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# ビルドステージから生成された jar ファイルのみをコピー
COPY --from=builder /app/build/libs/*-SNAPSHOT.jar app.jar

# コンテナが開放するポートを指定 (デフォルトは 8080)
EXPOSE 8080

# メモリ最適化オプションを付けてアプリケーションを起動
ENTRYPOINT ["java", "-XX:+UseG1GC", "-jar", "app.jar"]
```

### コンテナのビルドと起動手順
```bash
# ① イメージのビルド
docker build -t kotlin-spring-app:latest .

# ② コンテナの起動
docker run -d -p 8080:8080 --name my-kotlin-app kotlin-spring-app:latest
```

---

## 2. AWS App Runner 向け CI/CD と環境変数の引き渡し

### 起動時の環境変数を Docker に渡す方法
本番環境のデータベース接続情報などの機密情報は、Dockerfile 内ではなくコンテナ起動時に外部から注入します。

#### Spring Boot 側の設定 (`src/main/resources/application.yml`)
```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${SPRING_DATASOURCE_USERNAME:postgres}
    password: ${SPRING_DATASOURCE_PASSWORD:password}
```

### GitHub Actions による CI/CD 設定 (`.github/workflows/deploy.yml`)
```yaml
name: Deploy to AWS App Runner

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  deploy:
    name: Build and Deploy
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-kotlin-spring-repo 
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

    - name: Deploy to Amazon App Runner
      run: |
        aws apprunner start-deployment --service-arn ${{ secrets.AWS_APP_RUNNER_SERVICE_ARN }}
```

---

## 3. OpenAPI Generator ✕ ECS Fargate ✕ Secrets Manager 完全最適化構成

OpenAPI Generator を使用する Kotlin + Spring Boot アプリケーション向けに、キャッシュ効率を極限まで高めた本番用構成です。

### ① 最適化された `Dockerfile`
「仕様書（YAML）の変更」と「コード（Kotlin）の変更」を分離してキャッシュさせる構造にしています。

```dockerfile
# ==============================================================================
# 1. ビルドステージ (Build Stage)
# ==============================================================================
FROM amazoncorretto:21 AS builder

WORKDIR /app

# Gradle ラッパーと設定ファイルのコピー (依存関係キャッシュ用)
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .

# OpenAPI 仕様書のみを先にコピー (コード生成キャッシュ用)
COPY src/main/resources/openapi.yaml src/main/resources/openapi.yaml

# 依存関係のダウンロードと OpenAPI コード生成を先に実行
RUN chmod +x ./gradlew && ./gradlew openApiGenerate dependencies --no-daemon

# 残りのソースコードをコピーして最終ビルド (テストはスキップ)
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

# ==============================================================================
# 2. 実行ステージ (Runtime Stage) - Fargate用に最適化
# ==============================================================================
FROM amazoncorretto:21-alpine

WORKDIR /app

# セキュリティ対策：root 権限を排除し、一般ユーザー(spring)を作成して実行
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# ビルドステージから実行可能な jar のみをクリーンにコピー
COPY --from=builder /app/build/libs/*-SNAPSHOT.jar app.jar

# ECS Fargate (ALB) からトラフィックを受けるポート
EXPOSE 8080

# コンテナ起動コマンド (G1GCによるメモリ最適化)
ENTRYPOINT ["java", "-XX:+UseG1GC", "-jar", "app.jar"]
```

### ② ローカル成果物の混入を防ぐ `.dockerignore`
```text
.gradle
.vsc/
.idea/
build/
out/
bin/
Dockerfile
.dockerignore
```

### ③ Spring Boot 側の RDS 接続設定 (`src/main/resources/application.yml`)
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${RDS_HOSTNAME:localhost}:${RDS_PORT:5432}/${RDS_DB_NAME:mydb}
    username: ${RDS_USERNAME:postgres}
    password: ${RDS_PASSWORD:password}
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate

management:
  endpoints:
    web:
      exposure:
        include: health
```

### ④ ECS Fargate タスク定義 (Task Definition JSON)
AWS Secrets Manager に登録した `username`, `password`, `host`, `port`, `dbname` の情報を、コンテナ起動時に自動で環境変数へと安全に展開します。

```json
{
  "containerDefinitions": [
    {
      "name": "kotlin-spring-openapi-app",
      "image": "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/my-kotlin-repo:latest",
      "cpu": 256,
      "memory": 512,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "SPRING_PROFILES_ACTIVE",
          "value": "prod"
        }
      ],
      "secrets": [
        {
          "name": "RDS_USERNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:my-rds-secret-AbCdEf:username::"
        },
        {
          "name": "RDS_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:my-rds-secret-AbCdEf:password::"
        },
        {
          "name": "RDS_HOSTNAME",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:my-rds-secret-AbCdEf:host::"
        },
        {
          "name": "RDS_PORT",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:my-rds-secret-AbCdEf:port::"
        },
        {
          "name": "RDS_DB_NAME",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:123456789012:secret:my-rds-secret-AbCdEf:dbname::"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/kotlin-spring-app",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512"
}
```
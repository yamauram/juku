# DynamoDB を使用した Spring アプリケーションの検証

## 背景・目的

- サーバレス API の検証で API Gateway + Lambda による構成での疎通はできたが、コールドスタートにより初回起動時のレスポンスに多大な時間がかかることがわかった。
- コールドスタート問題に対処する方法はいくつかあるが、起動の遅い VPC Lambda から非 VPC Lambda に変更することで高速化が図れないか検証する。
- 非 VPC Lambda に変更するため、受付情報の永続化に、RDS ではなく DynamoDB を使用する。

## 環境構築

### AWS リソースの設定

- DynamoDB で以下のテーブルを作成する。

  1. reception（受付情報管理テーブル）

     - `reception_number`（受付番号）をパーティションキーとし、以下の属性を持つ。
       - `request_file_path`（要求ファイルパス）
       - `response_file_path`（回答ファイルパス）
       - `status`（ステータス）
       - `reception_time`（受付日時）

  1. sequences（シーケンス管理テーブル）
     - `name`（テーブル名）をパーティションキーとし、以下の属性を持つ。
       - `current_number`（現在値）
         - アトミックカウンタとして扱うことで RDB のシーケンスと同様に重複しない連番を採番する。

- Lambda の IAM ロールに DynamoDB のアクセス権限を追加する。

### アプリケーションの設定

- `pom.xml`に以下の依存を追加する。
  ```
  <dependency>
    <groupId>com.github.derjust</groupId>
    <artifactId>spring-data-dynamodb</artifactId>
  </dependency>
  ```
- DynamoDbConfig を作成

  - DynamoDB 接続用の Config クラスを作成する。

  ```java
  @Configuration
  @EnableDynamoDBRepositories(basePackages = "org.debugroom.mynavi.sample.spring.data.dynamodb.domain.repository")
  public class DynamoDBConfig {

    @Value("${amazon.dynamodb.region}")
    private String region;
    @Value("${amazon.dynamodb.endpoint}")
    private String endpoint;

    @Bean
    public AmazonDynamoDB amazonDynamoDB(){
        return AmazonDynamoDBClientBuilder.standard().withEndpointConfiguration(
             new AwsClientBuilder.EndpointConfiguration(endpoint, region)).build();
    }
  }
  ```

- application.yml の設定
  - `application.yml`に DynamoDB への接続情報を設定する。
  ```yaml
  amazon:
    dynamodb:
      region: ap-northeast-1
      endpoint: https://dynamodb.ap-northeast-1.amazonaws.com
  ```
-

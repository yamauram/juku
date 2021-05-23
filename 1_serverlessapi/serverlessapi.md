# サーバレス API の構築

### 背景・目的

- 以下の連載記事を参考に API Gateway ＋ Lambda のオンライン AP の構築を検証する。
  - `API GatewayとLambdaを使ったサーバレスSpringアプリケーション`  
    https://news.mynavi.jp/itsearch/article/devsoft/4316
- 構築の目的は、現在業務で構築中のオンライン AP（ECS on EC2） について、サーバレスに移行することが可能かどうか検証したいため。
  - お客様から可能な限りコストを抑えるためにサーバレスにできないか、と要望あり。
  - また、現状バッチ処理は ECS on Fargate を採用しており、オンライン AP もサーバレスにすることができれば、EC2 の管理を不要にできるため。
- 対象は、大量データのディレード処理を受け付けるオンライン AP（以下の 2 の部分）。
  - 大まかな仕様（処理の流れ）は以下の通り。
    1. 要求元システムが S3 バケットに入力ファイルを格納し、S3 のパスをオンライン AP に通知する。
    1. オンライン AP は DB から受付番号を採番し、後続のディレード処理起動のために SQS にメッセージを登録後、ユーザに採番した受付番号を返却する。
    1. 後続のディレード処理の完了後、結果ファイルを S3 のパスに出力する。
    1. 要求元システムからの結果取得の要求（受付番号がキー）に対し、S3 のオブジェクトパスを返却する。

### 検証構成

- 別紙の構成とする。
- 検証の段階として、以下の 5 段階で検証する。
  1. APIGateway + Lambda  
     ⇒ とりあえず何かを返す
  1. APIGateway + Lambda + `SQS`  
     ⇒SQS にメッセージ送信したうえで何かを返す
  1. APIGateway + Lambda + SQS + `Aurora`  
     ⇒ 受付けた情報を DB に登録し、SQS にメッセージ送信したうえで、DB から採番した受付番号を返す
  1. APIGateway + Lambda + SQS + `RDS Proxy` + Aurora  
     ⇒（4 と同じだが）RDS Proxy を介して、DB にアクセスする
  1. APIGateway + Lambda + SQS + RDS Proxy + Aurora + `S3`  
     ⇒（5 までの処理に追加で）S3 へアクセスしてファイルチェック等の処理を行う

### 検証結果

- <u>検証 1：APIGateway + Lambda でとりあえず何かを返してみる</u>

  - （最初自力でやってみたものの上手くいかなかったので…）基本的に連載記事通りに実施。※Java のバージョンを JDK1.8 ではなく AmazonCorretto11 に変更
  - 四苦八苦するも無事疎通完了。
  - リソースに関係なくコールドスタートがめちゃくちゃ遅い

    | Lambda 割り当てメモリ | 1 回目レイテンシー | 2 回目レイテンシー |
    | --------------------- | ------------------ | ------------------ |
    | 128MB※下限            | 29024ms            | 29005ms            |
    | 1024MB                | 8380ms             | 41ms               |
    | 4096MB                | 3776ms             | 43ms               |
    | 10240MB※上限          | 3492ms             | 45ms               |

  - 以下、Lambda 実行時間について
    - Lambda の実行時には以下の順で関数が実行されるらしい。
      - https://qiita.com/ny7760/items/700ae917da2c5b5e3f8a
      1.  ENI の作成
      1.  コンテナの作成
      1.  デプロイパッケージのロード
      1.  デプロイパッケージの展開
      1.  ランタイム起動・初期化
      1.  関数・メソッドの実行
    - 初回実行時には 1 から順に実行され、時間がかかる（コールドスタート）。
    - 一定時間（時間不明？）内に Lambda が再実行された場合は前回起動時のコンテナが再利用されるため、6 からの実行となり起動が高速になる（ウォームスタート）。
    - 商用での利用を検討する場合コールドスタート対処はマスト？
    - 考えられる対処としては以下の 3 つ
      - コンテナイメージから Lambda を起動した方が早いらしい？
        - https://dev.classmethod.jp/articles/measure-container-image-lambda-coldstart/
      - Provisioned Concurrency
        - 2019 年の re:Invent で発表されたサービスで、Lambda を事前にプロヴィジョニングしておくサービスとのこと（当然お金がかかる）。
          - https://aws.amazon.com/jp/lambda/pricing/
      - Lambda を定期実行する
        - 「Provisioned Concurrency」の登場まではこれしか選択肢がなかったとのこと。
        - コールドスタートの確率を下げることはできるが、ウォームスタートを担保できるわけではない。

- <u>検証 2：SQS にメッセージを送信してみる</u>

  - クライアントへ返却する内容を、連載記事通りの`「Completed!」`から以下の通り JSON に修正する。

  ```shell
  {
  "receptionNumber": 1
  }
  ```

  - SQS を作成し、SQS の接続、クライアント設定等を行う Config を作成。
  - Bean 定義が上手くいっていないよう（`Error creating bean with name～`）でエラー発生、解析中。

  ***

  （以降、4/26 の報告以降の作業報告）

  - 以下の連載記事を参考に、`QueueMessagingTemplate`を使用して SQS へのメッセージ送信を実装。

    - `Amazon SQS を使った Spring アプリケーション`  
      https://news.mynavi.jp/itsearch/article/devsoft/4656

  - Lambda で IAM ロールから認証情報を取得するために、以下のプロパティを application.yml に設定。

    ```yaml
    cloud:
      aws:
        credentials:
          useDefaultAwsCredentialsChain: true
    ```

  - SQS へのキューにメッセージが送信されていることを確認。
    - リクエスト本文
      > {"requestFilePath":"s3://testBucket/testData.xml"}
    - レスポンス本文　※receptionNumber の値はこの時点では固定値
      > {
      > "receptionNumber": 1
      > }

- <u>検証 3：RDS に接続してみる</u>

  - receptionNumber（受付番号） を DB から取得する設定を行う。
  - Lambda が VPC リソースにアクセスできるよう Lambda の設定を行う。https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/configuration-vpc.html  
    また、Lambda の IAM ロールに以下の権限を追加する。
    ```json
    ec2:CreateNetworkInterface
    ec2:DescribeNetworkInterfaces
    ec2:DeleteNetworkInterface
    ```
  - VPC Lambda 設定後、これまでアクセスできていた SQS へアクセスできなくなった。  
     Lambda を作成した VPC の`private subnet`から`public subnet`に作成した NATGW 経由でアクセスできるようルートテーブルを設定したところ解消した。  
     ※本来は VPC エンドポイント経由、が理想。
  - RDS（PostgreSQL）を作成し、MyBatis（自担当のプロジェクトでいようしているため） で DB アクセスする。

    - DB のセットアップは AWS Cloud9 で行う。

      ```sql
      # postgresqlのインストール
      sudo yum install -y postgresql

      # パスワード設定
      export PGPASSWORD=<パスワード>

      # PostgreSQLへ接続
      psql -h <RDSのエンドポイント> -p 5432 -U postgres -d postgres

      # テーブル作成
      # キーとなるRECEPTION_NUMBERはSERIAL型とし、MyBatisのgeneratedKeyでキー生成する仕組み
      create table request_info(
        reception_number serial,
        request_file_path varchar(200),
        response_file_path varchar(200),
        status varchar(20),
        reception_time timestamp
      );
      ```

    - generatedKey で生成した Key を取得し、クライアントに返却する。

      - com/juku/serverless/domain/common/repository/RequestInfoRepository.java
        ```java
        @Mapper
        public interface RequestInfoRepository {
          public void create(RequestInfo requestInfo);
        }
        ```
      - com/juku/serverless/domain/common/repository/RequestInfoRepository.xml
        ```xml
        <mapper namespace="com.juku.serverless.domain.common.repository.RequestInfoRepository">
        <insert id="create" useGeneratedKeys="true" keyProperty="receptionNumber" parameterType="com.juku.serverless.domain.common.model.RequestInfo">
          INSERT INTO request_info (
            request_file_path,
            status,
            reception_time)
          values(
            #{requestFilePath},
            #{status},
            #{receptionTime}
          )
        </insert>
        </mapper>
        ```
      - com/juku/serverless/domain/common/service/ReceptionServiceImpl.java

        ```java
        @Service
        @Transactional
        public class ReceptionServiceImpl implements ReceptionService {

          @Value(value = "${reception.queue.name:reception-queue}")
        	String queueName;

        	@Autowired
        	QueueMessagingTemplate queueMessagingTemplate;

        	@Autowired
        	RequestInfoRepository repository;

        	@Override
        	public ResponseDto accept(String requestFilePath) {

           	// 受付情報をDBに登録（ステータスは01）
          	RequestInfo requestInfo = RequestInfo.builder().requestFilePath(requestFilePath).status("01")
          			.receptionTime(LocalDateTime.now()).build();
          	repository.create(requestInfo);

          	// キューにメッセージを登録（受付番号）
          	ReceptionDto receptionDto = ReceptionDto.builder().requestFilePath(requestFilePath)
          			.receptionNumber(requestInfo.getReceptionNumber()).build();
        	queueMessagingTemplate.convertAndSend(queueName, receptionDto);

          	// レスポンス情報を返却
          	ResponseDto responseDto = ResponseDto.builder().receptionNumber(requestInfo.getReceptionNumber()).build();
          	return responseDto;
        	}
        }
        ```

    - DB
      ```sql
      postgres=> select * from request_info;
       reception_number |      request_file_path       | response_file_path | status |       reception_time
      ------------------+------------------------------+--------------------+--------+----------------------------
                      6 | s3://testBucket/testData.xml |                    | 01     | 2021-05-02 04:57:02.355337
                      7 | s3://testBucket/testData.xml |                    | 01     | 2021-05-02 04:57:10.295876
                      8 | s3://testBucket/testData.xml |                    | 01     | 2021-05-02 04:57:11.689139
                      9 | s3://testBucket/testData.xml |                    | 01     | 2021-05-02 04:57:12.727186
      (6 rows)
      ```

- <u>検証 3.5：大量のリクエストを送ってみる</u>

  - 現状単純な`Lambda＋RDSしただけのアンチパターン構成`であり、この状態で大量のリクエストを API に送ってみる。  
    想定通りであれば、コネクション枯渇で途中からエラーになるはず。
  - Cloud9 から以下の通り API のエンドポイントをたたく。負荷掛けは JMeter で行う。  
    JMeter を Cloud9 にインストールする。

    ```bash
    # JavaはCloud9デフォルトのAmazonCorrettoを使用。

    # optへ移動
    cd /opt/

    # Jmeterのバイナリを取得
    sudo wget http://ftp.meisei-u.ac.jp/mirror/apache/dist//jmeter/binaries/apache-jmeter-5.4.1.tgz

    # 展開
    sudo tar xvzf apache-jmeter-5.4.1.tgz

    # PATH設定
    vi ~/.bashrc
    ###
    export PATH=$PATH:$HOME/bin:/opt/apache-jmeter-5.4.1/bin
    ###

    # 確認
    jmeter --version
    ```

  - ここまでは API Gateway のテストから検証を行ったが、curl でのアクセスを行うために API のデプロイを行う。  
    デプロイのステージは`「prod」`とする。発行された API の URL は以下の通り。
    ```bash
    https://<APIのエンドポイント>/prod
    ```
  - 以下のコマンドで API にリクエストを送る。

    ```bash
    jmeter -n -t serverless.jmx -l result.jtl
    ```

    - Jmeter 設定（serverless.jmx）
      - スレッド数：100
      - Ramp-Up 期間（秒）：1
      - ループ回数：100
    - 単発の場合
      ```bash
      curl -X POST  https://<APIのエンドポイント>/prod/sample -H "Content-Type: application/json" -d '{"requestFilePath":"s3://<バケット名>/test_data.xml"}'
      ```

  - 結果：エラーが発生することなくさばけてしまった。。。

- <u>検証 4：RDS に RDS Proxy 経由で接続してみる</u>

  - RDS Proxy をコンソールから作成する（結構時間がかかる…）。  
    ※当初 PostgreSQL のバージョンを 13.2 で作成していたが、RDS Proxy が対応していなかったため、RDS 自体をバージョン`11.5`で再作成した。注意。
  - DB のエンドポイントを作成した RDS Proxy のエンドポイントで置き換える。
    ```yaml
    spring:
      datasource:
        url: jdbc:postgresql://<RDS Proxyのエンドポイント>:5432/postgres
    ```
  - proxy 経由で接続を確認。  
    ただし、JMeter で大量リクエストを投げると詰まってしまった。要検証。

- <u>検証 5：S3 にアクセスして入力ファイルを検証してみる（実アプリと同様の仕様にしてみる）</u>

  - S3 バケットを作成してテストデータ（XML ファイル）を配置。
  - `ResourceLoader`を使用し、S3 バケットにあるファイルの存在確認チェックを行う。

    ```java
    @Autowired
    private ResourceLoader resourceLoader;

    /* 略 */

    @Override
    public ResponseDto accept(String requestFilePath) {

      // S3ファイルパスの存在確認
      Resource resource = resourceLoader.getResource(requestFilePath);
      if(resource.exists()) {
    	  log.info("要求ファイルは存在します。:{}", requestFilePath);
      }else {
    	  log.error("要求ファイルは存在しません。:{}", requestFilePath);
      }

      /* 略 */
    ```

  - 無事正常終了を確認。

- <u>検証 6：Provisioned Concurrency を試してみる</u>

  - 以下の条件で Provisioned Concurrency の有無によるレイテンシー分布を調査。

    - Jmeter 設定（serverless.jmx）
      - スレッド数：100
      - Ramp-Up 期間（秒）：1
      - ループ回数：100

    1. `Provisioned Concurrencyなし`
       - 10000 リクエスト中、101 リクエストがレイテンシー 1000ms を超過
         - 非超過：271ms、超過 Avg:15336ms
    1. `Provisioned Concurrencyあり`

       - Provisioned Concurrency は「エイリアス」または「特定のバージョン」指定が必要なため（＝$Latest は不可）、Lambda 関数にバージョンを設定

         - リージョンごとのデフォルトの上限数は 1000
         - 料金は以下参照。  
           https://aws.amazon.com/jp/lambda/pricing/
           - 設定時には以下のようなサジェストがあった（同時実行数 100）。
             > 期間とリクエストの料金に加えて、月額 $5,768.20 です。

       - API Gateway から呼び出す Lambda 関数のバージョンを以下の通り指定。

         > serverless-application`:1`

         - 本来は、Lambda 側のバージョンとエイリアスを組み合わせ、API Gateway のデプロイステージ（`${stageVariables.alias}`） > Lambda のエイリアス > Lambda のバージョンという流れで、API の URI からバージョンを特定するのがセオリー（以下参照）  
           https://dev.classmethod.jp/articles/version-management-with-api-gateway-and-lambda/  
           https://qiita.com/leomaro7/items/86ae586d278265720429

       - 結果、Provisioned Concurrency 無しの場合と変わらず、10000 リクエスト中 100 リクエストがレイテンシー 1000ms を超過。  
         ※1 リクエスト分は誤差と思われる。
       - ログを確認したところ、Provisioned Concurrency によって事前のデプロイは行われているものの、リクエスト発生後に SpringBoot 起動時の処理で 10 ～ 20 秒程度の時間がかかっており、Provisioned Concurrency の効果があまりなかった。
       - Provisioned Concurrency ではなく CloudWatchEvents 等で事前に暖機運転しておくか、Spring Cloud Function ではなく通常の SpringMVC による RestAPI とするか検討（この場合もはや Fargate でよいのでは…）。
       - そもそも Spring を無理して使うのではなく、JavaScript やネイティブ Java、軽量なフレームワーク（Micronaut など）、GraalVM（現在 Spring ではベータ版）の利用なども検討してみる。

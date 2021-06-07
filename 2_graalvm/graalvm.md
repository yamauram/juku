# GraalVM による Spring アプリケーションの Native ビルド検証

## 背景・目的

- サーバレス API の検証で API Gateway + Lambda による構成での疎通はできたが、コールドスタートにより初回起動時のレスポンスに多大な時間がかかることがわかった。
- コールドスタート問題に対処する方法はいくつかあるが、先日ベータ版が発表された GraalVM の Spring 対応により、アプリケーションの起動時間がどれほど早くなるか計測し、商用環境へ適用可能か検証する。
- 使用するアプリケーション自体はサーバレス API の検証の中で使用したものと同様のものを使用する。

## 環境構築

- 以下のページから GraalVM をダウンロード、解凍する。  
  https://www.graalvm.org/downloads/
  - JDK バージョンは 11 を採用。
  - GraalVM には Community Edition（CE）と Enterprise Edition（EE）の 2 つがあるが、今回は CE を採用。
- PATH、JAVA_HOME の設定

  ```sh
  C:\Users\MASAKI>set PATH=%PATH%;C:\graalvm-ce-java11-21.1.0\bin

  C:\Users\MASAKI>set JAVA_HOME=C:\graalvm-ce-java11-21.1.0
  ```

- バージョン確認
  ```sh
  C:\Users\MASAKI>java --version
  openjdk 11.0.11 2021-04-20
  OpenJDK Runtime Environment GraalVM CE 21.1.0 (build 11.0.11+8-jvmci-21.1-b05)
  OpenJDK 64-Bit Server VM GraalVM CE 21.1.0 (build 11.0.11+8-jvmci-21.1-b05, mixed mode, sharing)
  ```
- Native Image コンポーネントインストール
  - GraalVM で Java プログラムを NativeImage 化するためのコンポーネントをインストールする。
  - Native Image コンポーネントは GraalVM をインストールしたフォルダの bin 以下にある gu 経由でインストールする（gu ってのは GraalVM Updater の事らしい）。
  ```sh
  C:\Users\MASAKI>gu install native-image
  Downloading: Component catalog from www.graalvm.org
  Processing Component: Native Image
  Downloading: Component native-image: Native Image  from github.com
  Installing new component: Native Image (org.graalvm.native-image, version 21.1.0)
  ```

## Native イメージビルド

- `native-image`コマンドによりビルドするも、失敗する。
  ```
  C:\Users\MASAKI\git\graalvm\target>native-image -jar C:\Users\MASAKI\git\graalvm\target\serverless-0.0.1-SNAPSHOT-aws.jar
  [serverless-0.0.1-SNAPSHOT-aws:9028]    classlist:   3,839.16 ms,  0.96 GB
  [serverless-0.0.1-SNAPSHOT-aws:9028]        setup:     929.00 ms,  0.96 GB
  Fatal error:java.nio.file.InvalidPathException: Illegal char <"> at index 0: "C:\Users\MASAKI\AppData\Local\Microsoft\WindowsApps\cl.exe
        at java.base/sun.nio.fs.WindowsPathParser.normalize(WindowsPathParser.java:182)
        at java.base/sun.nio.fs.WindowsPathParser.parse(WindowsPathParser.java:153)
        at java.base/sun.nio.fs.WindowsPathParser.parse(WindowsPathParser.java:77)
        at java.base/sun.nio.fs.WindowsPath.parse(WindowsPath.java:92)
        at java.base/sun.nio.fs.WindowsFileSystem.getPath(WindowsFileSystem.java:229)
        at java.base/java.nio.file.Path.of(Path.java:147)
        at java.base/java.nio.file.Paths.get(Paths.java:69)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.lambda$lookupSearchPath$1(CCompilerInvoker.java:485)
        at java.base/java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:195)
        at java.base/java.util.Spliterators$ArraySpliterator.tryAdvance(Spliterators.java:958)
        at java.base/java.util.stream.ReferencePipeline.forEachWithCancel(ReferencePipeline.java:127)
        at java.base/java.util.stream.AbstractPipeline.copyIntoWithCancel(AbstractPipeline.java:502)
        at java.base/java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:488)
        at java.base/java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:474)
        at java.base/java.util.stream.FindOps$FindOp.evaluateSequential(FindOps.java:150)
        at java.base/java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
        at java.base/java.util.stream.ReferencePipeline.findFirst(ReferencePipeline.java:543)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.lookupSearchPath(CCompilerInvoker.java:487)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.getCCompilerPath(CCompilerInvoker.java:497)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.getCCompilerInfo(CCompilerInvoker.java:345)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.<init>(CCompilerInvoker.java:68)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker$WindowsCCompilerInvoker.<init>(CCompilerInvoker.java:108)
        at com.oracle.svm.hosted.c.codegen.CCompilerInvoker.create(CCompilerInvoker.java:82)
        at com.oracle.svm.hosted.NativeImageGenerator.setupNativeImage(NativeImageGenerator.java:902)
        at com.oracle.svm.hosted.NativeImageGenerator.doRun(NativeImageGenerator.java:580)
        at com.oracle.svm.hosted.NativeImageGenerator.lambda$run$2(NativeImageGenerator.java:495)
        at java.base/java.util.concurrent.ForkJoinTask$AdaptedRunnableAction.exec(ForkJoinTask.java:1407)
        at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
        at java.base/java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
        at java.base/java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
        at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
        at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:183)
  Error: Image build request failed with exit status 1
  ```

```

```

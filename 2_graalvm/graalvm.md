# GraalVM による Spring アプリケーションの Native ビルド検証

## 背景・目的

- サーバレス API の検証で API Gateway + Lambda による構成での疎通はできたが、コールドスタートにより初回起動時のレスポンスに多大な時間がかかることがわかった。
- コールドスタート問題に対処する方法はいくつかあるが、先日ベータ版が発表された GraalVM の Spring 対応により、アプリケーションの起動時間がどれほど早くなるか計測し、商用環境へ適用可能か検証する。
- 使用するアプリケーション自体はサーバレス API の検証の中で使用したものと同様のものを使用する。
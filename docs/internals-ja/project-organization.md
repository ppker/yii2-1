プロジェクトの編成
==================

このドキュメントは Yii2 開発レポジトリの編成を説明するものです。
 
1. 個々のコアエクステンションとアプリケーションテンプレートは、[yiisoft](https://github.com/yiisoft) Github オーガニゼーションの下の *独立した* 別の Github プロジェクトとして保守されます。
    
   エクステンションのプロジェクト名は、先頭に `yii2-` を付けます。例えば、`gii` エクステンションは `yii2-gii` です。
   Composer のパッケージ名は Github レポジトリ名と同じで、例えば `yiisoft/yii2-gii` です。
   
   アプリケーションテンプレートのプロジェクト名は、先頭に `yii2-app-` を付けます。例えば、`basic` アプリケーションテンプレートは `yii2-app-basici` です。
   Composer のパッケージ名は Github レポジトリ名と同じで、例えば `yiisoft/yii2-app-basic` です。
   
   各々のエクステンション/アプリケーションのプロジェクトは、
 
   * "docs" フォルダにおいてそのチュートリアルドキュメントを保守します。API ドキュメントは、エクステンション/アプリケーションがリリースされるときにその場で生成されます。
   * "tests" フォルダにおいてそれ自身のテストコードを保守します。
   * それ自身のメッセージ翻訳やその他全ての関係するメタコードを保守します。
   * 対応する Github プロジェクトによって、課題 (issue) を追跡します。
      
   エクステンションのレポジトリは、必要に応じて、個別にリリースされます。アプリケーションテンプレートはフレームワークとともにリリースされます。
   詳細は [バージョンポリシー](versions.md) を参照して下さい。

2. `yiisoft/yii2` プロジェクトが、Yii2 フレームワーク開発のためのメインレポジトリです。
   このレポジトリは Composer パッケージ [yiisoft/yii2-dev](https://packagist.org/packages/yiisoft/yii2-dev) を提供します。
   これは、コアフレームワークコード、フレームワークの単体テスト、決定版ガイド、そして、フレームワーク開発とリリースのための一組のビルドツールを含んでいます。
   
   コアフレームワークのバグと機能要望は、この Github プロジェクトのイッシュートラッカーによって追跡されます。
   
3. `yiisoft/yii2-framework` レポジトリは、開発プロジェクトレポジトリの `framework` ディレクトリのリードオンリーな git subsplit です。
   このレポジトリが、フレームワークのインストールに使用される Composer 公式パッケージである [yiisoft/yii2](https://packagist.org/packages/yiisoft/yii2) を提供します。

4. 開発するときには、[build dev/app](git-workflow.md#prepare-the-test-environment) コマンドを使って、アプリケーションとエクステンションを開発プロジェクトの構成に含めることが出来ます。
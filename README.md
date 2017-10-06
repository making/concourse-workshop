# Concourse Workshop

!!!!WIP!!!!

コンテンツの要望があればissueにあげてください

Concourse 3.3以上を想定しています。

次のようなCI/CDパイプラインを構築できるように、Concourseの基本をステップバイステップで学びます。

![image](https://user-images.githubusercontent.com/106908/29496945-ecd53dc8-8618-11e7-8329-e2947aa23f47.png)


## 目次

* [事前準備](prerequisite.md)
* [初めてのパイプライン](first-pipeline.md)
* [Javaアプリケーションのユニットテスト](java-unit-test.md)
* [キャッシュの導入](caching.md)
* [Javaアプリケーションのデプロイ](java-deploy.md)
* [パッケージマネージャNexusの導入](nexus.md)
* [semverリソースでのバージョンの管理](semver-resource.md)
* [developブランチの管理](develop-branch.md)
* [mavenリソースでのパイプラインのシンプル化](maven-resource.md)
* Pull Requestに対するテスト
* Slackへの通知
* WebHook
* Docker in Dockerを利用したインテグレーションテスト
* `fly hijack`でデバッグ
* Taskのファイル分割
* VaultによるCredentialsの管理


## 利用規約

無断で本ドキュメントの一部または全部を改変したり、本ドキュメントを用いた二次的著作物を作成することを禁止します。ただし、ドキュメント修正のためのPull Requestは大歓迎です。

----
© 2017 Pivotal Japan

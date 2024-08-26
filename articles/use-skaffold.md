---
title: "Skaffoldを使って複数のCloudRunサービスをデプロイする"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [CloudRun,Skaffold]
published: true
publication_name: socialdog
---

こんにちは！
SocialDogで主にバックエンド、インフラを担当している[muratty](https://x.com/canon1ky)です。
少し前にデプロイプロセスの改善の一環として**Skaffold**を導入しました。この記事では、その導入過程と得られた利点について詳しく説明します。

## この記事で書くこと
本記事では以下の内容を取り上げます。

- Skaffold導入前のCloudRunの構成
- リリースにおける課題と目標
- 目標達成のために必要な施策
- Skaffold導入の詳細と効果

## Skaffoldとは
Skaffoldは、Kubernetes向けのコンテナベースやクラウドネイティブなアプリケーションの継続的なデプロイを容易にするオープンソースのプラットフォームです。

https://skaffold.dev/

Skaffoldは**CloudRun**にも使用することができます。

### Skaffold導入のメリット
様々なメリットがありますが、SocialDogにおいては主に以下のメリットがあると考えました。
- **複数のサービスを一括でデプロイできる**
  - `skaffold run` コマンドでデプロイが可能
- **各サービスの個別のデプロイを1つの設定ファイルで管理できる**
  - skaffold.yamlというファイルに、デプロイ対象のCloudRunサービスやGCPプロジェクトを記述する
- **デプロイだけでなくビルドプロセスも統合可能**
  - ビルドはCloudBuildを使うためcloudbuild.yamlというファイルで管理しているが、 デプロイ設定と合わせてskaffold.yamlにまとめることができる
- **各種CI/CDツールとの容易な統合**
  - 仮にCloudDeployを使わない方針になったとしても、Skaffoldをインストールして `skaffold run` コマンドを叩けば、CI/CDツール上でデプロイが可能
  - CI/CD用のファイル毎に複数の `gcloud run replace` コマンドを漏れなく記述するような気を使う必要がない

## Skaffold導入前のCloudRunの構成
SocialDogのCloudRunサービスには主にWebサーバーとバッチサーバー用のサービスがそれぞれ存在します。

そして、アプリケーションの開発リポジトリには、以下のCloudRunの設定ファイル^[アプリケーションの開発リポジトリに設定ファイルを置いている理由は、基本的には環境変数を用意するのはアプリケーション開発者であり、機能のレビューの一環として環境変数のレビューも行うような体制としたかったためです]が存在します。

- go-web.yaml
  - Webサーバー用の設定
  - CloudRunで使用するコンテナのスペックや、環境変数が記載されている
- go-batch.yaml
  - バッチサーバー用の設定
- その他用途別のバッチサーバーのyamlファイルがいくつか

これらのyamlに対して、リリース担当者がリリース時に以下のような複数のコマンドを叩いていました。

```
$ gcloud run replace go-web.yaml --project [本番のGCPプロジェクト]
$ gcloud run replace go-batch.yaml --project [本番のGCPプロジェクト]
# ... その他のCloudRunサービスも同様にコマンドを叩く
```

なお、これ以外にも別のタイミングでコンテナイメージのビルドが走っていたり、別のリリース対象物がありますが、それらの説明は省略させていただきます。

## 課題と目標
SocialDogのリリースに関する今後の方針の中に、**ユーザー体験の迅速な改善のため、リリース頻度を上げる**というものがあります。
そのアプローチの1つとして、リリース自体の運用コストを削減するために、ローカルPCから複雑な手順のリリースを行う形ではなく[CloudDeploy](https://cloud.google.com/deploy)を使ってGUIでサッとリリースできる構成を展望としています。

なお、その過程で以下のような課題があります。
- CloudDeployではSkaffoldを使用する^[関連リンク: https://cloud.google.com/deploy/docs/using-skaffold?hl=ja]ため、CloudDeploy導入時にSkaffoldを使用してデプロイできるようにしておく必要がある
- 将来的にCloud Run Jobsを多用する可能性があり、Jobを追加するたびにリリース時の `gcloud run replace` コマンドを増やすといった運用コストの増加

これらに対応するため、まずはローカルにSkaffoldを導入し、デプロイできる状態を作ることから始めることにしました。

## Skaffold導入後の構成、デプロイ手順
Skaffold導入後、以下のような変更を行いました。

- アプリケーションリポジトリにskaffold.yamlを配置
- 各設定ファイルをresourcesディレクトリにまとめる

### 具体的なディレクトリ構成
具体的なディレクトリ構成としては以下のような構成です。

```
- アプリケーションリポジトリ
  - application_go (アプリケーションのコード)
  - cloud_run
    - prod
      - skaffold.yaml
      - resources
        - go-web.yaml
        - go-batch.yaml
        ...
    - staging
      - skaffold.yaml
      - resources
        - go-web.yaml
        - go-batch.yaml
        ...
```

### skaffold.yaml
skaffold.yamlには以下のような中身を記載しました。

```yaml
apiVersion: skaffold/v4beta2
kind: Config

manifests:
  rawYaml:
    - resources/*.yaml

deploy:
  cloudrun:
    projectid: [GCPプロジェクト]
    region: asia-northeast1
```

結果として、以下のような状態となりました。

- **skaffold.yamlの中身を見れば、「resourcesディレクトリ以下のyamlファイルが、記載されているGCPにデプロイされること」が一目でわかる**
- **skaffold.yamlと同じディレクトリ内で `skaffold run` コマンドを叩くと、skaffold.yamlに記載の構成に従ってデプロイされる**

## 今後の展望
Skaffoldの導入によって、デプロイ構成の一元管理や、単体のコマンドでの複数のCloudRunサービスのリリースが可能となりました。
また、今後のCloudDeployの導入に近づいた上、CloudRun Jobsを追加する際にも低コストで運用できる見込みとなりました。

今回紹介していなかった課題として、Skaffoldとは別にgo-web.yamlなどのyamlファイルを独自のテンプレート機構を使って生成しているという課題もあります。
それらもSkaffoldと統合できる[helm](https://helm.sh/ja/)や[kustomize](https://kustomize.io/)^[一般的にはkubernetesと合わせて使うことが多いと思いますが、SkaffoldにおいてはCloudRunでも使用できるのではないかと考えています]を使って生成できるようにしていきたいと考えています。

今後も継続的な改善を進め、より迅速かつ安全なサービス提供を目指します。

## SocialDogについて

株式会社SocialDogは、「あらゆる人がSNSを活用できる世界を創る」をミッションとし、SNSマーケティング運用担当者のためのオールインワンツールを提供しています。

https://portal.socialdog.jp/

多くのユーザーが使うサービスを支える取り組みとして、今後もリリース周りやインフラ、セキュリティ周りのさらなる強化を行っていきたいと考えています。
エンジニア募集中ですので、興味を持ってくださった方は是非お話ししましょう！
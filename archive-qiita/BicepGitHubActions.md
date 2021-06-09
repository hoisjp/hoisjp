# Intro
Azure には、**Bicep** という Infrastructure as Code (IaC) の DSL があります。（カタカナ読み方は "バイセップ"、日本語訳は "上腕二頭筋"、ARM に対して・・・という由来だと思われます）ARM テンプレートの後継として登場してきました。2021/06 時点では v0.4 プレビュー版です。どんどん機能が追加されているところですが、中心となる機能群はすでに揃っていて、充分使い始めることができるレベルかと思います。

- [Azure Docs](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
- [GitHub Repository](https://github.com/Azure/bicep)

この記事は、その **Bicep で記述した設定ファイルを GitHub Actions で自動的に Azure に反映してみる**、という内容です。

## Why Bicep & GitHub Actions?

従来の ARM テンプレートは、JSON 形式ということに加え、どうしてもリソース定義以外の本質的ではない記述が多かったです。つまり、可読性が良くなかった、ということです。これは、担当者が定義ファイルを書く、誰かにプルリクエストをする、レビュアーは間違っていないかじっくり読む、といった、**頻繁に繰り返される 書く＆読む プロセスにおいて、致命的なネックになります。**レビュアーも「正しいかどうかは実行してみないとわからん」「どこ見ていいかわからん」状態になってしまうと、コードレビューの本来の役割が機能しません。[BicepとARM Templateのシンタックスの比較](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/compare-template-syntax)にもあるように、記述はシンプルになり、**レビューするほうもこれでやっとまともな指摘ができるようになった感**があります。
ということで、GitHub Actions を使って以下のプロセスをぐるぐる高速にまわす準備をさくっと済ませてしまいましょう。

- リソース定義をコードで記述する（IaC）
- プルリクエストを中心としたレビューを行う
- マージされたら環境への反映は自動的に行われる

コードを見たほうが早い、というかたは以下リポジトリに Template リポジトリを用意したので、あわせてご参考ください。"Use this template" していただければ、ご自身のリポジトリでもすぐに試せます。
https://github.com/hoisjp/vscode-remote-try-bicep

Bicep 用 devcontainer リポジトリ (https://github.com/Azure/vscode-remote-try-bicep) からフォークして、GitHub Actions のサンプルを追加しています。

# 手順

## テンプレートからご自身のリポジトリへ
https://github.com/hoisjp/vscode-remote-try-bicep
`Use this template` して、ご自身のリポジトリにひな型を作ります。Private リポジトリでもOKです。

利用手段は2種類あります。

1. **ローカルの Docker 環境で VS Code リモート開発機能を使う**  
VS Code のリモートコンテナ機能についての説明は割愛します。[このあたり](https://code.visualstudio.com/docs/remote/containers) のドキュメントをご参考ください。必要な準備は、ローカルのDocker環境と、[VS Codeの拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) だけです。  
2. **GitHub Codespaces を使う**  
まだ Private Beta 状態ですが、すでに使えるかたは、リポジトリから Codespaces を起動した環境でも可能です。
https://docs.github.com/en/codespaces/about-codespaces

いずれも Bicep 環境はコンテナビルド直後に仕上がっています。便利ですね。**複数人で使っていても全く同じ環境で作業ができる**わけです。

```sh
$ bicep -v
Bicep CLI version 0.4.63 (7ebed03284)

$ az version
{
  "azure-cli": "2.24.2",
  "azure-cli-core": "2.24.2",
  "azure-cli-telemetry": "1.0.6",
  "extensions": {}
}
```
## GitHub Actions 有効化
外部のリポジトリから fork した場合や、template 生成した場合は、GitHub Actions が無効になっています。悪意のある Actions が自身のリポジトリで実行されてしまうのを回避するため、セーフティガードが働いています。

https://docs.github.com/ja/actions/managing-workflow-runs/disabling-and-enabling-a-workflow

GitHub リポジトリの Actions タブメニューから、`Enable workflow` ボタンで有効化します。

## 準備 GitHub -> Azure の認証
GitHub Actions を実行する GitHub リポジトリを用意したらば、そのリポジトリが Azure に対して操作する権限を与える必要があります。Azure サービスプリンシパルを GitHub のシークレットに追加します。間接的な準備ですが、[Azure 公式ドキュメントの接続手順](https://docs.microsoft.com/ja-jp/azure/developer/github/connect-from-azure)に従って進めてOKです。少し補足しながら以下に同じことを書きます。

__Step 1. サービスプリンシパルを作ります。__
Cloud Shell が手軽でしょう。Azure ポータルの右上部分のシェルアイコンから開くか、https://shell.azure.com をブラウザで開いてもよいです。  
まずはサービスプリンシパルを作るアプリケーションの作成から始めます。

```sh
appName="myApp"

az ad app create \
    --display-name $appName \
    --homepage "http://localhost/$appName" \
    --identifier-uris http://localhost/$appName
```
この `appName="myApp"` は、組織全体の Azure AD に登録されることになるので、区別されやすい名前を与えるのがよいでしょう。またこのサービスプリンシパルは、GitHub Actions から利用するためだけに使われて、あちらこちらで流用されることはないと思うので、そんなキーワードを入れておくとよさそうです。 e.g. `SampleAppGitHubActions`

__Step 2. 新規サービスプリンシパル作成__
公式ドキュメントでは以下のように、このサービスプリンシパルが有効なスコープを、特定のリソースグループに限定しています。

`--scopes` スコープ：リソースグループ

```sh
az ad sp create-for-rbac --name $appName --role contributor \
    --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group} \
    --sdk-auth
```
リソースグループをひとつに限定できる場合は上記でも構いませんが、**複数のリソースグループを使って構成する場合も多いです。**そのような場合は、以下のように**スコープをサブスクリプションに広げる**ことで、そのサブスクリプションの下ならば操作ができるようになります。

`--scopes` スコープ：サブスクリプション

```sh
az ad sp create-for-rbac --name $appName --role contributor \
    --scopes /subscriptions/{subscription-id}/ \
    --sdk-auth
```
このコマンドの結果JSONを後続のシークレットに使います。

__Step 3. GitHub シークレット追加__
GitHub リポジトリメニューの `Setting` から、左メニュー `Secrets` を開きます__
右上の `New repository secret` ボタンからシークレット登録画面へ移動します。
![GitHubSecrets](https://github.com/hoisjp/hoisjp/blob/master/archive-qiita/assets/Screenshot_GitHubSecrets_20210608.png?raw=true)

`AZURE_CREDENTIALS` という名前で、上記手順2のサービスプリンシパルJSON結果をコピー＆ペーストします。

```json
{
    "clientId": "<GUID>",
    "clientSecret": "<GUID>",
    "subscriptionId": "<GUID>",
    "tenantId": "<GUID>",
    // (...)
}
```
補足：JSONの結果はプロパティがもう少し多くずらっと出力されます。そのままペーストしても問題ありませんが、最低限、上記4つのプロパティがあれば十分です。

## Bicep ファイル
これはフォーク元のサンプルファイルをそのままです。Storage Account を作成するものですね。ロケーションは東日本リージョンに変更しておきました。

```
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
    name: uniqueString(resourceGroup().id)
    location: 'japaneast'
    kind: 'Storage'
    sku: {
        name: 'Standard_LRS'
    }
}
```

## GitHub Actions ワークフローファイル
前置きが長くなりましたが、ここが記事のメインコンテンツです。  
長いですが、いったん全て貼り付けます。GitHub リポジトリ上では[こちら](https://github.com/hoisjp/vscode-remote-try-bicep/blob/main/.github/workflows/bicep-deploy-azcli.yml)です。ファイルパス＆ファイル名を以下のように作成すれば、GitHub Actionsと認識されます。
コードコメントで補足説明を追記しました。
[.github/workflows/bicep-deploy-azcli.yml](https://github.com/hoisjp/vscode-remote-try-bicep/blob/main/.github/workflows/bicep-deploy-azcli.yml)

```yaml
name: 'BicepDeploy with Azure CLI'

# トリガーはお好きなように
# docs も参考に > https://docs.github.com/ja/actions/reference/events-that-trigger-workflows
# Web UI から直接トリガーできるように、"workflow_dispatch" も追加
on:
  push:
    branches:
    - main
  pull_request:
  workflow_dispatch:

jobs:

  BicepDeployAzCLI:
    name: 'BicepDeploy with Azure CLI'
    runs-on: ubuntu-latest
    # リソースグループ名やロケーションは env で指定
    env:
      ResourceGroupName: rg-bicep-samples
      ResourceGroupLocation: "japaneast"
    environment: production

    steps:

    - uses: actions/checkout@v2

    # GitHub Secrets に追加したクレデンシャルを参照して Azure にログインする。
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    # ビルドする。ARM テンプレートのJSONファイルが生成される。
    - name: Azure Bicep Build
      run: |
        az bicep build --file ./main.bicep
    # リソースグループを作成する。
    - name: Az CLI Create Resource Group
      uses: Azure/CLI@v1
      with:
        inlineScript: |
          #!/bin/bash
          az group create --name ${{ env.ResourceGroupName }} --location ${{ env.ResourceGroupLocation }}
    # what-if コマンドで実際の環境と diff をプレビューする。
    - name: Preview Azure Bicep
      uses: Azure/CLI@v1
      with:
        inlineScript: |
          #!/bin/bash
          az deployment group what-if -f ./main.json -g ${{ env.ResourceGroupName }} --name ${{ env.ResourceGroupName }}
    # Azure へ反映する。
    - name: Deploy Azure Bicep
      uses: Azure/CLI@v1
      with:
        inlineScript: |
          #!/bin/bash
          az deployment group create --mode Complete -f ./main.json -g ${{ env.ResourceGroupName }} --name ${{ env.ResourceGroupName }}
```
要点をまとめておきます。

- Azure Login は GitHub Secrets に登録した `${{ secrets.AZURE_CREDENTIALS }}` を参照します。GitHub Actions 実行時に認証エラーが出た場合はこのあたりを確認するとよいです。
- Azure Bicep Build 部分  
Bicep v0.3 から、以下のように、ARM Template の JSON ファイルを経由せずとも、直接 Bicep ファイルを引数に渡せるようになりました。ここでは作成された ARM Template の JSON ファイルを確認したいので、あえて `az bicep build` コマンドで .bicep => .json の変換をしています。  
`az deployment group create -f ./main.bicep ...`
- Preview Azure Bicep 部分 (`az deployment group what-if ...`)  
何を作るのか、何を削除するのか、確認するために、ARM Template の機能のひとつである [What-if 機能](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deploy-what-if?tabs=azure-cli)を使います。  
ここではそこまで作りこんでいませんが、**プルリクエストではこのプレビューで止めておき、マージしたときに後続の反映を行うのも有効**だと思います。
- Deploy Azure Bicep 部分 (`az deployment group create --mode Complete ...`)  
`--mode Complete` : これはデプロイモードのオプションで、**テンプレートに定義のないリソースは削除されます。**  
参照：[Azure Resource Manager のデプロイ モード](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/templates/deployment-modes)  
`--mode Incremental` : もうひとつのデプロイモードは増分モード（Incremental mode）というものです。このモードでは、リソースの削除までは実行されません。実環境に存在しないリソースを追加するだけです。IaC の文脈では、前者の Complete モードを想像されるかたが多いのではないでしょうか。

## GitHub Actions 実行
あとは、プッシュするなり、プルリクエストするなり、トリガーに応じてワークフローが実行されます。
![GitHubActions-Result](https://github.com/hoisjp/hoisjp/blob/master/archive-qiita/assets/Screenshot_GitHubActions-Result_20210608.png?raw=true) 
実行が完了したら、リソース定義が正しく Azure に反映されているか確認してみてください。

## おまけ - リソース全削除

terraform をお使いのかたは、`bicep destroy` はサポートされていないの？と思われるかもしれません。開発環境など使い終わったらきれいさっぱり削除したいケースもあるでしょう。残念ながら Bicep ではまだサポートされていませんが、GitHub Discussion ですでに議論されています。

https://github.com/Azure/bicep/discussions/1680

将来、**"Deployment Stacks" と呼ばれるライフサイクル管理の仕組み** が提供されるようですね。いかにも CI/CD と相性がよさそうで楽しみです。

いざ全削除する際は、リソースグループごと削除してしまうのもよいですが、前述の `--mode Complete` の仕組みに乗って、リソース削除が可能です。空の empty.bicep ファイルを与えることで、リソースグループの中身が空である、という定義に従って実行されます。
この方法だと、空のリソースグループが残ってしまうのですが、`what-if` の何を削除するかというdiffの結果を GitHub Actions のログに残せる点は便利です。

[.github/workflows/bicep-empty.yml](https://github.com/hoisjp/vscode-remote-try-bicep/blob/main/.github/workflows/bicep-empty.yml
)

```sh
az deployment group create --mode Complete -f ./empty.bicep -n ... -g ...
```

# 次のステップへ、役に立つ Tips

- ひととおりのテクニックを学ぶためには Microsoft Learn の Bicep ラーニングパスがおすすめです。  
https://docs.microsoft.com/ja-jp/learn/paths/bicep-deploy/  
（まだ英語版のみですが、じきに日本語版も提供されるでしょう。）
- VS Code の拡張機能についているスニペット機能を使うと便利です。リソース名を一部タイプしたところで補完機能を使えば、VS Codeがリソースの必須入力部分だけを出力してくれるので、あとは必要な値を埋めていくだけです。  
[Quickstart: Create Bicep files with Visual Studio Code](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/quickstart-create-bicep-use-visual-studio-code?tabs=CLI)  
![add-snippet](https://docs.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/media/quickstart-create-bicep-use-visual-studio-code/add-snippet.png)
（画像出典 公式docsより）
- いろいろと組み立て始めると、サンプルコードが欲しくなります。Bicep のサンプルコードはリポジトリの以下に集約されています。まずはここに実現したいものがないか探してみるのもおすすめです。かなり多くのケースがカバーされています。  
https://github.com/Azure/bicep/tree/main/docs/examples
  - 101: リソース単体ごとのサンプルコード
  - 201: リソースを複数組み合わせた、よくある構成を集めたもの
  - 301: より発展的で特定のユースケースをカバーするもの

# Feedback welcome!
ささいなことでも構いませんので フィードバックをいただけると とても喜びます。

- GitHub リポジトリ - https://github.com/hoisjp/vscode-remote-try-bicep
- Twitter - https://twitter.com/hoisjp

Enjoy Bicep and Azure!

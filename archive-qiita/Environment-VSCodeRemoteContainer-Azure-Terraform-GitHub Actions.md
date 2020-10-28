# Intro
Fork して clone したらすぐに Azure を Terraform できる devcontainer を作りました。 [VS Code の Remote Development (Remote - Containers)](https://code.visualstudio.com/docs/remote/containers) 機能を使っているので、ローカルに VS Code と Docker Desktop の環境があれば、以下のようなことがほんの少しの準備で実行できます。

- Terraform で Azure を管理する
  - `terraform` や Azure CLI `az` コマンドがすでに Docker コンテナ上のインストールされています。
- GitHub Actions で Push された内容を自動で反映する

GitHub レポジトリは以下です。
https://github.com/hoisjp/terraform-azure-ghactions-devcontainer

# 手順

## 事前に必要なもの

以下を準備してください。たったこれだけです！

1. Visual Studio Code : https://code.visualstudio.com/download
1. VS Code Extension : [VS Code Remote Development (Remote - Containers)](https://code.visualstudio.com/docs/remote/containers)
1. Docker Desktop : https://www.docker.com/get-started  
Windows/Mac/Linux、いずれの OS でも大丈夫なはずです。

## いざ！Remote Development！
1. ローカル環境で、Docker Desktop が実行中であることを確認してください。
1. 以下の GitHub リポジトリを Fork してください。あやまって秘密キーなどをプッシュしてしまわないよう、まずはプライベートレポジトリにしておくことをおすすめします。  
https://github.com/hoisjp/terraform-azure-ghactions-devcontainer  
※ fork せずに直接 git clone も可能ですが、後続の GitHub Actions を利用するためにはご自身のリポジトリに fork していただく必要があります。
1. fork したリポジトリを clone します。リポジトリ名は置き換えてください。

    ```sh
    git clone <your repository>
    ```
4. VS Code を clone したディレクトリで開いて、左下にある、Remote Development アイコンをクリックします。（コマンドパレットから実行しても構いません。）
    ![VS Code Remote Development](https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/raw/master/docs/images/launch-vscode-remotecontainer-01.png)
1. メニューから、`Remote-Containers: Reopen in Containers...` を選択します。
    ![Reopen in containers](https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/raw/master/docs/images/vscode-remote-menu-reopenincontainer.png)
1. ウィンドウが開き直りました。これは、ローカルの Docker コンテナ上で VS Code を開いている形になります。
VS Code のターミナルで以下を実行してみましょう。

    ```sh
    $ terraform -v
    Terraform v0.12.29
    
    $ az --version
    azure-cli                          2.9.1
    ...
    ```
    できました！準備OKです！

    FYI: ちなみに、`tflint` や `terragrunt` コマンドもすでにインストールされています。くわしくは、[Dockerfile](.devcontainer/Dockerfilehttps://github.com/hoisjp/terraform-azure-ghactions-devcontainer/blob/master/.devcontainer/Dockerfile) を見てください。

# Azure を Terraform で操作

## Terraform backends 用の Azure Storage
まず、[Terraform backends 用の Azure Storage](https://www.terraform.io/docs/backends/types/azurerm.html) を作ってみます。この Storage は Terraform のステートを管理するために必要なものです。

1. VS Code のターミナルを開きます。レポジトリのホームディレクトリにいるはずです。

    ```sh
    $ pwd
    /workspace/<your repository name>
    ```
1. Azure にログインします。ログイン用のページを開いて、指定されたコードを入力するよう求められます。以下、`************` の部分は置き換えてください。

    ```sh
    $ az login
    To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code ************ to authenticate.
    ```
    ログイン成功したら、所有している Azure サブスクリプションの一覧が JSON フォーマットでずらっと返されます。
1. 適切なサブスクリプションを選んで、以下のコマンドを実行します。ご自身のサブスクリプションの GUID で置き換えてください。先ほどの JSON 結果の `id` プロパティのものです。 

    ```sh
    $ az account set --subscription <your subscription GUID>
    ```
1. では、以下のディレクトリに移動します。
    ```sh
    $ cd 00-create-azurerm-backend
    ```
1. Terraform を実行するために状態を初期化する必要があります。
以下のように `terraform init` を実行します。

    ```sh
    $ terraform init

    Initializing the backend...

    Initializing provider plugins...

    Terraform has been successfully initialized!

    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.

    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
   ```
1. 初期化が成功しました。次に、`terraform plan` を実行します。
ストレージアカウント名の入力を求められます。これはグローバルでユニークである必要があります。他と重複しないよう入力します。

    ```sh
    $ terraform plan

    var.backend_storage_account_name
    Storage account name for terraform backend

    Enter a value: ****
    ```
    以下のような terraform plan 結果が出力されればOKです。

    ```
    ...
    ...
    Plan: 3 to add, 0 to change, 0 to destroy.
    ```

    注意: もし Azure にログインしていないと、以下のようなエラーメッセージが出ます。戻って `az login` から進めてください。

    ```
    Error: Error building AzureRM Client: Authenticating using the Azure CLI is only supported as a User (not a Service Principal).
    ```
1. では `terraform apply` を実行してリソースを作成します。ストレージアカウント名をもう一度求められますので、先ほど terraform plan で確認した名前を入力します。

    ```sh
    $ terraform apply

    var.backend_storage_account_name
    Storage account name for terraform backend

    Enter a value: ****
    ```
    確認メッセージを求められますので、確認OKであれば `yes` を入力します。

    ```sh
    ...
    ...
    Plan: 3 to add, 0 to change, 0 to destroy.

    Do you want to perform these actions?
    Terraform will perform the actions described above.
    Only 'yes' will be accepted to approve.

    Enter a value: yes
    ```
    作成されていますね。少し待ちましょう。

    ```
    azurerm_resource_group.rg: Creating...
    azurerm_resource_group.rg: Creation complete after 0s [id=/subscriptions/****GUID****/resourceGroups/terraform-rg]
    azurerm_storage_account.strg: Creating...
    azurerm_storage_account.strg: Still creating... [10s elapsed]
    azurerm_storage_account.strg: Still creating... [20s elapsed]
    azurerm_storage_account.strg: Creation complete after 20s [id=/subscriptions/****GUID****/resourceGroups/terraform-rg/providers/Microsoft.Storage/storageAccounts/****your storage account name****]
    azurerm_storage_container.strg-container: Creating...
    azurerm_storage_container.strg-container: Creation complete after 0s [id=https://********.blob.core.windows.net/tfstate]
    ```

    以下のメッセージが表示されたら完了です！:thumbsup:

    ```
    Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
    ```
1. 念のため、ストレージアカウントが作成されているか確認してみましょう。

    ```sh
    $ az group show --name terraform-rg --out table
    (result)
    
    $ az storage account show --name '<replace by yours>' --out table
    (result)
    ```

# Go for GitHub Actions

（ここは後ほど更新します。仕組みをご存じのかたは以下のファイルを見ていただければと）
https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/blob/master/.github/workflows/01-hello-azure.yml

# カスタマイズ

一度動いたらばいろいろと環境をカスタマイズしたくなってきますね。以下のファイルたちを変更するとカスタマイズできます。
詳しくは、[VS Code のドキュメント](https://code.visualstudio.com/docs/remote/containers) も参考にしてください。

- [.devcontainer/devcontainer.json](https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/blob/master/.devcontainer/devcontainer.json)
- [.devcontainer/Dockerfile](https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/blob/master/.devcontainer/Dockerfile)
  - Azure CLI のイメージ `mcr.microsoft.com/azure-cli:latest` をベースに、Terraform 各種をインストールしています。

## dotfiles で各自のお好み味付け

このリポジトリをチームで利用すると便利ですが、シェルの設定などお好みのものがあるかもしれません。そんなときは利用者それぞれがご自身の dotfiles でコンテナ上の設定をパーソナライズできます。以下のファイルの該当箇所を見てみましょう。

[.devcontainer/devcontainer.json](https://github.com/hoisjp/terraform-azure-ghactions-devcontainer/blob/master/.devcontainer/devcontainer.json)

```json
    "settings": {
        // ...
        // dotfiles
        "dotfiles.repository": "hoisjp/terraform-azure-ghactions-devcontainer", // change here to your repository.
        "dotfiles.targetPath": "~/.devcontainer/dotfiles",
        "dotfiles.installCommand": "~/.devcontainer/dotfiles/install.sh"
    },
```
ここで dotfiles リポジトリの参照先を変更するとコンテナ作成時に適用してくれます。
仕組みはこのドキュメントを参考にしてください。 [Personalizing with dotfile repositories](https://code.visualstudio.com/docs/remote/containers#_personalizing-with-dotfile-repositories)

# まとめ

VS Code の Remote Development 機能は非常に強力です。このように、チームメンバーで全く同じコンテナ環境を、**特別な手間なく自然に**（ここが大事）共有することができます。
従来、Azure CLI をインストールして、Terraform をインストールして、、、といった準備を、チームの全員が各自それぞれで行う必要がありましたが、devcontainer の仕組みを使うことで、もう「環境準備手順書」のようなものは必要ありません。環境設定はすべて git 上で管理されています。誰かが環境を改善すれば、たたちにチーム全員に反映されます。
これで Azure と Terraform に集中できますね！

# 参考

## VS Code ドキュメント

- Developing inside a Container : https://code.visualstudio.com/docs/remote/containers
- VS Code 本を書きましたのでよかったら！:wink: : https://www.amazon.co.jp/dp/4839970920/

## Terraform for Azure ドキュメント

- https://docs.microsoft.com/en-us/azure/developer/terraform/
- Terraform - Azure Provider : https://www.terraform.io/docs/providers/azurerm/index.html
- [Terraform - Azure Provider - GitHub Repos](https://github.com/terraform-providers/terraform-provider-azurerm)
- [Terraform - Azure Provider - GitHub Repos - Examples](https://github.com/terraform-providers/terraform-provider-azurerm/tree/master/examples)

## GitHub Actions for Azure ドキュメント
- https://docs.microsoft.com/ja-jp/azure/developer/github/github-actions

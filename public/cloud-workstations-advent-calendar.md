---
title: "Google Cloud Workstations入門: 安全かつ再現可能な開発環境の作り方"
tags:
  - GoogleCloud
  - CloudWorkstations
  - SSH
private: true
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

_この記事は [3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake) (12 日目) の投稿です。_

こんにちは！ スリーシェイクのイリドリシ愛民 ([@realaminevg](https://x.com/realaminevg)) です。

最近は主にクライアントワークを行なっているため、セキュリティやオンボーディングを徹底する必要があります。

ローカルで開発する時は、

- チーム全員が安全な開発環境を使用していること
- 新しいチームメンバーがすぐに作業を開始できること

としていますが、実際に上記条件を保つのは難しいです。

そこで、**Google Cloud Workstations**を導入しました。セキュリティと再現性を重視し構築されており、日々の開発がより楽になりました。

この記事では、以下の 3 点について解説します。

- Google Cloud Workstations とは何か、なぜ使用するのか
- Cloud Workstations を設定する方法
- ブラウザまたはローカルの IDE から Cloud Workstations に接続する方法

さらに公式ドキュメントに載っていない、ローカルの IDE から**ワンステップ**で Cloud Workstations を起動する方法も紹介します。

# Google Cloud Workstations とは

まずは公式の定義を見てみましょう。

> _（Google Cloud Workstations は）_ 機密性の高いエンタープライズのニーズに応えるように構築されたフルマネージド開発環境。Google Cloud 向けの Gemini とのネイティブな統合などにより、開発環境のセキュリティを強化するとともに、デベロッパーのオンボーディングと生産性を加速します。

つまり、

- どこからでもログインできる**安全な開発環境**が手に入る
- **チーム全員が同じ**開発環境を利用できる
- 開発環境で**Gemini**を使用し、**AI を活用した開発**ができる

ある意味、[Github Codespaces](https://github.com/features/codespaces?locale=ja)や Google 独自の[Firebase Studio](https://firebase.studio/)と共通点が多いかもしれません。一番異なるのが、**カスタマイズの幅**です。

- プリセットなどに縛られず、インスタンスタイプを自由に設定したり
- IAM を通じて開発環境の安全性を向上させたり
- 最初に提供されるコードエディタ（多くはブラウザでのみ利用可能）以外の IDE を使用したり
- などなど

Github Codespaces や Firebase Studio は、すぐに開発を着手できる環境を提供しますが、上記のカスタマイズは不可能に近いです。

一方、Google Cloud Workstations は、セットアップに少し手間がかかりますが、環境を**完全にカスタマイズできるのが大きな利点です**。

# Cloud Workstations を使ってみよう

早速 Cloud Workstations を使ってみましょう！

ワークステーションクラスタとワークステーション構成の設定手順をガイドします。その後、ワークステーションを作成できるようになります。

## 前提

- Google Cloud アカウントとプロジェクトを作成していること
- [Cloud Workstations API](https://console.cloud.google.com/flows/enableapi?apiid=workstations.googleapis.com)を有効にしていること

## 1. ワークステーションクラスタを作成する

[ワークステーションクラスタ](https://cloud.google.com/workstations/docs/architecture?hl=ja#workstation_cluster)は、複数のワークステーションを管理するものです。Google Cloud リージョンごとに一度だけクラスタを作成する必要があります。

Cloud Workstations の[クラスタ管理ページ](https://console.cloud.google.com/workstations/clusters)に移動し、新しいクラスタを作成します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/06b4f8a4-c240-462a-a471-fc6072b9c9fe.png)

クラスタ名を「test-cluster」にします。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/8ab0ee22-2f7d-411c-8ea2-88decd52b181.png)

これでクラスタが作成されます。作成されるまで 20 分かかることがあるのでご注意ください。

## 2. ワークステーション構成を作成する

[ワークステーション構成](https://cloud.google.com/workstations/docs/create-configuration?hl=ja)はマシンタイプ、ディスクサイズ、コンテナイメージなど、ワークステーションの設定を共通化するテンプレートです。ワークステーション構成を作成し、チーム全体の環境を統一化できます。

今回はワークステーション構成のデフォルト設定をそのまま利用しますが、詳しく知りたい方は[ドキュメント](https://cloud.google.com/workstations/docs/create-configuration?hl=ja)を参照してください！

Cloud Workstations の[ワークステーション構成ページ](https://console.cloud.google.com/workstations/configurations)に移動し、「作成」ボタンを押します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/31b1480e-0f75-4ebc-a293-07ee3fe5b2dd.png)

構成は「test-configuration」と名付け、前に作成した「test-cluster」クラスタを利用します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/9e165968-d698-4f23-b003-12ed58b06740.png)

## 3. ワークステーションを作成する

いよいよ、ワークステーションを作成できます！

Cloud Workstations の[ワークステーションページ](https://console.cloud.google.com/workstations/list)で「ワークステーションを作成する」ボタンを押します。

![ワークステーションを作成する](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/5cefe225-161b-411c-81d8-f783ad65d668.png)

ワークステーションは「test-workstation」と名付け、前に作成した「test-configuration」構成を利用します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/27f1540f-1b36-43b0-9d8e-952b6bcfa842.png)

「作成」ボタンを押すと、ワークステーションが作成されます！🎉

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/28a1fd98-5262-4f3a-855f-5afb142bf518.png)

# Cloud Workstations に接続しよう

ワークステーションへの接続方法を 2 つ紹介します。

- ブラウザから接続する
- ローカル IDE から接続する

まず、Cloud Workstations の[ワークステーションページ](https://console.cloud.google.com/workstations/list)で「Start」（起動）ボタンを押し、ワークステーションを起動します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/b63d9020-c00b-47dd-8811-87056bdee66b.png)

## ブラウザから接続する

ワークステーションを起動したときに表示された「開始」ボタンを押します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/4d16f39b-7bdd-4927-8e38-a1345443e329.png)

ブラウザ内でコードエディタが表示されます。VSCode のオープンソース版「Code-OSS」をもとにしたコードエディタのため、UI に見慣れている方が多いのではないでしょうか。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/d54c9fe6-56db-4774-9cea-b8049a224965.png)

このコードエディタでは、リポジトリをクローンしたり、プログラムを実行したりと、通常の開発環境と同じように作業ができます！

## ローカル IDE から接続する

ローカル IDE を通じてワークステーションに接続することもできます。

[公式ドキュメント](https://cloud.google.com/workstations/docs/develop-code-using-local-vscode-editor?hl=ja)をもとに接続方法を紹介します。

（ここでは VS Code ベースのエディタのみを扱いますが、[JetBrains IDE](https://cloud.google.com/workstations/docs/develop-code-using-local-jetbrains-ides)も使用できます！）

接続する前に、以下手順を完了してください。

- ワークステーションを起動する
- [gcloud CLI](https://cloud.google.com/sdk/docs/install)をインストールする
- ローカル IDE で「[Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)」 VS Code 拡張機能をインストールする

まず、ワークステーション名を調べる必要があります。Cloud Workstations の[ワークステーションページ](https://console.cloud.google.com/workstations/list)で「詳細を表示」リンクをクリックしワークステーション名を確認できます。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/17da3cb1-fbb7-42c7-b99d-e3a710d35832.png)

ワークステーション名をコピーします。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/dfb58ba2-d163-47a3-a61e-33972659e67c.png)

ワークステーション名が分かったら、以下のコマンドを実行します。（ワークステーション名を置き換えます）

```bash
gcloud workstations start-tcp-tunnel --local-host-port=:9998 <YOUR_WORKSTATION_NAME> 22
```

ローカル環境からワークステーションへの TCP トンネルをポート 9998 で開きます。

```
Listening on port [9998].
```

開いた TCP トンネルを利用しローカル IDE からワークステーションに接続できます！[リモートホストへの接続方法](https://code.visualstudio.com/docs/remote/ssh#_connect-to-a-remote-host)に従い、リモートホストは`user@localhost:9998`を入力します。

# Cloud Workstations へのワンステップ接続方法

ローカル IDE からワークステーションに接続する方法を紹介しました。が、もっと良い方法はないでしょうか？

上記の方法では、ワークステーションに接続するには**3 つのステップ**が必要です。

1. Cloud Workstations の[ワークステーションページ](https://console.cloud.google.com/workstations/list)でワークステーションを起動する
2. ターミナルで`gcloud workstations start-tcp-tunnel`コマンドを実行する
3. ローカル IDE で[ワークステーションに接続する](https://code.visualstudio.com/docs/remote/ssh#_connect-to-a-remote-host)

3 つのステップと聞くと、思うほど多くないかもしれません。しかし、「Web ブラウザ」「ターミナル」「ローカル IDE」と 3 つのプログラムを操作する必要があります。このような[コンテキストスイッチが生産性の低下](https://atlassian-teambook.jp/_ct/17507299)に繋がり、できるだけ避けたいものです。

そこで、**ワンステップでワークステーションに接続する**、公式ドキュメントに載っていない接続方法を紹介します。

## 1. スクリプトを利用しワンステップ接続を実現する

ワークステーションの起動から TCP トンネルの作成まで、接続作業全体を自動化するスクリプトを提供します。SSH 設定ディレクトリに配置します。（例：`~/.ssh/workstation-startup.sh`）

```bash
GCP_WORKSTATION_NAME="$1"
GCP_TUNNEL_HOST="$2"
GCP_TUNNEL_PORT="$3"

trap 'kill "$jpid" 2>/dev/null' EXIT

>&2 echo "Starting server..."

gcloud workstations start $GCP_WORKSTATION_NAME 2> /dev/null

>&2 echo "Server is ready!"

>&2 echo "Starting TCP tunnel..."

gcloud workstations start-tcp-tunnel --local-host-port=$GCP_TUNNEL_HOST:$GCP_TUNNEL_PORT $GCP_WORKSTATION_NAME 22 2> /dev/null &
jpid="$!"

until nc -z $GCP_TUNNEL_HOST $GCP_TUNNEL_PORT 2>/dev/null; do
    sleep 1
done

>&2 echo "Started tunnel on host $GCP_TUNNEL_HOST and port $GCP_TUNNEL_PORT"

nc $GCP_TUNNEL_HOST $GCP_TUNNEL_PORT
```

スクリプトを実行するときの流れは以下になります。

- ワークステーションを起動する (`gcloud workstations start`)
- ワークステーションへの TCP トンネルをバックグラウンドで開く (`gcloud workstations start-tcp-tunnel`)
- TCP トンネルが開くまでに待機する (`until nc ...`)
- netcat (`nc`) コマンドを通じて TCP 通信を SSH にリダイレクトする

## 2. SSH 設定を変更する

次は、SSH 設定ファイル（通常は`~/.ssh/config`）を開き、以下を追加します。

```yaml
Host gcp-cloud-workstation # Host名は好きな名前に変更OK
  HostName localhost
  Port 9998 # Portは好きなポートに変更OK
  User user
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  ConnectTimeout 0
  ProxyCommand sh ~/.ssh/workstation-startup.sh "<YOUR_WORKSTATION_NAME>" %h %p # 実際のワークステーション名に変更
```

`ProxyCommand`を利用しスクリプトを実行します。

:::note info
**`StrictHostKeyChecking`と`UserKnownHostsFile`の設定に関して**

ワークステーションを起動するたびに SSH ホストキーが変わるため、そのままスクリプトを実行すると、以下の警告が表示され、接続できなくなります。

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
```

TCP トンネル経由で localhost に安全に接続しているため、この警告を無効にしても問題ありません。
:::

## 3. ワークステーションを起動し接続する

これで、ワンステップでワークステーションに接続できます！追加した SSH ホスト「gcp-test-workstation」を選択すれば、ローカル IDE がワークステーションを起動し、TCP トンネルを開き、ワークステーションに接続します。

![](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/6d350229-1fb5-435f-8ccd-8a9fbe0fcb05.png)

# まとめ

この記事では、Google Cloud Workstations の基本的な使い方や、ローカル IDE からのワンステップ接続方法まで解説しました。

Google Cloud Workstations をうまく利用すれば、安全性・再現性の高い開発環境を実現できます。ブラウザ内であろうとローカル IDE であろうと、この記事が Google Cloud Workstations を試してみるきっかけになれば嬉しいです。

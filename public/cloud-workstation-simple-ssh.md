---
title: ローカルエディタからワンクリックでGoogle Cloud Workstationに接続する方法
tags:
  - SSH
  - netcat
  - GoogleCloud
  - CloudWorkstations
private: false
updated_at: '2025-07-02T22:43:33+09:00'
id: 27f55b1809b6629567f6
organization_url_name: null
slide: false
ignorePublish: false
---

# 背景

皆さんは、Google Cloud Workstationsという製品はご存知ですか？

「フルマネージド開発環境」を提供していて、セキュリティの強化や開発者のオンボーディングの加速を期待できる製品です。

https://cloud.google.com/workstations?hl=ja

クラウド上の開発環境ということもあって、ブラウザ内での開発が可能ですが、ローカル環境での開発が必要な場面があります。

本記事では、ローカル環境からCloud Workstationsに接続できる、簡単な方法をご紹介します！

## ローカル環境での開発の必要性

Google Cloud上の開発環境ということで、Webからアクセスし、ブラウザ内で開発を進めることができます。

ただし、「ローカル環境で開発したい」という方が多いと思います。例えば、

- すでに設定しているキーボードショートカットやテーマをそのまま使いたい
- CursorやGitHub Copilotなどの最新AI開発ツールを活用したい
- レスポンスの良い操作感で快適に開発できる

などなど、ローカル環境での開発の方が快適な場面が少なくありません。

## 公式の接続方法の問題点

それに備えて、ローカル環境のエディタ（VSCode）からCloud Workstationに接続する方法を公式が公開しています。

https://cloud.google.com/workstations/docs/develop-code-using-local-vscode-editor?hl=ja

上記リンクの手順に従えば、問題なくCloud Workstationに接続することができます。

ただし、問題としてはやはり手間がかかります。なぜかというと、Cloud Workstationに接続したいときに必ず以下の3ステップに従う必要がありますから。

1. **GCPコンソールで**Cloud Workstationを起動する（シャットダウンした場合）
2. **ターミナルで**TCPトンネルを開く
3. **ローカルエディタで**Cloud WorkstationにSSHを通して接続する

開発を着手したい時に、3箇所を開き、毎日同じ作業を行うのが少しストレスがかかります。

そこで、ローカルエディタで完結できる方法をご紹介します。

# セットアップ

## 事前準備

以下の準備を完了していることを前提としています。
        
- `gcloud` CLIがインストール済み・認証完了
- SSHが利用可能（以後は設定フォルダを`~/.ssh`としますが、異なる場合は適宜修正してください）
- Cloud Workstationの**ワークステーション名**は把握済み
  - GCPコンソールのWorkstations詳細画面で確認できます（以下を参考にしてください）

![cloud-workstation-simple-ssh-sample-project-name.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/df30e8d1-6366-47b8-9ada-7058bbdc5d21.png)

## 1. SSH接続スクリプトを作成

以下のスクリプトを`~/.ssh/workstation-startup.sh` に置きます。

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

1. Cloud Workstationを起動 (`gcloud workstations start`)
2. Cloud WorkstationのTCPトンネルをバックグラウンドで開く (`gcloud workstations start-tcp-tunnel`)
    - 開いたTCP通信をリダイレクトしたいのでバックグラウンドで実行しています
    - SSH切断時にはバックグラウンドプロセスも自動的に終了するよう設計しています
3. TCPトンネルが開くまでに待機
4. TCP通信をSSHにリダイレクト

## 2. SSH設定を追加

次は、sshのconfigファイル（ `.ssh/config` ）に以下を追加してください。

Host、Port、「ワークステーション名」を適宜変更してください。

```bash
Host gcp-cloud-workstation # Host名はお好みの名前に変更OK
  HostName localhost
  Port 9997 # Portはお好みに変更OK。ローカルサーバーと被る場合は変更が必要
  User user
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  ConnectTimeout 0
  ProxyCommand sh ~/.ssh/workstation-startup.sh "ワークステーション名" %h %p # 実際のワークステーション名に変更
```

:::note info
StrictHostKeyChecking、UserKnownHostsFileの設定に関して

Workstationが起動するたびにHost Keyが変わるため、既存設定では以下の警告が表示され、接続ができなくなります。
        
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
```

TCPトンネル経由でlocalhostに安全に接続しているため、この警告を無効にしても問題ありません。
:::

これでセットアップ完了です！

# 実際の接続方法（VSCode/Cursor）

ワンクリック接続の手順になります。

1. [**Remote - SSH**](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)拡張機能をインストール
2. コマンドパレット (`Cmd/Ctrl + Shift + P`) を開き、「`Remote-SSH: Connect to Host...`」を選択
3. 先ほど設定したHost名（例: `gcp-cloud-workstation`）を選択し接続

これだけです！今度はコンソールやターミナルをいじることなく、ローカルのエディタから完結して接続することができます。

## もしタイムアウトする場合

接続がタイムアウトする場合は、VSCode設定の `remote.SSH.connectTimeout` 値を変更してください。（300秒に設定すれば概ねOKです）

参考として以下のGithub issueをご確認ください。

https://github.com/microsoft/vscode-remote-release/issues/8519

# まとめ

この記事では、Google Cloud Workstationsへの接続を**3ステップから1ステップに短縮**する方法をご紹介しました。

これで、Google Cloud Workstationsがより身近で使いやすいツールになったはずです。参考になれば嬉しいです！

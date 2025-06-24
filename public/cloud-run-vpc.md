---
title: Terraformを使ってVPC内のCloud Runサービス間で通信させる
tags:
  - GoogleCloud
  - CloudRun
  - VPC
  - Terraform
  - IaC
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## 背景

## なぜVPCが必要なの？

* プライベートサービスを利用したい場合（例：バックエンドをプライベート、フロントエンドをパブリック）
* Cloud RunサービスからVPC内の他リソースにアクセスしたい場合 (Compute Engine, Cloud SQLなど)
* VPC内では

## Cloud RunとVPC

実は「VPC内」という言い方は語弊で、正確にいうと「VPCに接続する」の方が正しい

接続方法

* Direct VPC Egress
* サーバーレスVPCアクセス

Directの方が良いという話をする

## やり方

## まとめ

VPC内にCloud Run間で通信できるようには（「！！」は詰まったところ）
* VPCを作る
  * auto modeはだめ！！（多分）
* VPCのsubnetを作る
  * private google accessをonにする！！
* 繋げたいCloud RunサービスにVPCをつける
  * Direct VPC Egress
  * egress設定はやりたいことによる
  * LLMなど叩くならPRIVATE_RANGES_ONLY
  * 外部（インターネット）通信はないならALL_TRAFFIC
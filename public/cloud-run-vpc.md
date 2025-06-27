---
title: Terraformを使ってVPC内のCloud Runサービス間で通信させる
tags:
  - IaC
  - vpc
  - Terraform
  - GoogleCloud
  - CloudRun
private: false
updated_at: '2025-06-27T16:26:55+09:00'
id: 4912c95b795c6739d703
organization_url_name: null
slide: false
ignorePublish: false
---
# 背景
Cloud Runはサーバーレスでコンテナを動かせる便利なサービスですが、複数のサービスを連携させようとすると、ネットワーク構成で悩むことがあります。

例えば、
* フロントエンドは一般公開し、バックエンドは内部ネットワークからのみアクセス可能にしたい場合
* VPC内の他のリソース（Cloud SQLなど）と通信したい場合

など、VPCの導入が必要な場合は、意外と落とし穴が多く、ハマってしまう方が少なくないはずです。

本記事では、Terraformを使い、VPCネットワークを構築して2つのCloud Runサービス（非公開バックエンドと公開フロントエンド）を安全かつ効率的に通信させる方法を、具体的なコードを交えて解説します。

# なぜVPC接続が必要か？
Cloud RunでVPC接続が役立つ代表的なケースは以下の通りです。

*   **プライベートサービスの実現**
    * フロントエンドは公開し、バックエンドは非公開にする構成が作れます。
*   **VPC内リソースへのアクセス**
    * Compute EngineやCloud SQLといったVPC内の他リソースへ安全に接続できます。
*   **通信の効率化**
    * VPC内の通信は、場合によってレイテンシの削減が期待できます。

# Cloud RunのVPC接続方式
Cloud RunをVPCに接続するには、主に2つの方法があります。

* [**Direct VPC Egress**](https://cloud.google.com/run/docs/configuring/vpc-direct-vpc?hl=ja)
    * 推奨されている新しい方式です。セットアップが簡単で、追加料金が発生しません。
* [**サーバーレスVPCアクセス**](https://cloud.google.com/run/docs/configuring/vpc-connectors?hl=ja)
    * 従来からある方式です。特別な要件がない限り使わなくて良い印象です。

「どれを選べばいいか？」を悩む方は、以下比較表を参考にすると良いです。

https://cloud.google.com/run/docs/configuring/connecting-vpc?hl=ja#comparison-table

本記事では**Direct VPC Egress**を利用します。

# Terraformによる実装
Terraformを使った具体的な実装方法を解説します。

全体のコードは以下のリポジトリで確認できます。実行方法もこちらを参照してください。

https://github.com/aminevg/cloud-run-vpc-terraform-sample

## 全体構成
今回の構成図は以下の通りです。インターネットからアクセスできるフロントエンドと、VPC内部でのみ通信可能なバックエンドを構築します。

![cloud-run-vpc-architecture.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3812756/f68fff26-2d5a-42f8-bf58-6271a71c9cc0.png)

## アプリケーション
VPC内での通信を確認するため、2つのシンプルなサービスを用意します。

### バックエンド (Bun)
リクエストを受けるとランダムなUUIDを返す単純なAPIサーバーです。

```ts
import { randomUUIDv7 } from "bun"

const server = Bun.serve({
  port: process.env.BACKEND_PORT ?? "3000",
  routes: {
    "/random-uuid": (req) => {
      return new Response(randomUUIDv7());
    }
  }
})

console.log(`Server is running on ${server.url}`);
```

### フロントエンド (Astro)
バックエンドAPIを呼び出し、取得したUUIDを表示するWebページです。オンデマンドレンダリングを有効にしているため、リロードするたびに新しいUUIDが表示されます。

```tsx
---
const backendUrl = process.env.BACKEND_URL ?? "http://localhost:3000";

const response = await fetch(`${backendUrl}/random-uuid`);
const data = await response.text();
---

<html lang="en">
	<head>
		<meta charset="utf-8" />
		<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
		<meta name="viewport" content="width=device-width" />
		<meta name="generator" content={Astro.generator} />
		<title>Random UUID</title>
	</head>
	<body>
		<h1>{data}</h1>
	</body>
</html>
```

## インフラ構成 (Terraform)
インフラはTerraformでコード化します。

### 1. Artifact Registry
はじめに、バックエンドとフロントエンドのDockerイメージを保存するためのArtifact Registryリポジトリを作成します。

```hcl
resource "google_artifact_registry_repository" "cloud_run_vpc_terraform_sample_repository" {
  repository_id = "cloud-run-vpc-terraform-sample-repository"
  format        = "docker"
}
```

### 2. ネットワーキング
次に、サービス間通信の基盤となるネットワークを構築します。

#### VPC
まず、VPCネットワークを定義します。

```hcl
resource "google_compute_network" "cloud_run_vpc_terraform_sample_network" {
  name                    = "cloud-run-vpc-terraform-sample-network"
  project                 = var.project
  auto_create_subnetworks = false
}
```

:::note info
`auto_create_subnetworks`は`false`に設定すること（自動モードを無効にすること）がポイントです！

なぜかというと、次のステップでPrivate Google Accessを有効にするため、カスタムサブネットを作成する必要があるからです。

自動モードを有効にする場合に注意するところも他にもあります。詳しくは以下リンクを参照してください。

https://cloud.google.com/vpc/docs/vpc?hl=ja#auto-mode-considerations

本記事では自動モードを無効にします。
:::


#### サブネット
Cloud Runサービスを接続するためのサブネットを作成します。

```hcl
resource "google_compute_subnetwork" "cloud_run_vpc_terraform_sample_subnetwork" {
  name                     = "cloud-run-vpc-terraform-sample-subnetwork"
  network                  = google_compute_network.cloud_run_vpc_terraform_sample_network.id
  region                   = var.region
  ip_cidr_range            = "10.0.0.0/24"
  private_ip_google_access = true
}
```

`private_ip_google_access = true`にすることで、Private Google Access（限定公開のGoogleアクセス）を有効にし、Cloud Run間の通信の他、Cloud SQLなど外部IPアドレスが持たないサービスとの通信ができるようになります。

https://cloud.google.com/vpc/docs/configure-private-google-access?hl=ja

#### DNS
VPC経由でバックエンドサービスを呼び出すには、Cloud Runのエンドポイント (`run.app`) をGoogleのプライベートIPアドレス範囲に解決させる必要があります。

https://cloud.google.com/run/docs/securing/private-networking?hl=ja#from-other-services

```hcl
resource "google_dns_managed_zone" "cloud_run_vpc_terraform_sample_dns_zone" {
  name     = "cloud-run-vpc-terraform-sample-dns-zone"
  dns_name = "run.app."

  visibility = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.cloud_run_vpc_terraform_sample_network.id
    }
  }
}

resource "google_dns_record_set" "cloud_run_vpc_terraform_sample_dns_record_set_a" {
  name         = "run.app."
  type         = "A"
  ttl          = 60
  managed_zone = google_dns_managed_zone.cloud_run_vpc_terraform_sample_dns_zone.name
  rrdatas      = ["199.36.153.4", "199.36.153.5", "199.36.153.6", "199.36.153.7"] # private.googleapis.com
}

resource "google_dns_record_set" "cloud_run_vpc_terraform_sample_dns_record_set_cname" {
  name         = "*.run.app."
  type         = "CNAME"
  ttl          = 60
  managed_zone = google_dns_managed_zone.cloud_run_vpc_terraform_sample_dns_zone.name
  rrdatas      = ["run.app."]
}
```

これでVPCの設定がやっと完了です！

### 3. Cloud Run
最後に、フロントエンドとバックエンドのCloud Runサービスを定義します。

#### バックエンドサービス
バックエンドはVPC内部からのリクエストのみを受け付け、外部には公開しません。

```hcl
resource "google_cloud_run_v2_service" "cloud_run_vpc_terraform_sample_backend" {
  name     = "cloud-run-vpc-terraform-sample-backend"
  location = var.region
  ingress  = "INGRESS_TRAFFIC_INTERNAL_ONLY"

  template {
    vpc_access {
      egress = "ALL_TRAFFIC"
      network_interfaces {
        network    = google_compute_network.cloud_run_vpc_terraform_sample_network.id
        subnetwork = google_compute_subnetwork.cloud_run_vpc_terraform_sample_subnetwork.id
      }
    }

    containers {
      image = "${var.region}-docker.pkg.dev/${var.project}/${google_artifact_registry_repository.cloud_run_vpc_terraform_sample_repository.repository_id}/cloud-run-vpc-terraform-sample-backend:1"

      ports {
        container_port = 3000
      }

      env {
        name  = "BACKEND_PORT"
        value = "3000"
      }
    }
  }
}
```
*   `ingress = "INGRESS_TRAFFIC_INTERNAL_ONLY"`: VPC内部とロードバランサからのトラフィックのみを許可します。
*   `egress = "ALL_TRAFFIC"`: 全ての下り通信（egress）が、指定したVPCを経由するようにします。
    * この設定では外部通信（LLM APIの呼び出し）ができなくなるので、外部通信が必要な場合は `PRIVATE_RANGES_ONLY` にしても良いかもしれません。

#### フロントエンドサービス
フロントエンドは一般公開し、バックエンドへのリクエストはVPC経由で行います。

```hcl
resource "google_cloud_run_v2_service" "cloud_run_vpc_terraform_sample_frontend" {
  name     = "cloud-run-vpc-terraform-sample-frontend"
  location = var.region
  ingress  = "INGRESS_TRAFFIC_ALL"

  template {
    vpc_access {
      egress = "PRIVATE_RANGES_ONLY"
      network_interfaces {
        network    = google_compute_network.cloud_run_vpc_terraform_sample_network.id
        subnetwork = google_compute_subnetwork.cloud_run_vpc_terraform_sample_subnetwork.id
      }
    }

    containers {
      image = "${var.region}-docker.pkg.dev/${var.project}/${google_artifact_registry_repository.cloud_run_vpc_terraform_sample_repository.repository_id}/cloud-run-vpc-terraform-sample-frontend:1"

      ports {
        container_port = 4321
      }

      env {
        name  = "BACKEND_URL"
        value = google_cloud_run_v2_service.cloud_run_vpc_terraform_sample_backend.uri
      }
    }
  }
}
```
*   `ingress = "INGRESS_TRAFFIC_ALL"`: インターネットからの全ての上り通信（ingress）を許可します。
*   `egress = "PRIVATE_RANGES_ONLY"`: プライベートIP宛ての下り通信（egress）のみをVPC経由にします。これにより、バックエンドへの通信はVPC内で行われ、それ以外の外部APIへの通信はVPCを経由しません。


# まとめ

ここまでお読みいただきありがとうございます！

本記事では、Terraformを用いてCloud Runのフロントエンドサービスとバックエンドサービスを構築し、VPC経由で安全に通信させる方法を解説しました。

Direct VPC Egressの活用、Private Google Accessの有効化、DNSの設定など、注意すべきところが多かったので、皆さんのインフラ構成の参考になれたら嬉しいです。

最後に、今回のTerraform実装はこちらのGitHubリポジトリで確認できます。

https://github.com/aminevg/cloud-run-vpc-terraform-sample

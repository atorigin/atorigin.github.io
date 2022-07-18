---
layout: post
title: 002_利用 terraform 管理 cloudflare dns
date: '2022-07-18 15:46:31 +0800'
categories: [DevOps]
tags: [terraform,cloudflare]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---
# 目標
配置 terraform 來管理特定 cloudflare 的 zone 之 dns record 規則。

# 參考文件

## 設定 cloudflare 的 provider 說明
https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs

# module 使用說明
- terraform 要 >= 1.2.0, < 1.3.0
- cloudflare provider 要 >= 3.18.0

## tf 檔案配置

### provider
```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "3.18"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

### variables
```hcl
variable "cloudflare_api_token" {
  type        = string
  sensitive   = true
  description = "The Cloudflare API token."
}
variable "cloudflare_zone_id" {
  default = "<zone_id>"
}
```

### main
```hcl
resource "cloudflare_record" "blog" {
  zone_id = var.cloudflare_zone_id
  name    = "blog"
  value   = "atorigin.github.io"
  type    = "CNAME"
  ttl     = 3600
}
```

# 流程
1. 新增 tf 檔案配置
2. 建立 cloudflare API token
3. terraform 專案初始化
4. terraform plan 試跑查看部署結果
5. terraform apply 執行真實部署

# 步驟說明

## 新建 tf 檔
![](/commons/image/20220718/01-tf_file_setup.png)

## 創建 Cloudflare 的 API Token (用於 terraform)
![](/commons/image/20220718/02-cloudflare_create_token_1.png)
![](/commons/image/20220718/02-cloudflare_create_token_2.png)
![](/commons/image/20220718/02-cloudflare_create_token_3.png)
![](/commons/image/20220718/02-cloudflare_create_token_4.png)
![](/commons/image/20220718/02-cloudflare_create_token_5.png)
![](/commons/image/20220718/02-cloudflare_create_token_6.png)

### 注意 variables.tf 的 zone_id 需參考
![](/commons/image/20220718/02-cloudflare_create_zoneid_1.png)

## 執行 terraform 初始化、檢驗及部署
```bash
terraform init
terraform plan
terraform apply
```
![](/commons/image/20220718/03_terraform_command_1.png)
![](/commons/image/20220718/03_terraform_command_2.png)
![](/commons/image/20220718/03_terraform_command_3.png)
---
title: 022_使用 AWS DMS 來移轉 RDS Aurora MySQL 資料庫進行 VPC Resource 無痛遷移
date: '2024-12-24 00:00:00 +0800'
categories: [Cloud]
tags: [aws]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 起因
舊有的 AWS 上，VPC 在不同的環境中會有 CIDR Overlay 的問題，為了改善當前架構並因應外部稽核，需要重新設計 VPC 及其網路規劃。

# 目標
1. 需要轉移現有 VPC 內所有資源
2. 重新規範 VPC 內網路設定，其中包含 NACL、VPC Endpoint、Routing，Logging、Security Group 甚至是 Transit Gateway (多個 VPC Peering)

# 工具 - AWS DMS
- DMS Fleet Advisor 可以用來收集 on-prem 資料和分析 server 的資料來構建移轉到 aws 的 inventory 和 schema
- 如果要 migrate 不同的 db engine 可以使用 DMS Schema Conversion。可以用 AWS 提供的 AWS Schema Conversion Tool (AWS SCT) 來轉換 local 的 Schema
- 當完成轉換條件後，可以透過 DMS 做一次性移轉或持續變更性的移轉來保持 source 和 target 的資料 sync

> AWS DMS replication process 概念圖
{: .prompt-tip }
![](/commons/image/20241224/dms00.png)

## 執行 Migration Task 時 DMS 會做的

## 關於此次移轉的 Source And Target
- https://docs.aws.amazon.com/zh_tw/dms/latest/userguide/CHAP_Source.MySQL.html#CHAP_Source.MySQL.AmazonManaged
- https://docs.aws.amazon.com/zh_tw/dms/latest/userguide/CHAP_Target.MySQL.html


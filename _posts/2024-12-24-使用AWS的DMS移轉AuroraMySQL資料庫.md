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
- 在傳統 Solution 裡面，遷移 DB 需要分析 hardware 和 software，針對其安裝及管理。但使用 AWS DMS 會自動做完這些評估，並進行遷移。
- DMS 可以動態調整遷移的用量來符合實際的 workload，例如要增加額外的 disk space，只需要暫停遷移，調整後重新啟動遷移即可
- DMS 是 pay-as-you-go，按量計費
- DMS 自動管理所有執行 migration server 的 infra，包含 hardware/software/software patching 和 error reporting
- DMS 會自動 failover，如果主要的 replication server 發生任何原因而造成 failure，backup 的 replication server 會接手，只可能會造成一點或甚至 0 中斷。
- DMS Fleet Advisor 自動管理 inventory 中的 data infra，它會生成 report 來協助評估遷移的計畫
- DMS Schema Conversion 自動評估複雜的 source data provider，它也能轉換 database schema 和 code object 並格式化成 compatible 的 target database 可 applied 的內容
- DMS 也可以協助更換最新的 DB Engine，有可能會比現在使用的更加 cost-effective
- DMS 支援現在最熱門的所有 DB Engine 作為 source，和廣泛的 target
- DMS 支援 heterogeneous 的 data migration (相異的 db engine 轉換)
- DMS 可以確保安全性，對於 data 可以用 KMS 加密，對於傳輸，DMS 提供 SSL 來加密。

## 關於此次移轉的 Source And Target
- [MySQL Source](https://docs.aws.amazon.com/zh_tw/dms/latest/userguide/CHAP_Source.MySQL.html#CHAP_Source.MySQL.AmazonManaged)
- [MySQL Target](https://docs.aws.amazon.com/zh_tw/dms/latest/userguide/CHAP_Target.MySQL.html)
- [Homogeneous Migration Networking](https://aws.amazon.com/tw/blogs/database/migrate-an-on-premises-mysql-database-to-amazon-aurora-mysql-over-a-private-network-using-aws-dms-homogeneous-data-migration-and-network-load-balancer/)

## Source Data Provider (Database) - AWS RDS Aurora MySQL-8

### Prerequisites for CDC
- Automatic Backup Enabled on Instance Level
- Parameter Group `binlog_format` Set to `ROW`
- Ensure binlog remain time for DMS available
    - `call mysql.rds_set_configuration('binlog retention hours', 24);`
- `binlog_row_image` Set to `Full`
- `binlog_checksum` Set to `None`
- If Read Replica Source，`log_slave_updates` Set to `TRUE`
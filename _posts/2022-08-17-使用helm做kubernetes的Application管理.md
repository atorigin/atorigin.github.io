---
title: 010_使用 helm 來管理 kubernetes application
date: '2022-08-17 16:09:00 +0800'
categories: [Course]
tags: [java]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 目標
隨著專案越來越多，管理 kubernetes 內部 deployment、service、ingress、configmaps 物件越來越多。對於多檔切換來切換去的管理也越來越不堪負荷，也害怕自己哪裡改壞了，沒正確上道對應的配置導致部署有問題沒發現。最近就開始想要導入 helm 來來做在 kubernetes 內的 application 做管理。

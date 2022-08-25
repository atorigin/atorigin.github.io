---
title: 011_關於 SpringBoot 框架的配置魔術
date: '2022-08-17 16:09:00 +0800'
categories: [Course]
tags: [java,springboot]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 目標
了解 Java Springboot 框架在配置過程中，發生了哪些事情，經過哪些流程。

# 參考
<https://tanzu.vmware.com/content/springone-platform-2017/its-a-kind-of-magic-under-the-covers-of-spring-boot-brian-clozel-st%C3%A9phane-nicoll>

## SpringBoot 啟動器
<https://start.spring.io/>

> 通常 IDE 也會有相關的快速初始化一個 Springboot 專案的功能，例如 InteliJ IDEA 或 eclipse
{: .prompt-tip }

# 關於 Springboot
Springboot 的一個強大特性是它可以自動化配置，而該關鍵要素來自於 convention-over-configuration (約定優於設定)

# 拆解 annotations
- 在一個 springboot application 下，會在入口看到有一個 annotation 為 `@SpringBootApplication`
- 再往下看會看到 3 個 annotations，`@SpringbootConfiguration`、`@ComponentScan` 和 `@SpringBootAutoConfiguration`，關鍵在 `@SpringBootAutoConfiguration`

# 查看 pom.xml
- 針對 `parent` 字段，可以查看其依賴的父項目為 `spring-boot-starter-parent`
- 針對 `spring-boot-starter-parent` 的 pom 檔還可以查看到有 `parent` 字段，其依賴父項目為 `spring-boot-dependencies`
- 最後在看到 `spring-boot-dependencies` 的 pom 檔發現沒有 `parent` 字段了，而這個 pom 檔往下拉，會發現 `dependencyManagement` 字段，在該字段下就是在 `spring-boot-dependencies`所有依賴的 Jar

> 從 pom 可以看見 springboot 在 <b>`約定優於設定`</b> 的部分細節，因為在我們初始化整個專案的過程，它默默的依賴了相關的套件，默默定義的相關的設定並且記錄在 pom 檔內
{: .prompt-info }

# Java 階段學習
1. <https://www.bilibili.com/video/BV12J41137hu?>
2. <https://www.bilibili.com/video/BV1V4411p7EF?>
3. <https://www.bilibili.com/video/BV1LJ411z7vY?>
4. <https://www.bilibili.com/video/BV12J411M7Sj?>
5. <https://www.bilibili.com/video/BV1NE411Q7Nx?>
6. <https://www.bilibili.com/video/BV1WE411d7Dv?>
7. <https://www.bilibili.com/video/BV1aE41167Tu?>
8. <https://www.bilibili.com/video/BV1PE411i7CV?>
9. <https://www.bilibili.com/video/BV1jJ411S7xr?>

> Java SE -> Java 多線程 -> Java 網路程式設計 -> Java EE Web -> Spring -> SpringMVC -> SpringBoot -> SpringCloud
{: .prompt-tip }

## 課程 Note 參考
- <https://mp.weixin.qq.com/mp/homepage?__biz=Mzg2NTAzMTExNg==&hid=1&sn=3247dca1433a891523d9e4176c90c499>


# 自我學習紀錄

## Java EE Web

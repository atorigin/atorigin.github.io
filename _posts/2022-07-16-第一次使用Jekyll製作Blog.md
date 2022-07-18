---
title: 我製作的第一篇基於 Jekyll Chripy 主題的文章
author: atoring
date: 2022-07-16 20:48:00 +0800
categories: [Blog]
tags: [github-pages]     # TAG names should always be lowercase
mermaid: true
math: true
---

這是一篇關於我用 Jekyll 的主題 Chripy 製作的第一篇文章。希望透過這篇文章能夠展示 post 撰寫發布的基本功能。

## 修改 favicons 的相關圖檔
1. 先找到要用來做 favicons 的圖片，並利用 [製作 favicons 相關圖檔](https://realfavicongenerator.net/) 這個網站生成所需的相關檔案
    a. 上傳圖檔
    b. 移動到下方點擊 (Generate your Favicons and HTML code)
    c. 在頁面找到 (Download your package)，點擊旁邊的 Favicon package 下載。
2. 在資料夾內找到 assets/img/favicons 目錄，如果該目錄不存在，則創建此路徑
3. 將剛剛下載 favicon package 解壓縮，將裡面的 `.ico` 和 `.png` 檔案複製到 assets/img/favicons 目錄下

### 完成後，可以看見 favicon 已被換成自己設定的圖檔
![修改 favicon 完成後圖片](/commons/image/20220718/favicon_change.png)
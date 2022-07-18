---
title: 001_我製作的第一篇基於 Jekyll Chripy 主題的文章
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

## 關於 icon 的套件
這個 chirpy 主題會引入 submodule [fontawesome-free](https://github.com/cotes2020/chirpy-static-assets/tree/d1d2ec17c88176753d4dd2a1296620021dcc22fd)，路徑會在 assets/lib。
> 每當 build 的時候，這些 css 和 js 檔就會應用在生成 blog site 的網站座做使用

### 感想
基本上在使用 Jekyll 做為自己的 blog 建置的時候，樣式規則都遵循一套原則，可以參考 Jekyll 套件。

而如果要調整一些設定時，就要遵循 Jekyll 的 Content 配置方式。[Jekyll 文件](https://jekyllrb.com/docs/)

> 要客製化一些功能比較難，畢竟規則是固定的。
{: .prompt-info }
自己測試過要增加一個 sidebar 的 custom tab，起初以為在 `_tabs` 增加一個 `title.md` 檔案就可以了，但卻不知道要如何讓他 url 指到對應的目錄。

### 關於 Jekyll compose 工具
用來節省一些寫文件時的準備時間。[jekyll-compose Github 專案](https://github.com/jekyll/jekyll-compose)

可以透過這個工具建立 page / post 等等。

設定 jekyll compose 工具，以該工具新增頁面與新增 post 做測試

1. 先設定 Gemfile 安裝該工具
```bash
gem 'jekyll-compose', group: [:jekyll_plugins]
```
2. 安裝
```bash
bundle
```
3. 只用指令生成一個新的 post
```bash
bundle exec jekyll post '我的POST' --timestamp-format "%Y-%m-%d %H:%M:%S %z"
```
4. 查看 `_posts` 目錄下面會多一個 `.md` 就是剛剛生成的新的 post 文件檔，他會幫忙自動生成上面的 `Front Matter` 字段
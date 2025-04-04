---
title: 010_使用 helm 來管理 application on kubernetes
date: '2022-08-17 16:09:00 +0800'
categories: [DevOps]
tags: [helm,kubernetes]     # TAG names should always be lowercase
author: owen
mermaid: true
math: true
---

# 目標
隨著專案越來越多，管理 kubernetes 內部 deployment、service、ingress、configmaps 物件越來越多。對於多檔切換來切換去的管理也越來越不堪負荷，也害怕自己哪裡改壞了，沒正確上到對應的配置導致部署有問題沒發現。最近開始想要導入 helm 來做在 kubernetes 內的 application 做管理。

# 測試開發 Chart 心得總結
- 跟 golang template 有很大的關係，裡面語法大都類似
- 當要利用模板渲染時，要注意 scope，比較要注意的是 `with` 關鍵字。
- 條件控制 if-else 和 range 比較常用，range 雷同 foreach 的功能
- 要注意 template render 後的空格，適時的使用 {% raw %} `{{-` 和 `-}}` 絕大部分情況會用 `{{-` {% endraw %}
- 模板定義會用 `define` 包起來，並且 helm 建議所有定義都放在 `_helpers.tpl` 檔案裡
- 使用 `include` 或 `template` 引入模板，兩者差異在於 `include` 是 function，他可以透過 `|` 再次做數據處理，而 `template` 僅是執行複製貼上的動作，所以無法將輸出傳入給另一個 function 做輸入，因此 helm 建議用 `include` 可以更好的處理縮進問題


# Chart 基本概念

## Helm 安裝 resource 的順序
<https://helm.sh/docs/intro/using_helm/>

## Chart 基本結構

```yaml
mychart/
    Chart.yaml
    values.yaml
    charts/
    templates/
    ...
```

- templates 目錄存放 template 檔案，helm 在運行 chart 的同時，會將所有 templates 目錄裡面的檔案發送給 template rendering engine(模板渲染引擎)
- values.yaml 檔案對於 template 非常重要，這個檔案包含在整個 charts 的default values，當執行 `helm install` 或 `helm upgrade` 這些 values 會被用來覆寫。
- Chart.yaml 包含整個 chart 的描述，可以在 template 裡面訪問 Chart.yaml，而 charts/ 目錄會包含其他 chart，也可稱作為 subcharts

## 初探 templates/ 目錄的檔案
- NOTES.txt，一些提示字串，用來幫助安裝 chart 的時候顯示給 user 看的描述。
- deployment.yaml，基礎的 manifest，就是 kubernetes 的 deployment 物件。
- service.yaml，基礎的 manifest，就是 kubernetes 的 service 物件。
- _helpers.tpl，存放可以讓這個 chart 被重用的模板協助器。

> template 雖然沒有強制性規定命名規則，但 helm 建議還是用 `.yaml` 來作為 YAML 檔案的 suffix，且用 `.tpl` 作為模板協助器檔案的 suffix
{: .prompt-tip }

## 測試基本渲染 template 的方式

### 先創建 helm chart
```bash
helm create demo
```

> demo 為 chart name 可以隨意取，完成後會依照取的名子生成一個目錄。
{: .prompt-info }


### 移除 templates/ 目錄下所有檔案
```bash
rm -rf demo/templates/*
```

### 在 templates/ 目錄下新增一個 `configmaps.yaml` 檔案
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
{% raw %}  name: {{ .Release.Name }}-configmap {% endraw %}
data:
  myvalue: "Hello World"
```

> 這邊的關鍵在 metadata 下面的 name，後面用了一個 {% raw %} `{{ .Release.Name }}` {% endraw %}。
{: .prompt-info }

### 關於 {% raw %} {{ .Release.Name }} {% endraw %}的解讀
這邊的定義可以解讀為，從 top namespace 開始找 Release 物件，並從 Release 物件裡找到一個物件叫做 Name 的。

> Release 物件是 helm 物件的一種
{: .prompt-tip }

## helm 內建的物件
- 把定義的物件透過 template engine 傳入 template
- 測試 `with` 和 `range` statements
- 甚至有方法能直接在 template 裡面 new 一個物件，例如用 `tuple` function

> 物件可以單純是一個值，或是其它物件或 functions。
{: .prompt-tip }

### Release 物件
- Release 物件是在 template 中可以訪問的最高級物件
- Release 物件用來描述它自己本身，它內部有幾個物件
    - Release.Name 該 release 的名稱
    - Release.Namespace 該 namespace，如果 manifest 裡面沒有宣告
    - Release.Revision 該 release 的 revision 號碼，install 後為 1，往後 upgrade 會往上加，或 rollback 往下減

### Values 物件
- Values 透過 values.yaml 檔案傳入 template，預設情況下 Values 是空的

### Chart 物件
- 是 Chart.yaml 檔案的內容
- 在 Chart.yaml 檔案內的可用參數可參考 <https://helm.sh/docs/topics/charts/#the-chartyaml-file>


### Files 物件
- 用來提供在 charts 裡面訪問所有非特定 (non-special) 的檔案
- 檔案物件的使用方式，參考 <https://helm.sh/docs/chart_template_guide/accessing_files/>

### Capabilities 物件
- 提供 kubernetes cluster 支援的資訊

### Template 物件
- 包含當前被執行的 template 的資訊

> 補充，在 Go 裡面，大寫字母開頭通常被用來表示 build-in 的物件。是 Go 的命名原則。
{: .prompt-info }

## 進一步探討上面提到的 Value 物件
- 它可以透過多個來源來傳入 chart
    - 該 chart 裡面的 values.yaml 檔案
    - 如果是 subchart，來自 parent chart 的 values.yaml
    - 執行 command 時帶入的 `-f` 參數的指定 values.yaml 檔案
        - 例如 `helm install -f myvalues.yaml ./mychart`
    - 執行 command 時代入的 `--set` 參數指定的方式
        - 例如 `helm install --set foo=bar ./mychart`

### 順序優先級
- values.yaml 是最先的
- 如果 subchart 內的 values.yaml 值不存在，調用 parent 的 values.yaml 來覆蓋
- 而這些 values.yaml 檔可以被 user 定義的 `-f` 和 `--set` 參數覆蓋

> 測試為 `--set foo=bar` > `-f myvalue.yaml` > `values.yaml`
{: .prompt-tip }


# 延續基礎，在 template 中如何把內容再進一步做轉換變成我們想要用的內容

## 假想一種情況
前面我們透過 {% raw %} {{ .Release.Name }} {% endraw %} 的方式把 template 字元轉換成我們要的內容，但這種方式為固定一組對應的轉換，但隨著內容越來越複雜，我們會遇到要將內容轉換的同時，對這些內容作"再處理"。以下將使用 .Values 物件展示部分 Case。

## 在 inject 到 template 時，用 quote ("") 將內容包住

{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```
{% endraw %}

> 其實 helm template language 跟 go template language 類似
{: .prompt-tip }

## default function
是非常常使用的 function，該 function 可以指定一個預設的 value 在 template 裡面。例如：

{% raw %}
```yaml
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```
{% endraw %}

# 談談 templates/_helper.tpl 檔案
- 最主要是設計來讓所有 template 都可以 re-use 的字段
- 先 define 一個 block 名稱 -> 在 template 裡面透過 template 關鍵字或 include 關鍵字找到 _helper.tpl 內定義的 block 名稱，引入到該 template 裡面

## 具體實踐
1. 先建置一個 _helper.tpl 檔案，並定義 fullname 區塊
2. 新增一個 configmaps template 並定義 include 關鍵字，使其能引用 _helper.tpl 的值

### _helper.tpl
{% raw %}
```
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```
{% endraw %}

### configmaps.yaml
{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fair-worm-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default (printf "%s-tea" (include "fullname" .)) }}
  food: "PIZZA"
```
{% endraw %}

### 執行 helm 指令查看生成的 object 樣子

```bash
helm template demo .
```

![](/commons/image/20220818/000_helm.png)

> demo 可以隨意，這個參數所代表的意義是 chart name
{: .prompt-info }

# 常用 function 整理

## Date 相關的
- now 取得當前的時間，常用來搭配其他時間 function
- date 格式化時間

> Golang 的 time 套件在 format 時間字串時有自己定義的做法。他不像其他語言用 %D 或 %Y 之類的代碼，而是直接用一個自訂標準時間，它的規律就像計數 12345 那樣。
{: .prompt-warning }

### 例如宣告以下的 function，我們會得到 UTC+8 的標準 24 小時制當前時間
{% raw %}
```yaml
{{ now | date "2006-01-02 15:04:05 +0800" }}
```
{% endraw %}


## String 相關的
- contains，用來測試字串內有無包含特定的字，返回 true 或 false
  - 範例 `contains "cat" "catch"`，返回 `true`
- quote 和 squote，把 String 用雙引號(quote)或單引號(squote)包起來
  - 範例 `"hello" | quote` ，返回 `"hello"`
- indent，會把字串用空格縮排，nindent 跟 indent 一樣，但會在一開始先空一行
  - 範例 `indent 4 "hello"` 返回 `"    hello"`
- replace，取代字串
  - 範例 `"I have a pen" | replace " " "-"` 返回 `"I-have-a-pen"`

## 其他參考
<https://helm.sh/docs/chart_template_guide/function_list/>


# flow control 重點摘要

## 甚麼情況會判斷為 false
1. 直接給定 boolean 為 false 時
2. 當數字型別為 0 時
3. 空字串 `""` 時
4. 出現 `nil` 關鍵字時
5. 出現空的 collection 時(無論是 map / slice / tuple / dict / array)

## 排版問題
在 template engine 渲染過程，會把 {% raw %} `{{ 中間內容 }}` {% endraw %} 大括號對的內容刪除並且保留其空白，而最後渲染出來就會變成多了空格，文件舉例

### 當我們給定一個這樣的 configmaps.yaml
{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{ end }}
```
{% endraw %}

### 渲染結果會變成
{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telling-chimp-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"

  mug: "true"
```
{% endraw %}

> 可以看到 food 和 mug 之間多了一行空白
{: .prompt-tip }

### 這時候 helm 提供的作法是去除空白，在大括號後加上 `-` 符號來告訴 template engine 把空格處理掉，這邊要注意的是，在 helm 裡面，渲染後的換行就等同於空格，因此我們可以改成
{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{- end }}
```
{% endraw %}

{% raw %}
> 注意，{{- 代表是清除左邊的空格，而 -}} 代表是去除右邊的
{: .prompt-warning }
{% endraw %}
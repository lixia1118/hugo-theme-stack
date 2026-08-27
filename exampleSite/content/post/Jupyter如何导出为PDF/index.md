+++
author = "Joseph"
title = " Jupyter Notebook 如何导出为 PDF或其他格式？"
date = "2026-08-27"
description = ""
slug= "jupyter-pdf"
image= ""
categories = [
    "Python"
]
tags = [
    "Python"
]
+++
Jupyter Notebook 是科研、数据分析和机器学习中非常常用的工具。它最大的特点是可以在同一个文件中同时保存 Python 代码、运行结果、数学公式、图表以及 Markdown 文字说明。但 `.ipynb` 文件本身并不特别适合直接分享，vscode用户在导出为PDF或其他格式时经常遇阻。对此，本文介绍几种比较实用的方案。

## 一、直接通过 Jupyter Notebook 导出 PDF

最直接的方法是使用 Jupyter Notebook 自带的导出功能。不过这种方式通常依赖 LaTeX，Jupyter 可以先把 Notebook 转换成 LaTeX，再生成 PDF。

### 操作步骤

打开需要转换的 Jupyter Notebook，在菜单中依次选择：
`File→ Download as / Save and Export Notebook As→ PDF`

系统随后会调用 LaTeX，将当前 Notebook 转换为 PDF。

如果你的 LaTeX 环境已经配置好，那么基本上点击几下鼠标即可生成 PDF。但是如果电脑没有安装 LaTeX，或者缺少相应的 LaTeX package，导出过程中就可能出现错误。

---

## 二、使用 nbconvert 直接转换为 PDF

对于使用vscode或者其他IDE编辑的`ipynb`文件，可以在终端中使用`nbconvert` 命令，可转换的文件格式包括：

* PDF
* HTML
* Markdown
* LaTeX
* reStructuredText
* Python Script
* WebPDF


### 第一步：安装 nbconvert

如果没有安装，可以运行：

```bash
pip install nbconvert
```

如果使用 Conda，也可以：

```bash
conda install nbconvert
```
值得注意的是，电脑必须安装 [pandoc](https://pandoc.org/)

### 第二步：打开终端

进入`.ipynb`所在目录，在空白处右键点击`在终端中打开`

### 第三步：运行转换命令

运行`jupyter nbconvert --to pdf [文件名].ipynb`

官方文档给出的基本语法也是：

```bash
jupyter nbconvert --to FORMAT notebook.ipynb
```

其中 `FORMAT` 替换为希望导出的格式即可。


### 注意

1. `--to pdf` 默认通过 LaTeX 生成 PDF，因此仍然需要本地存在相应的 TeX 环境
2. 上述生成的PDF对长代码换行显示并不友好，只适合简单短代码


## 3. 先转换成 HTML，再打印为 PDF

这是我比较推荐普通用户使用的方法。如果你不想折腾 LaTeX，可以把 Notebook先转换成 HTML，再利用浏览器打印成 PDF。相比直接通过 LaTeX 生成 PDF，这种方法通常更加稳定。

### 第一步：把 Notebook 转成 HTML

打开终端，进入 `.ipynb` 所在目录，然后运行`jupyter nbconvert --to html [文件名].ipynb`，转换完成会生成`[文件名].html`，其中代码、Markdown、运行结果以及大部分图表都可以比较完整地保留下来。

### 第二步：打印为 PDF

以 Chrome 或 Edge 为例，按`Ctrl + P`然后选择`Save as PDF`即可。


## 4. 特别推荐的方法：WebPDF

WebPDF 的逻辑更接近`Notebook→ HTML→ Chromium→ PDF`，因此可以在一定程度上避开复杂的 LaTeX 环境。对于代码、表格和图形较多的 Notebook，这个方案比 LaTeX PDF 更省心也更友好。

在终端中输入：

```bash
jupyter nbconvert --to webpdf [文件名].ipynb
```

## 5. 我最推荐哪种？

对于很多 Notebook 来说，里面可能包含：

* 很宽的 DataFrame；
* matplotlib 图片；
* seaborn 图片；
* HTML 输出；
* 中文字符；
* 很长的代码；
* 特殊 Unicode 字符

这些内容直接经由 LaTeX 转换为PDF后容易产生截断，难以阅读。浏览器本身就是一个非常成熟的 HTML 渲染器，因此更建议直接转为HTML或者经过webpdf转为PDF。


## 参考资料

1. [7 Effective Ways to Download Jupyter Notebooks as PDF](https://www.pdfgear.com/pdf-converter/how-to-download-jupyter-notebook-as-pdf.htm)

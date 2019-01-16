---
layout: post
title: "手写字符采集：采集表生成"
date: 2019-01-13 20:50:06 +8000
categries:
    - NLP
tags:
    - LaTex
    - Python
---

对一种文字脱机手写字符识别，通常需要准备大量的手写样本来进行特征提取与训练。对于使用范围较广的文字而言，数据的获取比较简单，通常在网上就能直接找到比较成熟的手写数据集。但对一门新的文字进行检测与识别研究时，由于一些原因，我们往往很难获得符合要求的手写数据集，这个时候就需要我们自己动手去采集手写数据。本文针对手写字符的采集，使用LaTex配合Python的Jinja2模板引擎构建了一个简单的手写字符采集表。

# 样式设计

对于采集表，一般情况下，我们需要一个表头来存放采集表的相关信息（页码、标题、备注等等）和一个书写区域来存放参考字符和书写者的所书写的字符。同时为了使得表格能够被程序自动化处理，我们往往需要为每一页设计一个标识区域，该区域存放采集表当前页的标识信息（版本信息与页码），为此，我们将标识信息以二维码的形式放置在表头的左侧。对于汉字或其他相对比较复杂的亚洲字符而言，单元格的大小通常应当大于1cm，推荐的值是1.3cm-1.5cm。单元格个小于1cm时，书写体验较差，复杂的字符将有可能溢出单元格，而大于1.5cm的单元格又会造成纸张的浪费。

![handwritten collection table draft](/assets/pic/handwritten-collection-table-draft.jpg)

我们以A4纸为底稿，将单元格的宽度设为1 (cm)，整个书写区域的大小（HxW）为18x14，实际共能采集9x14=126个字符。将表头的高度设置为1.5，二维码的高度为表头高度的0.9，即1.275。最后再将整个采集表放大1.34倍。一个完整的采集表页面如下图所示。

![handwritten collection table](/assets/pic/handwritten-collection-table-ready.jpg)

# Tex模板

对于自动化的文档生成，首选当然是Tex排版系统（虽然语法丑到爆，但架不住包多，功能多）。现有的Tex排版系统发行版主要有：[MacTeX][]、[TeXLive][]、[MiKTeX][]以及面向中文的[CTex][]。其中[MacTeX][]和名字一样主要是面向苹果的Mac OS，[MikTex][]仅适用于Windows平台，而[CTex][]的最近一次更新还是在2012年，年久失修，同时默认的中文设置会造成各种谜一般的问题。而[TexLive][]则是全平台制霸，因此本文使用[TeXLive][]作为默认的Tex系统。这里选用的是TeXLive 2018，其可以从中科大获取[镜像][TeXLive-201804-USTC]安装。

## 导言区设置

- 默认字体大小为11pt，纸张大小为A4
- tikz 以画图的形式绘制整个采集表
- qrcode 将文本转换为二维码图像
- fontspec 默认字体环境配置
- xeCJK 为XeLaTex提供中日韩字体环境支持

字体设置应当为字体名称，若为文件名（如：xxx.ttf），则必须为相对路径。为了防止出错，应当保证所有的资源与tex源文件均处于同一个工作目录下。这里设置默认中文字体为`Noto Sans CJK SC`，英文字体为`Times New Roman`，参考字符字体为`gzywzt.ttf`（这里将其定义为`referencefont`）。同时导入tikz数据计算库，后续直接使用tikz进行数学计算。

```tex
\documentclass[11pt, a4paper]{article}
\usepackage{tikz}
\usepackage{qrcode}
\usepackage{fontspec}
\usepackage{xeCJK}

\setCJKmainfont{Noto Sans CJK SC}
\setCJKfamilyfont{referencefont}{gzywzt.ttf}
\setmainfont{Times New Roman}
\usetikzlibrary{math}
\pagestyle{empty}
```

接下来设置全局变量

- `\concat` 将两个字符串连接起来。
- `\tikzmath` 计算全局变量

```tex
% string concatenation
\newcommand{\concat}[2]{#1#2}
% set variables
\newcommand{\scaleratio}{1.35}
\newcommand{\headerwidth}{14}
\newcommand{\headerheight}{1.5}
\newcommand{\tablewidth}{14}
\newcommand{\tableheight}{18}
\newcommand{\tabletitle}{手写字符采集表}
\newcommand{\tablecomment}{请使用黑色签字笔填写}
\tikzmath { % compute global variables
	\commentwidth = \headerwidth / 2;
	\qrwidth = \headerheight * 0.85 * \scaleratio;
	\qrX = \headerheight / 2;
	\qrY = \headerheight / 2 + \tableheight;
}
```

## 采集表绘制

首先将外层tikzpicture的remember picture与overlay设置为true，使得其可以在整页上进行绘图。随后以当前页面的中点为锚点绘制采集表。最好在tex文件中使用Unicode字符串表示将要绘制的字符，可以有效避免某些字符由于字体问题在编辑时无法正常显示的问题。同时对于某些字体错误或不存在字体文件的字符，可以以贴图的形式进行绘制。

```tex
\begin{document}
 \begin{tikzpicture}[remember picture, overlay] % draw page
        \draw (current page.center) node {
            \begin{tikzpicture}[remember picture=false, overlay=false, scale=\scaleratio]
                % draw frame
                \draw[very thick] (0, 0) rectangle (\tablewidth, \tableheight + \headerheight);
                % draw header
                % draw QRCode with page info embeded
                \draw (\qrX, \qrY) node {\qrcode[height=\concat{\qrwidth}{cm}]{page info}};
                % draw table title
                \draw (\headerheight, \tableheight + \headerheight / 2) node[above right] {\LARGE \tabletitle};
                % draw table comment
                \draw (\headerheight, \tableheight + \headerheight / 2) node[below right] {\small
                    \begin{minipage}{\concat{\commentwidth}{cm}}
                        \tablecomment
                    \end{minipage}
                };
                % draw page info
                \draw (\headerwidth, \tableheight) node[above left] {\small page info};
                % draw grid
                \clip (0, 0) rectangle (\tablewidth, \tableheight);
                \draw[dashed, black!25!white] (0, 0) grid (\tablewidth, \tableheight);
                \draw[dotted, black!35!white, xshift=0.5cm, yshift=0.5cm] (-1, -1) grid (\tablewidth, \tableheight);
                \draw[very thick] (0, 0) rectangle (\tablewidth, \tableheight);
                % draw reference characters
                \draw (1.5, 0.5) node {\CJKfamily{referencefont} \LARGE \char"AC01};
            \end{tikzpicture}
        };
    \end{tikzpicture}
\end{document}
```

## 模板化

随着字符总量的增多以及字符种类的增多，手动修改文件添加字符费时费力。对此，我们可以将原始的Tex文件改造为一个模板文件，使用Jinja2模板引擎依据不同的字符集和字体进行渲染得到最终的Tex文件。这里我们做出以下规定：

- `%%` 表示控制语句
- `%#` 表示Jinja注释
- `\var{...}` 表示输出当前变量的值
- `\block{...}` 表示开始一段语句块

更多关于Jinja的知识，参考[官方文档][Jinja]。

首先我们需要在Tex文件头部定义一个YAML Front，在里面存储模板文件所用到的所有变量的默认值。

```tex
% ---
% # YAML header, default values for template
% title: Handwritten Character Acquisition
% comment: Please write with a black pen
% date: 2019-01-11 09:58
% version: mfu-ref
% layout:
%   scale: 1.3
%   padding: 0.7,  # physic size (cm)
%   logic_written_area: [9, 14]  # h x w, logic ratio -> acquisition area: 18 x 14
%   logic_header_area: [1.5, 14]
%   qrcode: true
% fonts:
%   global: Noto Sans CJK SC
%   reference: Noto Sans CJK SC
% ---
```

在对Tex模板文件进行解析时，其YAML Front必须位于文件头部。我们通过判断第一行是否为`% ---`判断文件头开始，并逐步向下解析文件，并依次取出文件头部的`%`，直至再次遇到`% ---`。将去除`%`后的字符串丢给PyYAML进行解析得到默认配置。

接下来，模板引擎对传入的`data -> {'raw': [data1, ..., dataN]}`进行切片，分别对每一个数据切片渲染一个页面。这里不要使用Jinja的`slice`函数，其实质上是将数据均分为N份。

```tex
    %% for page_id in range(num_pages)
    %%     set chars = data.raw[page_capacity * page_id: page_capacity * (page_id + 1)]
    %%     set page_signature = ('v%s,p%s' | format(version, page_id))
    %%     set chars = data.raw[page_capacity * page_id: page_capacity * (page_id + 1)]
    %%     set page_signature = ('v%s,p%s' | format(version, page_id))
    % pages#\var{num_pages}, chars#\var{num_chars}
    \begin{tikzpicture}[remember picture, overlay] % draw page#\var{loop.index} chars#\var{chars | length}
        \draw (current page.center) node {
```

# XeLaTex编译

得到最终的Tex文件后，我们需要使用XeLaTex进行编译`xelatex -halt-on-error xxx.tex`。一般情况下，这里需要编译两次才能得到正确的PDF文件。重要的一点，一定要确保所需要的资源文件与Tex文件处于同一个工作目录下，不然很容易出现一堆莫名其妙的错误。

# 后记

到目前为止，我们构建了一个简单的手写字符采集表，其全部代码见仓库 [wkyo/handwritten-character-acquisition][repo]


[MacTex]: http://www.mactex.com/
[TeXLive]: https://www.tug.org/texlive/ 
[MiKTeX]: http://www.miktex.org/
[CTex]: http://www.ctex.org/HomePage
[TeXLive-201804-USTC]: https://mirrors.ustc.edu.cn/CTAN/systems/texlive/Images/texlive2018-20180414.iso
[Jinja]: http://jinja.pocoo.org/
[repo]: https://github.com/wkyo/handwritten-character-acquisition
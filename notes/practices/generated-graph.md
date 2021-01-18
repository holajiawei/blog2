---
title: 像Markdown一样使用代码绘制依赖图、流程图、时序图
description: 
published: true
date: 2021-01-18T13:38:54.291Z
tags: graphviz, mermaid
editor: markdown
dateCreated: 2021-01-18T13:38:54.291Z
---

> 使用Markdown书写带格式的文档对不懂艺术的程序员来说已经是极大的简化了书写文档的复杂程度，有时我们还需要绘制UML、架构图等等，如果真的实际操作起Visio、OmniGraffle、Sketch等工具来画图，感觉效率和美难以两全。所以我去寻找有没有像Markdown类似使用简单的格式语言来便捷的生成美观的图像。

> yWorks(付费) Graphviz Mermaid WebSequenceDiagrams Nomnoml都是可以满足要求的工具
{.is-success}

> 我个人的选择是 **Nomnoml** 和 **Mermaid**
{.is-info}

## Graphviz <http://www.graphviz.org/>
这个我第一次知道是因为我在使用golang的debug pprof工具时才了解到这款非常优秀的图像代码生成工具，后面才了解到在机器学习领域用于决策树绘制早已非常流行。
Graphviz使用一门叫做dot的简化语言来绘制，具体语法就在这里不赘述了。其大致风格如下:
~~~
strict graph { 
  a -- b
  a -- b
  b -- a [color=blue]
} 
~~~
使用在线编辑工具WebGraphviz<http://webgraphviz.com/>呈现如下
![](https://javatuchuang.oss-cn-shanghai.aliyuncs.com/img/20210118213807.png)
WebGraphviz上面有很多很强大的例子，我表示对于追求效率和美观的我来说这个可能强大但不够便捷


## yWorks <https://www.yworks.com/yed-live/>
### 优点:
- Layout比较多
- 比较好的交互体验
- 开发者支持

### 缺点:
- 付费
- 支持的画图类型较为单一，交互、时许图画起来比较困难



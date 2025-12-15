---
title: 保存文章至Obsidian的解决方案
description: 用 WebInk 插件一键把网页正文和元数据导出成 Markdown，带模版的 Obsidian 收藏流程记录。
pubDate: 2025-12-15 22:16:00
---
## 需求
最近学习了一些前端原理相关的知识，看到了许多特别好的文章，想要将它们保存下来，记录到我的Obsidian中，方便以后有用到的时候再看。

保存的内容必须包含正文、图片。另外最好能有文章的一些元数据信息，例如创建时间、编辑时间、作者、平台等。

## 解决方案
找到一个Chrome插件可以实现这个能力：[WebInk](https://chromewebstore.google.com/detail/webink-intelligent-web-to/lhifbnmampdmdadbhpeeoikkljhiaohn)

这个插件可以将页面的正文内容提取为markdown，并且还可以通过模版功能，自定义输出的markdown的格式。

## 示例
以知乎上的一篇文章为例：
1. 安装[WebInk](https://chromewebstore.google.com/detail/webink-intelligent-web-to/lhifbnmampdmdadbhpeeoikkljhiaohn)插件。
2. 打开知乎文章： https://zhuanlan.zhihu.com/p/78362028
3. 点击WebInk插件。这时插件会自动打开侧边栏，并且展示markdown格式的文章。

到这一步我们就可以将内容复制到Obsidian中了。但我还希望能够保留文章的元数据信息，可以通过模版功能做到。

1. 点击侧边栏右上角的齿轮按钮，进入配置页面。
2. 在下边可以看到我们的模版列表，点击`Basic Frontmatter`的`Edit`。
3. 这时我们可以编辑模版以添加一些元数据信息了，以下是我的模版
```
---
title: {{title}}
url: {{url}}
publishedTime: {{publishedTime}}
author: {{byline}}
site: {{siteName}}
type: 收藏好文
---
```
4. 然后点击`Update Template`以保存。
5. 这时我们回到知乎文章的页面，点击左上角的selector，选取模版，可以看到按照我们的模版格式生成了markdown。
6. 复制markdown并粘贴到obsidian中。

大功告成！

## 可能的改进
1. 流程里仍然有手动操作、可以被自动化的部分，比如复制markdown、粘贴的过程。是不是可以搞个工作流把这步也去掉了。
2. 图片链接仍然是知乎的，是不是可以存到我自己的云上。
---
title: hexo博客搭建日记
date: 2014-03-01 22:07:41
categories: 
- blog
tags: 
- hexo
- blog
description: 百度BAE要开始收费了，想换个平台，搜到了jekyll，屡次尝试后进展缓慢，错误不断，之间搜到了hexo，体验做得很好。
keywords: hexo scaffolds,hexo配置

---
>百度BAE要开始收费了，想换个平台，搜到了jekyll，屡次尝试后进展缓慢，错误不断，之间搜到了hexo，体验做得很好。


<!--more-->

```
title:	  #文章页面上的显示名称，可以任意修改，不会出现在URL中
date:	   #文章生成时间，一般不改，当然也可以任意修改
categories: #文章分类目录，可为空
tags:	   #文章标签，可为空，多标签请用格式[tag1,tag2,tag3]
description: 
keywords: SEO用
---
正文部分
```
### tips:


1. 上面每项冒号后面都要有**空格**,所有标点必须为英文标点，hexo对空格，符号，大小写很严格，我就犯了categories后面的冒号写成了中文状态下的冒号
2. 如果是在命令行下通过`hexo new 'hello world'`创建新文章，默认会加上备注信息，在目录`hexo\scaffolds`下可以找到
3. 
`post.md`文件，可在其中修改项，默认为：
```
title: {{ title }}
date: {{ date }}
categories: 未分类
tags:
---
```
如果不是，就要自己加上备注信息，方便管理。




折腾了jekyll，多数教程都是linux下的，在windows下安装实在蛋疼。
遇到了下面一些问题：

每次修改本地文件后，需要`hexo generate`才能保存。每次使用命令时，都要在根目录`H:\hexo`下。

默认主题是`landscape`
更换主题后，generate出错
解决：hexo clean
清除缓存文件

```
[error] { rawMessage: 'Unable to parse.',
  parsedLine: 3,
  snippet: 'categorie：',
  parsedFile: null,
  message: 'Unable to parse.',
  domain: 
   { domain: null,
     _events: { error: [Function] },
     _maxListeners: 10,
     members: [],
     _disposed: true },
  domainThrown: true }
```
错误信息：categorie 冒号出错

所用主题代码块不加语言格式，自动高亮


主题色彩路径：
`D:\Documents\hexo\themes\pacman\source\css\_base\variavle.styl`

_base文件夹内还有其他配置文件


该颜色的过程中发现了一个有趣的网址：
[http://www.color-hex.com/](http://www.color-hex.com/)

主页上还有：
    `Latest Users Favourite Colors`
列表里的颜色都很漂亮
    
~~本文用[Rock! Markdown Editor](http://blog.jsfor.com/skill/2013/08/23/rock-markdown-editor-develop-notes/)编辑
附上github上的[地址](https://github.com/superRaytin/nodeLab/tree/master/client/Rock%20MarkDown%20Editor)
除了启动比较慢，编辑状态下的颜色，字体大小不好调节外，其他都很好，实时预览响应很快，win平台下比较好的一款软件了。~~
2015-06-03更新：
目前使用MarkdownPad
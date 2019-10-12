---
title: 使用Hexo+GitHub搭建静态博客教程
cover: /images/hexo-blog-building/poster.png
categories: 
    - 其他
tags:
  - Hexo
date: 2019-10-12 10:41:28
---
使用[Hexo](https://hexo.io/zh-cn/docs/)搭建静态博客

- 依赖：
  - Nodejs
  - Git
  - GitHub

**如果不会yml文件，可以狠狠的[点这里](http://www.ruanyifeng.com/blog/2016/07/yaml.html)**

## 安装

``` shell
$ npm install -g hexo-cli
$ # 或者
$ yarn global add hexo-cli
```

## 初始化
``` shell
$ hexo init <folder>
$ cd <folder>
$ npm install
```
新建完成后，指定文件夹的目录如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

- _config.yml是网站的[配置文件](https://hexo.io/zh-cn/docs/configuration)
- scaffolds: 模版 文件夹。当您新建文章时，Hexo 会根据 scaffold 来建立文件。
Hexo的模板是指在新建的文章文件中默认填充的内容。例如，如果您修改scaffold/post.md中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。

- source
资源文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。

- themes
主题 文件夹。Hexo 会根据主题来生成静态页面。

## 布局（Layout）

| 布局  | 路径 |
|-------|------|
| post | source/_posts |
| page | source |
| draft | source/_draft |

## 文件名称
```Hexo``` 默认以标题做为文件名称，但您可编辑 ```new_post_name``` 参数来改变默认的文件名称，举例来说，设为 ```:year-:month-:day-:title.md``` 可让您更方便的通过日期来管理文章。

| 变量 |  描述 |
|---|---|
| :title | 标题（小写，空格将会被替换为短杠）|
| :year | 建立的年份，比如， 2015 |
| :month | 建立的月份（有前导零），比如， 04 |
| :i_month | 建立的月份（无前导零），比如， 4 |
| :day | 建立的日期（有前导零），比如， 07 |
| :i_day | 建立的日期（无前导零），比如， 7 |

## 草稿

刚刚提到了 ```Hexo``` 的一种特殊布局：```draft```，这种布局在建立时会被保存到 ```source/_drafts``` 文件夹，您可通过 ```publish``` 命令将草稿移动到 ```source/_posts``` 文件夹，该命令的使用方式与 ```new``` 十分类似，您也可在命令中指定 ```layout``` 来指定布局。

``` shell
$ hexo publish [layout] <title>
```

## 模版（Scaffold）
在新建文章时，```Hexo``` 会根据 ```scaffolds``` 文件夹内相对应的文件来建立文件，例如：

``` shell
$ hexo new photo "My Gallery" # photo是模版文件名称
```

变量 | 描述
---|---
layout | 布局
title |	标题
date | 文件建立日期


## Cli
新建页面（page）、文章（post）、草稿（draft）
```shell
$ hexo new [layout] <title>
```

# 开始写作
``` shell
$ hexo new post my-post # /source/_posts/my-post.md
$ # 为了更好的归档可以像下面那样
$ hexo new post :2019-:10-:12-:diary.md
```
生成的文件长这样：
```yaml
---
title: ':2019-:10-:12-:diary.md'
tags:
  - Diary
date: 2019-10-12 11:25:29
---
```
两个三横线中间是什么呢？可以狠狠的[点这里](https://jekyllrb.com/docs/front-matter/)

### 以下是预先定义的参数:

参数  | 描述  | 默认值
---|---|---
layout | 布局
title | 标题
date | 建立日期 | 文件建立日期
updated | 更新日期 | 文件更新日期
comments | 开启文章的评论功能 | true
tags | 标签（不适用于分页）
categories | 分类（不适用于分页）
permalink | 覆盖文章网址
keywords | 仅用于 meta 标签和 Open Graph 的关键词（不推荐使用）	

### 分类和标签
```yaml
categories:
- Diary
tags:
- PS3
- Games
```

## 静态资源

### 少量资源

图片、CSS、JS 文件等。比方说，如果你的Hexo项目中只有少量图片，那最简单的方法就是将它们放在 source/images 文件夹中。然后通过类似于 ![](/images/image.jpg) 的方法访问它们

### 文章资源文件夹

可以通过将 config.yml 文件中的 post_asset_folder 选项设为 true 来打开。
```yaml
post_asset_folder: true
```
当资源文件管理功能打开后，```Hexo```将会在你每一次通过 ```hexo new [layout] <title>``` 命令创建新文章时自动创建一个文件夹。这个资源文件夹将会有与这个文章文件一样的名字。将所有与你的文章有关的资源放在这个关联文件夹中之后，你可以通过相对路径来引用它们，这样你就得到了一个更简单而且方便得多的工作流。

## 部署到GitHub
Hexo 提供了快速方便的一键部署功能，让您只需一条命令就能将网站部署到服务器上。

- 首先安装部署工具

```shell
$ npm install hexo-deployer-git --save
```

- 修改部署配置```_config.yml```

```yaml
deploy:
  type: git
  repo: <repository url> # 远程仓库地址
  branch: [branch] # 要发布到哪个分支
  message: [message] # commit message
```

- 发布

```shell
$ hexo deploy
```

---
title: 基于Hexo搭建个人博客
date: 2017-09-17 10:28:04
tags:
 - hexo
categories:
 - 开发工具
---
[Hexo](https://hexo.io/zh-cn/)是一个快速、简洁且高效的基于`Node.js`的博客框架。这里简单介绍下基于Hexo搭建博客的步骤。
## 安装Hexo 
在安装Hexo之前，首先需要安装[Node.js](https://nodejs.org/en/)和[Git](https://git-scm.com/)，具体的安装过程可参考[官方文档](https://hexo.io/zh-cn/docs/)。
然后就是安装Hexo：
``` shell
npm install -g hexo-cli
```
安装完成后，需要建站：
``` shell
hexo init blog
cd blog
npm install
```
上述代码创建了`blog`站点，在`blog`目录下是如下内容：
``` shell
.
├── _config.yml #网站的配置信息，例如：标题、语言等信息
├── package.json 
├── scaffolds #文件模板
├── source 
|   ├── _drafts
|   └── _posts #文章内容
└── themes #网站主题
```
更详细的介绍，可参见[官方文档](https://hexo.io/zh-cn/docs/setup.html)

<!-- more -->

## Next主题
Hexo有很多可选的[主题](https://hexo.io/themes/)和[插件](https://hexo.io/plugins/)，这里以[Next](https://github.com/iissnan/hexo-theme-next)为例，介绍下添加主题的步骤。

1. 首先cd到themes目录下，把Next clone到本地目录
``` shell
git clone https://github.com/iissnan/hexo-theme-next.git next
```
2. 在网站配置文件`_config.yml`中选择next主题
``` shell
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next
#theme: spfk
```
3. 然后，本地部署下
``` shell
hexo clean
hexo generate
hexo server
```
输入`http://localhost:4000`，就可以看到效果了。
接下来就是针对Next主题，添加定制化信息，详细信息可参考[Next官方文档](https://github.com/iissnan/hexo-theme-next)
Next主题本身支持多种样式，可以在主题配置文件`_config.yml`下选择不同的样式：
``` shell
# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```
### 添加文章标签页面
首先cd到blog站点目录下

1. 创建一个`tags`的页面
``` shell
hexo new page "tags"
```
2. 编辑位于`blog/source/tags/index.md`的文件，最终会显示在网站的标签页面
``` shell
title: All tags
date: 2016-10-22 13:39:04
type: "tags"
```
3. 在主题配置文件`_config.yml`，添加`tags`菜单
``` shell
menu:
  home: / || home  #主页
  about: /about/ || user  #个人简介页面
  tags: /tags/ || tags #标签页面
  categories: /categories/ || th #分类页面
  archives: /archives/ || archive #归档页面
  commonweal: /404.html || heartbeat #公益404页面
```
### 添加文章分类页面
添加分类页面的步骤和标签页面类似，首先cd到blog站点目录下。

1. 创建一个`categories`的页面
``` shell
hexo new page "categories"
```
2. 编辑位于`blog/source/categories/index.md`的文件，最终会显示在网站的分类页面
``` shell
title: All categories
date: 2016-10-22 13:50:22
type: "categories"
```
3. 在主题配置文件`_config.yml`，添加`categories`菜单
``` shell
menu:
  home: / || home  #主页
  about: /about/ || user  #个人简介页面
  tags: /tags/ || tags #标签页面
  categories: /categories/ || th #分类页面
  archives: /archives/ || archive #归档页面
  commonweal: /404.html || heartbeat #公益404页面
```

---

添加好标签和分类页面后，我们怎么对文章进行分类那？很简单，只要在文章的顶部添加`tags`和`categories`信息，Hexo就会把文章添加到对应标签和分类下面。如下所示：
``` md
---
title: 文章标题
date: 2016-10-16 09:12:43
tags:
 - tag1
 - tag2
categories: 
 - category1
 - category2
---
文章内容... 
```
### 添加个人简介页面
与添加标签、分类页面类似，首先cd到blog站点目录下。

1. 创建一个`about`的页面
``` shell
hexo new page "about"
```
2. 编辑位于`blog/source/about/index.md`的文件，最终会显示在网站的`about`页面
``` shell
---
title: 博主简介
date: 2016-04-06 22:23:25
---

# 简介
姓名：leon
QQ : 1031747903
```
3. 在主题配置文件`_config.yml`，添加`about`菜单
``` shell
menu:
  home: / || home  #主页
  about: /about/ || user  #个人简介页面
  tags: /tags/ || tags #标签页面
  categories: /categories/ || th #分类页面
  archives: /archives/ || archive #归档页面
  commonweal: /404.html || heartbeat #公益404页面
```
### 添加404公益页面
首先，在Next主题的source目录下，建立404.html文件，内容如下：
``` html
<!DOCTYPE HTML>
<html>
<head>
  <meta http-equiv="content-type" content="text/html;charset=utf-8;"/>
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
  <meta name="robots" content="all" />
  <meta name="robots" content="index,follow"/>
</head>
<body>

<script type="text/javascript" src="http://www.qq.com/404/search_children.js" charset="utf-8" homePageUrl="你的站点地址" homePageName="回到主页"></script>

</body>
</html>
```
然后，在主题配置文件`_config.yml`，添加`404.html`配置：
``` shell
menu:
  home: / || home  #主页
  about: /about/ || user  #个人简介页面
  tags: /tags/ || tags #标签页面
  categories: /categories/ || th #分类页面
  archives: /archives/ || archive #归档页面
  commonweal: /404.html || heartbeat #公益404页面
```
### 添加评论
Hexo支持多说、[搜狐畅言](http://changyan.kuaizhan.com/)、Disqus、[友言](http://www.uyan.cc/)等多个评论系统。其中多说已挂、畅言需要对网站备案，因此我选择了Next原生支持的友言。

只要在[友言](http://www.uyan.cc/)上进行注册，获得你的用户ID，然后在主题配置文件`_config.yml`中配置下友言ID就可以了。
``` shell
# Support for youyan comments system.
# You can get your uid from http://www.uyan.cc
youyan_uid: xxx
```
### 添加文章阅读次数
这里仅介绍Next主题直接支持的方法。
#### 基于leancloud统计
在Next主题的主题配置文件`_config.xml`中有对`leancloud`的原生支持。
``` shell
# Show number of visitors to each article.
# You can visit https://leancloud.cn get AppID and AppKey.
leancloud_visitors:
  enable: true
  app_id: xxx
  app_key: xxx
```
上面的`app_id`和`app_key`需要我们在[leancloud](https://leancloud.cn)上创建应用后，在`应用 key`里面找到。
这种统计方式仅会在每篇文章的顶部展示当前的阅读次数，无法统计博客总的访问次数和人数。

#### 基于不蒜子统计
[不蒜子](http://ibruce.info/2015/04/04/busuanzi/)是个人开发的网站计数工具。Next主题也提供了原生支持，只需要按照如下配置即可：
``` shell
# Show PV/UV of the website/page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi/
busuanzi_count:
  # count values only if the other configs are false
  enable: true
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: <i class="fa fa-user"></i> 访问人数
  site_uv_footer:
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: <i class="fa fa-eye"></i> 访问次数
  site_pv_footer:
  # custom pv span for one page only
  page_pv: true
  page_pv_header: <i class="fa fa-file-o"></i> 浏览
  page_pv_footer:
```
如上所示，配置不蒜子后，我们可以看到每篇文章的访问次数、整个站点的访问人数和次数。
但是如果网站已经运行一段时间了，那么整个站点的访问人数和次数是比较离谱的（不知道怎么计算来的？），所以需要初始化访问次数。官方提供的说法是：注册登录，自行修改阅读次数。但是目前[官网](http://busuanzi.ibruce.info/)还不能注册...
## 部署网站
在本地编辑好网站后，需要上传到服务器，才能让更多的人看到，Hexo的部署很简单。
首先，在网站配置文件`_config.yml`的最后添加部署信息：
``` shell
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/ltlovezh/ltlovezh.github.io.git 
  branch: master
```
然后，通过`hexo deploy`命令，就可以部署到github上了。

## 参考文档
1. [Hexo官网](https://hexo.io/zh-cn/)
2. [Hexo插件](https://hexo.io/plugins/)
3. [Hexo主题](https://hexo.io/themes/)
4. [Next主题](https://github.com/iissnan/hexo-theme-next)

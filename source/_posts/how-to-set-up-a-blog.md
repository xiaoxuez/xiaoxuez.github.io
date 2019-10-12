---
title: how to set up a blog
date: 2019-10-12 11:32:16
tags:
categories:
- enjoyment
---
## 博客搭建

<font size="1">哈哈哈哈哈哈哈哈，继我工作3年后，终于搭建了博客了。(虽然迟到了这么久…但还是哈哈哈哈哈哈好开心啊)</font>

### 方案

Github Pages + Travis + Hexo



### Hexo

Hexo呢，是基于Node.js的静态博客框架。反正就是，可以减小那些别的操作，让写博客的人更专注于写博客内容。 

##### 安装和使用

前提条件，已安装git、nodejs。

```
npm install -g hexo-cli 
```

安装完hexo-cli工具后，就可以使用hexo命令进行操作了。

```
hexo init [dir]
```

初始化博客。

完成之后，会产生几个文件夹，如下。

```
.
├── _config.yml
├── package.json
├── scaffolds   //模板，用于创建博文等
├── source   //博文等具体位置
└── themes  //主题
```

关于hexo的操作命令如下

```
hexo --help
Usage: hexo <command>

Commands:
  clean     Remove generated files and cache.
  config    Get or set configurations.
  deploy    Deploy your website.
  generate  Generate static files.
  help      Get help on a command.
  init      Create a new Hexo folder.
  list      List the information of the site
  migrate   Migrate your site from other system to Hexo.
  new       Create a new post.
  publish   Moves a draft post from _drafts to _posts folder.
  render    Render files with renderer plugins.
  server    Start the server.
```

使用generate可产生静态文件夹`public`，使用server命令即可在本地启动node服务，使用浏览器即可查看到效果。

如果 hexo没有server的话，是因为版本的不同，将server独立出去了，需要单独安装` npm install hexo-server --save`

关于主题的话，如果选择开源主题，可直接下载到theme文件夹即可使用。我这里使用的是[cactus](https://github.com/probberechts/hexo-theme-cactus)



### Github Pages

> [GitHub Pages](https://pages.github.com/) is designed to host your personal, organization, or project pages from a GitHub repository.
>
>  Your site is published at https://xiaoxuez.github.io/

就是github提供的服务，将github仓库部署到https://username.github.io这样的域名下。

<font size=2>这样就不用自己买服务器，自己部署服务了。</font>

使用方式呢，就是创建一个仓库，然后从仓库setting页，即可看到Github Pages设置项，可选择要部署的分支和选择主题。这里要选择一个主题。然后如果使用username.github.io命名的仓库的话，只能部署master分支，不能修改。



### Travis

[travis]([https://travis-ci.com](https://travis-ci.com/))自动化部署

首先要理解的是，博客，其实是有两套代码的，一套是源代码，一套是生成的`public`。我们可以选择将源代码放到一个分支上，`public`提交到另一个分支上。然后Github Pages部署的分支选择`public`所在的分支即可。

那么，travis可以帮助的事呢，就是当我将源代码push到远程仓库后，做一些事，比如执行`hexo generate`脚本生成`public`文件夹并push到对应分支上。这样就很完美了。

关于travis的具体使用，可见[详细文档](https://docs.travis-ci.com/)

首先，要添加`.travis.yaml`配置文件到仓库代码里。

```
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - resource # build master branch only
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  on:
    branch: resource  //构建resource分支
  local-dir: public
  target_branch: master  //发布到master分支

```

target_branch默认呢是`gh-pages`。然后因为我的仓库名为`xiaoxuez.github.io`，上文提到了我需要将静态文件提交到master分支，所以我将`hexo generate`之后的`public`文件夹发布到mster分支上了。关于deploy各字段的介绍，可见[文档](https://docs.travis-ci.com/user/deployment/pages/)

最后，就是要github授权给travis，文档在[这里](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line)。Token的存放可以选择在本地，或者配置到[travis](https://docs.travis-ci.com/user/environment-variables#defining-variables-in-repository-settings)上。

这样，就可以使用啦。

还有就是在github上的personal settings里的Applicaitons会安装travis插件，可以控制travis的权限。



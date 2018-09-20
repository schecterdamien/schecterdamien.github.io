---
title: 使用travis-ci自动部署hexo博客
date: 2018-09-20 14:03:58
tags: devops
---

&nbsp;&nbsp;&nbsp;&nbsp;今天又开始折腾了，这次是给博客加上ci功能。ci是Continuous integration的缩写，说人话就是持续集成，具体定义和解释：[维基百科](https://zh.wikipedia.org/wiki/持續整合)
### 什么是持续集成
&nbsp;&nbsp;&nbsp;&nbsp;维基上的解释又长又抽象，一句话来说，持续集成是对软件项目进行持续的自动化的编译打包构建测试发布，来检查软件交付质量的一种行为。包含自动构建，自动测试，自动部署等功能，同时还是一种行为规范，就是快速集成代码到主分支，而不是闷头在功能分支上开发，最后合并的时候发现一大堆冲突，也不好测试。一般来说，规模稍大点的公司都会自己搭建一套用于持续集成的服务，比如gitlab提供的ci功能。而对我们大多数托管在github上的项目来说，使用travis-ci就行了。
> Travis CI是在软件开发领域中的一个在线的，分布式的持续集成服务，用来构建及测试在GitHub托管的代码。

### 部署hexo博客
&nbsp;&nbsp;&nbsp;&nbsp;之前每次部署博客的时候，都需要先`hexo g`然后再`hexo d`，貌似也不是很麻烦，但是如果整个构建和部署过程需要很多的命令，一系列的环境变量配置和漫长的等待，那整个过程就会非常痛苦，不要把时间花费在重复无意义的操作上。所以，我想把这两个命令也省略掉，在代码提交的时候，自动触发构建和部署。  
&nbsp;&nbsp;&nbsp;&nbsp;还有一个问题是，我们使用`GitPage`的个人站点来搭建博客，博客的静态文件必须在`master`分支上，那我博客的源码只能放别的分支上，就叫`source`吧，当然这样别人都能看得到博客源码了，只要源码中不涉及敏感信息，看到就看到吧。  
&nbsp;&nbsp;&nbsp;&nbsp;首先需要注册travis-ci，直接用github账号授权登录就行，然后选择项目：  
{% asset_img 1.png  %}  
&nbsp;&nbsp;&nbsp;&nbsp;travis-ci构建完后需要把静态文件推到github的master分支，所以需要用户的token，travis-ci才有权限做这些动作。如果之前没有，可以生成一个：
{% asset_img 2.png  %} 
&nbsp;&nbsp;&nbsp;&nbsp;点击后就会来到下面的界面，先给 Token 起一个名字，然后为它设置一些权限，其中红框内的权限是必须的，其他可以随意添加。
{% asset_img 3.png  %} 
&nbsp;&nbsp;&nbsp;&nbsp;点击下面的`create token`按钮，就会生成一个已经赋予好权限的 token 值，然后在环境变量里面设置一个`GH_TOKEN`的变量，value就是之前的token值，后面会用到这个值
{% asset_img 4.jpg  %} 
&nbsp;&nbsp;&nbsp;&nbsp;添加`.travis.yml`配置文件到hexo 源码的根目录下，因为Travis CI 在自动构建时需要获取这些配置信息，以此来完成构建任务；这些配置信息主要包括源码分支，静态文件推送分支，仓库地址等信息。具体配置如下：  

```
language: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install


#before_script:
 # - npm install -g gulp

script:
  - hexo g

after_script:
  - cd ./public
  - git init
  - git config user.name "schecterdamien"
  - git config user.email "schecterfirst@126.com"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:master
# E: Build LifeCycle

branches:
  only:
    - source
env:
 global:
   - GH_REF: github.com/schecterdamien/schecterdamien.github.io.git
```

&nbsp;&nbsp;&nbsp;&nbsp;然后每次在source分支上push代码的时候都会触发自动构建和自动部署：
{% asset_img 5.png %} 

### 总结
&nbsp;&nbsp;&nbsp;&nbsp;travis-ci还有很多更多复杂的功能，比如可以设置定时构建部署，比如还可以在配置文件里面设置构建的时候自动跑测试，对于小小的博客来说，这些必要性不大。还有就是一些敏感的信息，比如很多第三方服务的secret、token之类环境变量的千万不能直接写在配置文件里面，因为代码已经上传到github上了，所以别人都能看到到这些信息，这些可以在travis-ci项目设置中的`Environment Variables` 设置，如上面的`GH_REF `设置。


#### 本文参考链接：  
* https://www.cnblogs.com/dmego/p/7664877.html

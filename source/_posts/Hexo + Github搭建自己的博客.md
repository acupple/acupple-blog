---
title: Hexo + Github搭建自己的博客
date: 2016-02-25 00:14:52
tags: 杂
---

之前试过wordpress，博客园等，都觉得不是很爽
1. wordpress.com国内被墙，wordpress.org又需要自己部署
2. 博客园个人感觉又比较丑。。。。
3. 其次基于程序猿喜欢折腾的本性，hexo + github感觉更高大上一些，
   其次可以方便的在github上放一些自己的代码，作为以后做项目的一些参考。

### 前期准备
1. 安装git client
    {% codeblock %}brew install git{% endcodeblock %}
2. 创建Gihub账号(username), 并且创建username.github.io的仓库
3. 安装NodeJS, npm
    {% codeblock %}brew install nodejs{% endcodeblock %}
4. 安装hexo
    {% codeblock %}npm install -g hexo{% endcodeblock %}
5. 安装hexo-deployer-git，用于deploy自己的博客到github上
    {% codeblock %}npm install hexo-deployer-git --save{% endcodeblock %}

### 下面就可以利用hexo创建自己的博客了
1. 首先使用下面的命令初始化一个example的站点
    {% codeblock %}hexo init username.github.io{% endcodeblock %}
2. 修改_config.yml文件的deployment配置，改成：
    {% codeblock %} 
  	  deploy:
        type: git
        repository: git@github.com:username/username.github.io.git
        branch: master
    {% endcodeblock %} 
3. 然后就可以在本地预览博客了: 
    {% codeblock %}
    npm install
    hexo server
    {% endcodeblock %}
4. 可以http://localhost:4000/ 中打开了

### 装扮自己的博客
网上有很多炫酷的主题：https://hexo.io/themes/， 这里我们选择一个hueman作为例子
    详细的参考https://github.com/ppoffice/hexo-theme-hueman/wiki
1. 在博客工作根目录下执行如下命令:
	{% codeblock %} git clone https://github.com/ppoffice/hexo-theme-hueman.git themes/hueman{% endcodeblock %}
2. 编辑_config.yml配置文件，修改theme
    theme: hueman
3. 配置theme/hueman中的配置文件，简单的可以重名_config.yml.example为_config.yml
    {% codeblock %}mv ./themes/hueman/_config.yml.example ./themes/hueman/_config.yml{% endcodeblock %}
4. 然后再次用hexo server命令预览自己的博客，主题变了，cool

### 发布博客到github
经过上面的准备工作，发布博客只需在工作根目录执行下面的命令
   {% codeblock %}
       hexo clean generate
       hexo deploy
   {% endcodeblock %}
然后就可以打开: http://username.github.io 来看自己的博客了。


# [Markdown Example](http://blog.zhangruipeng.me/hexo-theme-hueman/2014/12/25/Markdown%20Example/#code)

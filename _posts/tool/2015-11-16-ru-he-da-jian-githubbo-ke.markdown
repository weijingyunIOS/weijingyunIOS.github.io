---
layout: post
title: "如何搭建GitHub博客"
date: 2014-10-17 13:47:41 +0800
comments: true
category: Tools

---

### GitHub搭建博客：
##### 1 绑定用户

	 git config --global user.name "LeeWongSnail"
	 git config --global user.email 
##### 2、SSHKey

	ssh-keygen -t rsa -C "wangli_0632@163.com"
	一路回车就可以了 然后找到目录下的id_rsa.pub 文件 打开，将里面的公钥写到github的SSHKey中（setting -SSH）
	ssh -T git@github.com
	这个命令可以加密解密的验证成功与否
##### 3、ruby 的安装

	1、验证安装 bogon:.ssh leewong$ ruby --version 正确显示版本是正确的
	2、安装ruby，文章最后会提供一片博客
##### 4、安装octopress

	1、克隆至本地
	git clone git://github.com/imathis/octopress.git octopress
	2、进入octopress进行下一步操作
	cd octopress
	3、修改软件源
	gem sources -a http://ruby.taobao.org
	可以移除原来的
	gem sources -a http://rubygems.org
	修改gemfile中的软件园
	vim gemfile
	修改最上面的sources为淘宝的软件园
	4、安装bundler
	gem install bundler
	bundle install
	5、安装 octopress
	rake install
	6、生成public
	rake generate
	7、启动一个本地的服务器帮我们自动提交 自动感知文件变化自己提交
	rake preview
	这是可以在浏览器中输入http://localhost:4000/ 查看当前的
	8、在浏览器中打开http://localhost:4000/
	 之后如果比较慢可以打开文件
	sources -- _includes -- head.html 
	将jquery源修改为百度的
	<script src="//libs.baidu.com/jquery/1.9.1/	jquery.min.js"></script>
	9、修改了网页的源文件需要重新生成一次 使用rake generate
#### 5、生成博客和单页面

	1、新建博客
		rake new_post['title'] 不能输入中文
	2、修改博客内容 会自动根据时间分 文件夹
		新建的博客的路径是
		/Users/leewong/Desktop/octopress/source/_posts/2015-11-14-firststep.markdown
#### 6、部署博客到gitHub

	1、新建一个名称为username.github.io的仓库
	2、rake setup_github_pages
		Enter the read/write url for your repository
		(For example, 'git@github.com:your_username/			your_username.github.io.git)
          or 'https://github.com/your_username/			your_username.github.io')
        输入地址的格式选择第一个替换用户名
	3、rake deploy
		将本地的博客部署到gitHub
		每次添加新的博客 需要先在本地生成在部署到网络上
	注意：
		如果在http://leewongsnail.github.io/
		网页上打不开 必须验证邮箱
#### 7、将源码托管到github

	1、git add .  将所有文件添加到github上
	2、git commit -m "提交信息"
	3、git push origin source
	4、这时候就可以在github上看到source这个分支了
#### 8、界面的配置

	1、打开_config.yml文件 可以修改标题（title）副标题（subtitle）作者(author) 搜索路径(可修改为百度),下面还可以设置底部推特等按钮的显示
	twitter_tweet_button: true
	2、增加评论的功能 使用多说，首先在_config.yml中添加一项
	duoshuo_momment: true
	修改layout/post.html中的评论节点为你要新建的html文件(duoshuo_comment)
	在_include/post/下新建一个duoshuo_comment.html,将多说的通用代码复制即可
	3、更换主题，可以在github上搜索 octopress theme 可以看到很多，随便选一个
		*克隆下来
		*rake install['themename'] 安装主题
		*rake generate
		*rake deploy

#### 9、结束语

	这样github博客 就搭建完成了，这篇文章只是我在搭建过程中的笔记，希望能够帮助到你！


*参考文档

1、[多说](http://duoshuo.com/)，注意要新建一个站点才能看到后台管理

2、[ruby的安装](https://ruby-china.org/wiki/install_ruby_guide)

3、[象写程序一样写博客：搭建基于github的博客](http://blog.devtang.com/blog/2012/02/10/setup-blog-based-on-github/)

4、[使用Octopress搭建个人博客](http://wxgbridgeq.github.io/blog/2015/05/15/octopress-personal-blog/)

5、[极客学院视频讲解](http://www.jikexueyuan.com/course/887.html)

	
	
	
	

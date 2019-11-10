---
title: hexo 使用笔记
date: 2019-11-10 00:13:27
tags:
 - 工具
 - hexo
categories: 
 - 工具
---

参考 [https://www.cxyxiaowu.com/6407.html](https://www.cxyxiaowu.com/6407.html)   
防止链接挂了，自己复制一份。在上面的基础上添加了图片的配置。   

>_config.yml
>post_asset_folder: true

然后使用 `hexo new "page"` 的时候会建一个同名文件夹，可以放本文章使用的静态资源。    
使用方式 
```
{% asset_path slug %}
{% asset_img slug [title] %}
{% asset_link slug [title] %}
```

使用图片的示例  
```
{% asset_img example.jpg This is an example image %}
```

## 复制内容 
使用 hexo 搭建的步骤，删去一些内容。留下关键步骤   
node 环境之类的就不说了。

### 安装 Hexo 初始化项目

*安装 hexo*
```
npm install -g hexo-cli
```

*初始化项目*
也就是你的笔记保存的地方  
```
hexo init {name}
```

*生成网站*
```
hexo generate
```
生成 `public` 目录，部署的时候是上传这个目录，本地的时候通常不用  


*预览*
```
hexo serve
```

然后去 `http://localhost:4000` 看自己写的东西

*部署*
```
hexo deploy
```

当然，前提是要配置好 `git` 地址
去了解 `github pages`,就是建一个 `{username}.github.io` 仓库
然后去 `_config.yml` 配置 
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: {git repo ssh address}
  branch: master
```

需要额外安装插件
```
npm install hexo-deployer-git --save
``` 

最后部署执行  
```
hexo deploy
``` 

*配置站点信息*
修改 `_config.yml` 里面的配置
```yml
title: Askluolei
subtitle: '个人的学习吐槽网站'
description: '主要涉猎的编程语言为 java ，js，go 主要是服务端开发和一丢丢前端，外加一些读书笔记和吐槽'
keywords: 'java, js, vue, go, 读书, 笔记'
author: Luo lei
language: zh-CN
timezone: ''
```

### 主题 Next
修改主题  
```sh
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

然后在 `_config.yml` 里面修改配置
```
theme: next
```

然后可以在 `themes/next/_config.yml` 里面调整主题的相关配置了 

```yml
# 调整布局
scheme: Pisces

# 网站图标，图片放在 themes/next/source/images 目录，然后修改下面的配置
favicon:
  small: /images/favicon-16x16.png
  medium: /images/favicon-32x32.png
  apple_touch_icon: /images/apple-touch-icon.png
  safari_pinned_tab: /images/safari-pinned-tab.svg

# 头像 放在 themes/next/source/images/avatar.png 然后修改配置
# Sidebar Avatar
avatar:
  # In theme directory (source/images): /images/avatar.gif
  # In site directory (source/uploads): /uploads/avatar.gif
  # You can also use other linking images.
  url: /images/avatar.png
  # If true, the avatar would be dispalyed in circle.
  rounded: true
  # If true, the avatar would be rotated with the cursor.
  rotated: true

# 代码块的展示，使用 mac 类型的
codeblock:
  # Code Highlight theme
  # Available values: normal | night | night eighties | night blue | night bright
  # See: https://github.com/chriskempson/tomorrow-theme
  highlight_theme: night bright
  # Add copy button on codeblock
  copy_button:
    enable: true
    # Show text copy result.
    show_result: true
    # Available values: default | flat | mac
    style: mac

# 回到顶端 ，开启配置
back2top:
  enable: true
  # Back to top in sidebar.
  sidebar: false
  # Scroll percent label in b2t button.
  scrollpercent: true

# 阅读进度条配置
reading_progress:
  enable: true
  # Available values: top | bottom
  position: top
  color: "#222"
  height: 2px

# 右上角加上 github 地址
github_banner:
  enable: true
  permalink: https://github.com/askluolei
  title: Follow me on GitHub

# 评论插件，开启 gitalk
comments:
  # Available values: tabs | buttons
  style: tabs
  # Choose a comment system to be displayed by default.
  # Available values: changyan | disqus | disqusjs | facebook_comments_plugin | gitalk | livere | valine | vkontakte
  active: gitalk
# 然后补充相关配置
gitalk:
  enable: true
  github_id: NightTeam
  repo: nightteam.github.io # Repository name to store issues
  client_id: {your client id} # GitHub Application Client ID
  client_secret: {your client secret} # GitHub Application Client Secret
  admin_user: germey # GitHub repo owner and collaborators, only these guys can initialize gitHub issues
  distraction_free_mode: true # Facebook-like distraction free mode
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW
  language: zh-CN

# 中英文之间有空格
pangu: true

# 数学公式 
math:
  enable: true

  # Default (true) will load mathjax / katex script on demand.
  # That is it only render those page which has `mathjax: true` in Front-matter.
  # If you set it to false, it will load mathjax / katex srcipt EVERY PAGE.
  per_page: true

  # hexo-renderer-pandoc (or hexo-renderer-kramed) required for full MathJax support.
  mathjax:
    enable: true
    # See: https://mhchem.github.io/MathJax-mhchem/
    mhchem: true

# 局部刷新
pjax: true
```

### 其他插件  

*rss*
```
npm install hexo-generator-feed --save

```

*数学公式*
```
npm un hexo-renderer-marked --save
npm i hexo-renderer-kramed --save
```

*pjax*
```
cd themes/next
git clone https://github.com/theme-next/theme-next-pjax source/lib/pjax
```

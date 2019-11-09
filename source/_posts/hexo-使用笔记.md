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





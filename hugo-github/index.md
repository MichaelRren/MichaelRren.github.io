# Github Pages 结合 Hugo 搭建个人博客


## 1. 使用 GitHub Pages 创建个人域名站点
首先，需要在 `GitHub` 上创建一个项目作为博客的域名。

这部分网上资料很多，本文不再赘述，可以参考[Github Pages 搭建个人网站](https://blog.csdn.net/zhuguanlin121/article/details/118409449)。


## 2. 本机安装 Hugo 并选择主题
其次，使用 `Hugo` 作为博客的静态网站，并选择主题进行呈现。
> `Hugo` [资源库](https://themes.gohugo.io/), 本文使用 `LoveIt` 进行演示。

```shell
# 下载 hugo
brew install hugo

# 进入目录生成 hugo 项目
cd Projects/opensource
hugo new site blog

# 选择主题目录并选择主题下载
cd themes
git clone https://github.com/dillonzq/LoveIt.git

# 回到 blog 目录，创建一个博文并本地进行测试
cd ..
hugo new posts/test.md
hugo server 
```

## 3. 本地调试 Hugo 主题
与上一步相关，对 `Hugo` 进行调试。

LoveIt 主题作为一个 `stars` 众多的成功项目，其配置文件较多且较为繁琐：

1. 进入 `themes/LoveIt/example` 目录，将该目录下**所有**的文件拷贝到`opensource`目录下，并覆盖原有文件，如 `config.toml`， `assets`， `static`，`content` 等
2. 打开 `config.toml` 并按照注释进行修改即可


## 4. 关联 GitHub Pages 与 Hugo
最后是**最关键的一步**，将 `GitHub Pages` 与 调试成功后的 `Hugo` 项目进行关联并展示。

1. 在**上文提到的 `blog` 目录下**，将 `GitHub Pages` 的 `Repo` 作为 `submodule` 添加到 `blog` 目录下的 `public` 文件中（`blog` 目录可能需要 `git init`）
```shell
cd blog
git init
# public 目录为 submodule 部署的目录，该命令自动生成
git submodule add -b main git@github.com:xxx/xxx.github.io.git public
```
2. 在 `blog` 目录下构建 `Hugo` 项目，并检查 `public` 目录下的文件，最后提交
```shell
hugo -t LoveIt
cd public
git add .
git commit -m "init"
git push origin main
```

## 5. 其他功能
### 5.1 评论功能
评论功能依旧是基于 `Github Repo` 和 `utterances` 实现的。
1. 在 Github 上创建一个 `public` 的 `Github Repo`，如 `comments`
2. 修改 `config.toml`，在属性 `[params.page.comment]` 处支持 `utterances`
```toml
  ...
  [params.page.comment]
    ...
      [params.page.comment.utterances]
        enable = true
        # owner/repo
        repo = "MichaelRren/comments"
        issueTerm = "pathname"
        label = ""
        lightTheme = "github-light"
        darkTheme = "github-dark"
    ...
  ...
```
3. 打开 [utterances](https://github.com/apps/utterances) 并进行配置

> 只选择 `github.io` 对应 `Repo` 即可；<br />
此外，由于评论功能不支持在**开发环境**使用，所以调试时使用`hugo server -e production`命令进行测试。

### 5.2 搜索功能
todo


### 5.3 文章增加 Tag 和 Category
修改 `config.toml`，增加如下属性：
```toml
...
[taxonomies]
  tag = "tags"
  category = "categories"
...
```

完成后打开博文所对应的`md`文件，在**文件首部**增加如下字段：

```plain
---
title: "测试"
date: 2023-02-10T16:20:42+08:00
tags: ["hugo"]
categories: ["test"]
---
```


# 引用
[1] [Creating a Blog with Hugo and Github in 10 minutes](https://www.youtube.com/watch?v=LIFvgrRxdt4)


[2] [hugo博客使用 utterances 作为评论系统](https://cloud.tencent.com/developer/article/1834230)


[3] [Blog养成记(4) Hugo中增加tags等分类](https://orianna-zzo.github.io/sci-tech/2018-01/blog%E5%85%BB%E6%88%90%E8%AE%B04-hugo%E4%B8%AD%E5%A2%9E%E5%8A%A0tags%E7%AD%89%E5%88%86%E7%B1%BB/)

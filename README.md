# 目录结构

    ├── archetypes
    ├── assets
    ├── content
    │   └── posts
    │       └── xx.md
    ├── data
    ├── layouts
    ├── public
    ├── static
    ├── themes
    └── hugo.toml

## archetypes

在通过`hugo new xxx` 创建内容页面的时候，默认情况下`hugo`会创建`date`、`title`等`front matter`，可以通过在`archetypes`目录下创建文件，设置自定义的`front matter`。

## content

站点下所有的内容页面，也就是我们创建的`md`文件都在这个`content`目录下面。

## data

`data`目录用来存储网站用到一些配置、数据文件。文件类型可以是`yaml|toml|json`等格式

## layouts

存放用来渲染`content`目录下面内容的模版文件，模版`.html`格式结尾，`layouts`可以同时存储在项目目录和`themes//layouts`目录下。

## public

`hugo`编译后生成网站的所有文件都存储在这里面，把这个目录放到任意`web`服务器就可以发布网站成功。

## static

用来存储图片、`css`、`js`等静态资源文件。

## themes

用来存储主题，主题可以方便的帮助我们快速建立站点，也可以方便的切换网站的风格样式。

## hugo.toml

所有的`hugo`站点都有一个全局配置文件，用来配置整个站点的信息，`hugo`默认提供了跟多配置指令。

# 命令

## new
```
# 指定的目录下创建网站骨架, 静态网站是根据这些网站骨架生成的.
hugo new site [path]
hugo new site blog --format yml --force

# 在content目录下创建一篇新内容文件. path为完整的路径, 包含文件名和扩展名. 以content目录为根目录.
hugo new [path] 

# 在themes目录下生成自定义模板
hugo new theme [name]
```

# 参考文章

1. HugoLoveIt官方文档：<https://hugoloveit.com/zh-cn/>
2. <https://hcdtc.github.io/zh/docs/80-team-doc/1-doc-quickstart/2-hugo-quickstart>
3. Hugo命令: <https://www.gohugo.org/doc/commands/>
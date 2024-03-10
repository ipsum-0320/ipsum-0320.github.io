# 👏 欢迎来到我的博客

这里将记录 ipsum-0320 的成长🏃，不仅仅是技术💻。

PS:

网站是基于 Jekyll 构建的，在以 markdown 的格式添加完 post 之后，可以本地将其编译成前端代码，预览一下效果，具体的操作流程如下：

1. 安装 `ruby`（macOS 可以使用 brew）。
2. `ruby` 自带一个依赖管理工具 `bundler`，之后执行 `bundle install` 安装相应的依赖（可以考虑换源）。
3. 在本地完成编译并测试站点 `bundle exec jekyll serve`。

注意，不需要连同生成的代码一起上传，github 会自动进行编译。

博客搭建踩坑：

1. Github Pages 并不支持 jekyll-archives 插件，因此无法使用 jekyll-archives 来生成 Tag/Category。
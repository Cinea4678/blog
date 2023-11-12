---
title: 在单页HTML中使用npm包
zhihu-url: https://zhuanlan.zhihu.com/p/666410931
---

# 在单页 HTML 中使用 npm 包

在笔者印象中，上一次写单页 HTML 还是在刚开始接触前端开发，学习 HTML 和 JavaScript 的时候。自从开始用 Vue 和 React 这样的框架之后，就几乎再也没写过单页应用了。框架和 npm 给开发带来了很多便利：对于想用的包，npm install 就能下载到目录下，然后在 Vue 和 React 代码中也可以直接使用 import 或者 require 来引入和使用。打包时，webpack 和 vite 会帮我们处理好一切，包括把充斥着 require 的代码变得在浏览器里也可以正常使用。

但是当我们习惯了 npm install，再回到单页 HTML 时，就会发现下包和导入好像没有那么水到渠成了。维护良好的包会提供浏览器可用的版本——我们可以直接在 CDN 的索引上搜索链接，但是维护不善的包就需要自己手动打包浏览器可用的版本。笔者将逐一介绍两种情况下如何将包引入浏览器。

## 提供浏览器版本的包

笔者个人喜欢在 bootcdn 上搜索包的 CDN 链接。jsDelivr 官网上找的包不知为什么导入浏览器之后工作不太正常。

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112124642775.png)

搜到包之后点击就能看到链接了：

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112124716085.png)

## 不提供浏览器版本的包

如果 bootcdn 上搜不到包，那么这个包大概率没有提供官方编译的浏览器版本。这种情况下就需要我们手动使用 webpack 工具来把这个包打包成浏览器版本了。

（ps：笔者发现 jsDelivr 上能搜到这类包，但是打开它提供的 CDN 链接，发现 js 里全是 require……给人一种沙漠中看到海市蜃楼的感觉）

接下来，以包 graphology-layout 为例来撰写后续打包和引入的过程：

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112132528081.png)

首先，我们确认这个库是没有直接提供浏览器版本的；在 jsDelivr 上虽然能搜到，但是点进去是这玩意儿：

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112132627188.png)

![image-20231112132639797](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112132639797.png)

这种东西直接引入浏览器显然用不了：

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112132755361.png)

所以我们要用打包工具把他们打包和转换成浏览器能用的版本。常用的工具有 browserify 和 webpack。browserify 历史比较悠久，主打的就是在浏览器里用上 require：

![项目的主页](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112133101856.png)

Webpack 功能更强大，也更丰富。它不仅可以处理 JavaScript，还可以处理 CSS、图片和字体等资源。

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112133310943.png)

Browserify 使用起来很简单，但是 Webpack 更强大也更流行。接下来，我们会用 Webpack 来完成打包工作。

我们首先克隆需要打包的项目：

```shell
git clone https://github.com/graphology/graphology.git
cd .\graphology\src\layout\
```

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112133904024.png)

现在我们来到了这样的一个地方。它虽然被放置在 graphology 仓库内，但是确实是一个完整的 node.js 项目。接下来我们安装一下依赖：

```shell
npm i   #（或者pnpm i，看你喜欢用哪个）
```

接下来，安装 webpack 和 webpack-cli

```shell
npm install --save-dev webpack webpack-cli   #（或者pnpm，看你喜欢用哪个）
```

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112134150339.png)

用你顺手的编辑器打开这个目录，并创建一个叫做`webpack.config.js`的文件，内容如下：

```js
const path = require("path");

module.exports = {
    mode: "production",
    entry: "./index.js", // 入口文件
    output: {
        path: path.resolve(__dirname, "dist"),
        filename: "graphology-layout.min.js", // 打包产物js的名称
        library: "graphologyLayout", // 打包的库的名称，之后需要用这个名字来调用这个库
        libraryTarget: "umd",
        globalObject: "this",
    },
};
```

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112134336318.png)

完成后，我们回到命令行，输入命令就开始打包了：

```shell
 npx webpack --config webpack.config.js
```

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112134534350.png)

在 dist 目录下，我们就能找到打包的产物了：

![有内味了](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112134559757.png)

之后，把这个 js 部署到云端，或者静态文件目录下，然后在像引用一个提供官方浏览器版本的包一样引用它就好了：

```html
<head>
    <script src="/graphology-layout.min.js"></script>
</head>
<script>
    graphologyLayout.circular.assign(graph); // 用刚刚配置文件里取的库名来引用里面的东西
    const settings = graphologyLayoutForceAtlas2.inferSettings(graph);
    graphologyLayoutForceAtlas2.assign(graph, { settings, iterations: 600 });
</script>
```

让我们试验一下：

![](https://blogsources-1305284863.file.myqcloud.com/images/image-20231112135007417.png)

非常成功！可以看到，功能的实现是正常的，控制台里没有再报 js 导入相关的错误了。

## 后记

笔者在这一次的实践中之所以需要使用单页 HTML 来完成这个页面，是因为笔者用于生成页面中显示的内容的程序是用 node.js 来写的。本着不想在一次作业里套两个 node 项目的原则，笔者没有考虑使用 vite+vue3 之类的常规实践，因而有了这一篇文章。

因为对 nodejs、JavaScript 和 es6 了解都不够深入，文章部分地方可能有误，我的实践也可能并非最佳实践。希望能帮助到读者。

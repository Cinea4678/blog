---
title: 轻松写自己的TinyMCE插件
zhihu-url: https://zhuanlan.zhihu.com/p/669766159
---

# 轻松写自己的TinyMCE插件

![](https://s.c.accr.cc/picgo/1701358912-570d3c.png)

TinyMCE无疑是在线编辑器中的Top级工具，而TinyMCE强大的扩展支持也赋予了它更多可能性。在官方提供的免费（和付费）扩展中，虽然已经有了大量开箱即用的扩展，但是我们难免还是会遇到需要根据自己需求定制的情况；又或者某个插件是付费的，但是我们又不想付费。这种时候会写插件的话，就能帮大忙了。接下来就由我带大家快速学习TinyMCE的插件怎么写吧~



**版本**：我用的是TinyMCE 6，不过据我对网络上看到的旧版TinyMCE的观察，大部分API还是兼容的。

## 创建插件脚本文件

我们选一个心仪的目录，创建我们的脚本文件~ （我用的是TypeScript，读者想用js也没关系）

```
src/tools/tinymce-plugins/emoticons.ts
```

我们首先向TinyMCE（以下简称tiny）注册我们的插件，这样我们才能用它：

```typescript
import tinymce from "tinymce"

tinymce.PluginManager.add("emoticons", function (editor) {

  // 为插件注册一个按钮
  editor.ui.registry.addButton("emoticons", {
    icon: "emoticons",
    tooltip: "常用表情",
    onAction: function () {
    },
  })

  // 为插件注册一个菜蛋项
  editor.ui.registry.addMenuItem("emoticons", {
    icon: "emoticons",
    text: "常用表情",
    onAction: function () {
    },
  })

  // 返回插件的元数据
  return {
    getMetadata: function () {
      return {
        name: "Emoticons",
        url: "https://www.cinea.cc/",
      }
    },
  }
})
```

然后我们在使用tiny的地方引入我们的插件：

```typescript
import "tinymce"
import "tinymce/themes/silver"
import "tinymce/icons/default"
import "tinymce/plugins/emoticons"
import "tinymce/plugins/emoticons/js/emojis.js"
import "tinymce/plugins/table"
// 省略很多插件……
import "tinymce/plugins/quickbars"
import "tinymce/plugins/autosave"
import "tinymce/plugins/searchreplace"
import "@/tools/tinymce-plugins/emoticons.ts"  // 在这里！
```

并且像使用其他插件一样把我们插件的名称和按钮的名称加入到tiny的启动配置中：

```typescript
  plugins: {
    type: [String, Array],
    default: "table codesample wordcount image preview autosave searchreplace emoticons",
  },
  toolbar: {
    type: [String, Array],
    default:
      "undo redo | styles | bold italic | alignleft aligncenter alignright alignjustify | emoticons link codesample image | preview",
  },
```

现在我们刷新页面，已经可以看到我们的插件了：

![](https://s.c.accr.cc/picgo/1701359939-d1e4bb.png)

不过令人颇为不满的是，图标现在是个小炸弹，仿佛在对我说：爆了爆了爆了！这好吗？这不好。所以，我们去找一个好看、符合tiny风格的svg图标：

![素材来自iconfont](https://s.c.accr.cc/picgo/1701360012-908bae.png)

然后直接把svg贴到代码里，就像我这样：

![](https://s.c.accr.cc/picgo/1701360049-6ce2f1.png)

最后，在刚刚注册按钮的地方把图标也注册上就好了：

```typescript
tinymce.PluginManager.add("emoticons", function (editor) {

  // 注册我们的图标
  editor.ui.registry.addIcon("emoticons", icon)

  editor.ui.registry.addButton("emoticons", {
    icon: "emoticons",
    tooltip: "常用表情",
    onAction: function () {
    },
  })

  editor.ui.registry.addMenuItem("emoticons", {
    icon: "emoticons",
    text: "常用表情",
    onAction: function () {
    },
  })

  return {
    getMetadata: function () {
      return {
        name: "Emoticons",
        url: "https://www.cinea.cc/",
      }
    },
  }
})
```

一切正常！

![](https://s.c.accr.cc/picgo/1701360146-4cdf1f.png)

### 避雷：SVG的长宽属性需要自己设置

这里有一个雷区：tiny的工具栏（toolbar）并不会为按钮强行约束大小，这也就是说如果你的svg里没有长宽信息，或者有远远大于适当大小的长宽的话，就会出现这种情况：

![](https://s.c.accr.cc/picgo/1701360488-d75f4b.png)

可以看到，我们刚刚放进去的笑脸变成了一个丑陋、怪异的粗横线；打开devtools，发现是图标没有缩放到正常的大小，而是按照svg属性里的长宽渲染了。

解决方案也很简单，我们直接在svg里找到长宽，改成20 x 20就好了；如果没有出现长宽属性的话，也可以自己加一个。

```
...link" width="20" height="20"><path d="M511.488 118.670222a398.22222...
```

很棒，恢复正常了~

![](https://s.c.accr.cc/picgo/1701360660-7e3e27.png)

## 编写插件行为

这里我以显示一个对话框为例，因为这也是tiny插件开发中比较棘手的一个部分；事实上，如果你不需要对话框的话，这篇文章看到这里已经足够了。

首先，回顾之前的代码，我们在按钮和菜单的`onAction`方法中留了空白：

```typescript
  editor.ui.registry.addButton("emoticons", {
    icon: "emoticons",
    tooltip: "常用表情",
    onAction: function () {
    },
  })

  editor.ui.registry.addMenuItem("emoticons", {
    icon: "emoticons",
    text: "常用表情",
    onAction: function () {
    },
  })
```

现在，我们为`onAction`方法填充插件的业务逻辑：

```typescript
  editor.ui.registry.addButton("emoticons", {
    icon: "emoticons",
    tooltip: "常用表情",
    onAction: function () {
      openDialog()
    },
  })

  editor.ui.registry.addMenuItem("emoticons", {
    icon: "emoticons",
    text: "常用表情",
    onAction: function () {
      openDialog()
    },
  })
```

`openDialog`是我们接下来要定义的另一个方法；我们先从核心开始：

```typescript
const openDialog = () => {
    return editor.windowManager.open({
      title: "常用表情",
      body: {
        type: "panel",
        items: [
            // ....
        ],
      },
    })
  }
```

我们打开了一个标题叫做“常用表情”的tiny窗口。body内的内容，是按照tiny的规范排布和编写的组件，这些组件的文档可以在这个网页上找到：[Custom dialog body components](https://www.tiny.cloud/docs/tinymce/6/dialog-components/)

我们也许会在窗口里放置按钮等交互组件。处理这些交互并不是直接在组件上写`onAction`函数，而是在对话框上：

```typescript
const openDialog = () => {
    return editor.windowManager.open({
      title: "常用表情",
      body: {
        type: "panel",
        items: [
          // 略
        ],
      },
      onAction(d, details) {
        console.log(details) // 打印交互的信息
        d.close() // 关闭对话框
      },
    })
  }
```

你可以在官网查阅[对话框开发文档](https://www.tiny.cloud/docs/tinymce/6/dialog/)，根据自己的需要调用tiny的API，实现你的需求。

## 实战示例：实现一个网络表情插入工具

接下来是我个人刚刚做完的一个小插件：贴吧/B站/某音表情插入工具：

![](https://s.c.accr.cc/picgo/1701358912-570d3c.png)

代码不过寥寥一百多行，且大部分都已经在上文中出现过了；接下来，以这个小项目为例，向大家演示怎么开发一个实际可用的插件吧~



在上文的基础上，我们继续编写代码。首先我们从Emoji All上整理出感兴趣的表情和它们的名称：

```typescript
interface Emoticon {
  id: string
  name: string
  url: string
}

const tiebaEmoticons: Emoticon[] = [
  { name: "[真棒]", id: "tb-zb", url: "https://www.emojiall.com/images/60/baidu/1f44d.png" },
  { name: "[疑问]", id: "tb-yw", url: "https://www.emojiall.com/images/60/baidu/1f928.png" },
  { name: "[汗]", id: "tb-han", url: "https://www.emojiall.com/images/60/baidu/1f613.png" },
  { name: "[开心]", id: "tb-kx", url: "https://www.emojiall.com/images/60/baidu/263a.png" },
]

const biliEmoticons: Emoticon[] = [
  { name: "[哦呼]", id: "bl-oh", url: "https://www.emojiall.com/images/60/bilibili/default005.png" },
  { name: "[喜欢]", id: "bl-xh", url: "https://www.emojiall.com/images/60/bilibili/1f60d.png" },
  { name: "[大哭]", id: "bl-dk", url: "https://www.emojiall.com/images/60/bilibili/1f62d.png" },
  { name: "[大笑]", id: "bl-dx", url: "https://www.emojiall.com/images/60/bilibili/1f604.png" },
  { name: "[doge]", id: "bl-doge", url: "https://www.emojiall.com/images/60/bilibili/1f436.png" },
  { name: "[打call]", id: "bl-call", url: "https://www.emojiall.com/images/60/bilibili/1f64c.png" },
  { name: "[灵魂出窍]", id: "bl-cq", url: "https://www.emojiall.com/images/60/bilibili/1f47b.png" },
  { name: "[生气]", id: "bl-sq", url: "https://www.emojiall.com/images/60/bilibili/1f621.png" },
]

const douyinEmoticons: Emoticon[] = [
  { name: "[泣不成声]", id: "dy-qbcs", url: "https://www.emojiall.com/images/60/douyin/cnc.png" },
  { name: "[送心]", id: "dy-sx", url: "https://www.emojiall.com/images/60/douyin/1f970.png" },
  { name: "[快哭了]", id: "dy-kkl", url: "https://www.emojiall.com/images/60/douyin/1f625.png" },
  { name: "[流泪]", id: "dy-ll", url: "https://www.emojiall.com/images/60/douyin/1f622-new.png" },
]
```

可以注意到，我定义了一个叫做`Emotion`的接口，并让我们的表情数据符合这个接口。这有利于我们获得ts的语法检查和智能提示。

接下来，我们来定义一个映射：它的作用是把`Emotion`类型的表情数据转换成tiny的组件：

```typescript
const emoticonToMceComponent = (e: Emoticon): BodyComponentSpec => {
  return {
    type: "bar",
    items: [
      {
        type: "htmlpanel",
        html: `<div style="height: 40px; width: 40px"><img alt="${e.name}" src="${e.url}" style="height: 40px; width: 40px"/></div>`,
      },
      {
        type: "button",
        name: e.id,
        text: "添加",
      },
    ],
  }
}
```

这个组件结构很简单；为了获得正确的语法检查和智能提示，我首先让函数的返回类型为tiny的窗口组件类型`BodyComponentSpec`；接下来，我们定义了一个bar，bar在tiny中类似于一个flex的div容器，且容器中的元素是水平居中的。

在容器内部，我们定义了一个html元素和一个按钮。html元素里的内容是我们的表情的图片；这里我们为图片显式地限制了宽高，这是为了避免浏览器按照原始图片的大小显示图片。对按钮，我们为它定义了一个name属性，这样我们就可以通过name属性判断出用户正在点击的按钮属于哪个表情了。

```typescript
return editor.windowManager.open({
      title: "常用表情",
      body: {
        type: "panel",
        items: [
          {
            type: "label",
            label: "贴吧表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...tiebaEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
          {
            type: "label",
            label: "B站表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...biliEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
          {
            type: "label",
            label: "抖音表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...douyinEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
        ],
      },
}
```

在组件的body部分中，我们首先放置了一个panel组件；接下来，在panel组件里，我们放了三个label组件。这里的label组件并不只是一个简单的标签元素；它事实上包裹住整整一组元素，并在它们的上方标注上标签，相当于一个div。

对每组表情，我们放置了一个grid组件，这个grid和CSS中的网格布局很像，但是却是用flex实现的；这也就决定了它注定不如网格布局好用（读者试试在columns为4的grid里放3个元素就知道为什么了）。grid里我们用map方法调用我们刚刚写的映射，把表情转换成组件。



最后，我们处理一下用户点击按钮的事件：

```typescript
  onAction(d, details) {
    let e: Emoticon
    switch (details.name.substring(0, 2)) {
      case "tb":
        e = tiebaEmoticons.find((e) => e.id === details.name) ?? tiebaEmoticons[0]
        break
      case "bl":
        e = biliEmoticons.find((e) => e.id === details.name) ?? biliEmoticons[0]
        break
      case "dy":
        e = douyinEmoticons.find((e) => e.id === details.name) ?? douyinEmoticons[0]
        break
      default:
        return
    }
    editor.insertContent(`<img alt="${e.name}" src="${e.url}" style="height: 80px; width: 80px"/>`)
    d.close()
  },
```

对话框的`onAction`方法接受两个参数，第一个是一组对话框的接口，我们可以在这里操作对话框的行为，比如最简单的，把它关掉；第二个参数是交互的具体信息，建议读者开发时用`console.log`看一下实际获得的数据的格式，为了避免谬误，我就不在这里下结论了。

最后，我们调用`editor`（这是`tinymce.PluginManager.add`的时候出现的一个参数，用于操作编辑器）的API，插入我们的表情包图片HTML，结束！

![](https://s.c.accr.cc/picgo/1701362955-4d92e4.png)

完整代码如下：

```typescript
import tinymce, { BodyComponentSpec } from "tinymce"

interface Emoticon {
  id: string
  name: string
  url: string
}

const icon = `<svg t="1701352119452" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="4290" xmlns:xlink="http://www.w3.org/1999/xlink" width="20" height="20"><path d="M511.488 118.670222a398.222222 398.222222 0 1 0 0 796.444445 398.222222 398.222222 0 0 0 0-796.444445z m0-85.333333a483.555556 483.555556 0 1 1 0 967.111111 483.555556 483.555556 0 0 1 0-967.111111zM292.067556 378.709333a69.063111 69.063111 0 1 1 138.126222 0 69.063111 69.063111 0 0 1-138.126222 0z m300.657777 0a69.063111 69.063111 0 1 1 138.183111 0 69.063111 69.063111 0 0 1-138.183111 0zM275.626667 545.336889h475.249777c0 108.828444-100.067556 239.502222-240.355555 239.502222-140.231111 0-234.894222-130.673778-234.894222-239.502222z" fill="#333333" p-id="4291"></path></svg>`

const tiebaEmoticons: Emoticon[] = [
  { name: "[真棒]", id: "tb-zb", url: "https://www.emojiall.com/images/60/baidu/1f44d.png" },
  { name: "[疑问]", id: "tb-yw", url: "https://www.emojiall.com/images/60/baidu/1f928.png" },
  { name: "[汗]", id: "tb-han", url: "https://www.emojiall.com/images/60/baidu/1f613.png" },
  { name: "[开心]", id: "tb-kx", url: "https://www.emojiall.com/images/60/baidu/263a.png" },
]

const biliEmoticons: Emoticon[] = [
  { name: "[哦呼]", id: "bl-oh", url: "https://www.emojiall.com/images/60/bilibili/default005.png" },
  { name: "[喜欢]", id: "bl-xh", url: "https://www.emojiall.com/images/60/bilibili/1f60d.png" },
  { name: "[大哭]", id: "bl-dk", url: "https://www.emojiall.com/images/60/bilibili/1f62d.png" },
  { name: "[大笑]", id: "bl-dx", url: "https://www.emojiall.com/images/60/bilibili/1f604.png" },
  { name: "[doge]", id: "bl-doge", url: "https://www.emojiall.com/images/60/bilibili/1f436.png" },
  { name: "[打call]", id: "bl-call", url: "https://www.emojiall.com/images/60/bilibili/1f64c.png" },
  { name: "[灵魂出窍]", id: "bl-cq", url: "https://www.emojiall.com/images/60/bilibili/1f47b.png" },
  { name: "[生气]", id: "bl-sq", url: "https://www.emojiall.com/images/60/bilibili/1f621.png" },
]

const douyinEmoticons: Emoticon[] = [
  { name: "[泣不成声]", id: "dy-qbcs", url: "https://www.emojiall.com/images/60/douyin/cnc.png" },
  { name: "[送心]", id: "dy-sx", url: "https://www.emojiall.com/images/60/douyin/1f970.png" },
  { name: "[快哭了]", id: "dy-kkl", url: "https://www.emojiall.com/images/60/douyin/1f625.png" },
  { name: "[流泪]", id: "dy-ll", url: "https://www.emojiall.com/images/60/douyin/1f622-new.png" },
]

const emoticonToMceComponent = (e: Emoticon): BodyComponentSpec => {
  return {
    type: "bar",
    items: [
      {
        type: "htmlpanel",
        html: `<div style="height: 40px; width: 40px"><img alt="${e.name}" src="${e.url}" style="height: 40px; width: 40px"/></div>`,
      },
      {
        type: "button",
        name: e.id,
        text: "添加",
      },
    ],
  }
}

tinymce.PluginManager.add("emoticons", function (editor) {
  const openDialog = () => {
    return editor.windowManager.open({
      title: "常用表情",
      body: {
        type: "panel",
        items: [
          {
            type: "label",
            label: "贴吧表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...tiebaEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
          {
            type: "label",
            label: "B站表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...biliEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
          {
            type: "label",
            label: "抖音表情",
            items: [
              {
                type: "grid",
                columns: 4,
                items: [...douyinEmoticons.map(emoticonToMceComponent)],
              },
            ],
          },
        ],
      },
      onAction(d, details) {
        let e: Emoticon
        switch (details.name.substring(0, 2)) {
          case "tb":
            e = tiebaEmoticons.find((e) => e.id === details.name) ?? tiebaEmoticons[0]
            break
          case "bl":
            e = biliEmoticons.find((e) => e.id === details.name) ?? biliEmoticons[0]
            break
          case "dy":
            e = douyinEmoticons.find((e) => e.id === details.name) ?? douyinEmoticons[0]
            break
          default:
            return
        }
        editor.insertContent(`<img alt="${e.name}" src="${e.url}" style="height: 80px; width: 80px"/>`)
        d.close()
      },
    })
  }

  editor.ui.registry.addIcon("emoticons", icon)

  editor.ui.registry.addButton("emoticons", {
    icon: "emoticons",
    tooltip: "常用表情",
    onAction: function () {
      openDialog()
    },
  })

  editor.ui.registry.addMenuItem("emoticons", {
    icon: "emoticons",
    text: "常用表情",
    onAction: function () {
      openDialog()
    },
  })

  return {
    getMetadata: function () {
      return {
        name: "Emoticons",
        url: "https://www.cinea.cc/",
      }
    },
  }
})

```

我之后有空的话，也许会考虑完善表情数据，然后开源到GitHub，供各位感兴趣的读者直接使用。

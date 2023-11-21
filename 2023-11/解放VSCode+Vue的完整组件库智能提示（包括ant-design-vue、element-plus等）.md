---
title: 解放VSCode+Vue的完整组件库智能提示（包括ant-design-vue、element-plus等）
zhihu-url: https://zhuanlan.zhihu.com/p/667931000
---

# 解放VSCode+Vue的完整组件库智能提示

最近因为一些原因从WebStrom转回VS Code，首先感受到的就是组件库没有智能提示了：

![](https://s.c.accr.cc/picgo/1700530198-a3d0ad.png)

这能忍吗？根本不可能！接下来，我带你花三分钟找回遗失的智能提示～

首先，本篇文章适用于通过**unplugin-vue-components**自动引入组件的项目；也就是说，在你的`vite.config.js`里应当有类似这样的部分：

![](https://s.c.accr.cc/picgo/1700530351-c30afb.png)

1. 首先，确认自己的项目目录下有`components.d.ts`：这是由unplugin-vue-components自动生成的，如果没有的话，请在vite的config文件中加上`dts: true`（如上一张图中所示）

   ![](https://s.c.accr.cc/picgo/1700530419-caf9d1.png)

2. 接下来，进入`tsconfig.json`，加入这一条：

   ```typescript
   {
     "files": ["./components.d.ts"],
   }
   ```

   ![](https://s.c.accr.cc/picgo/1700530513-c930fd.png)

   回到vue代码处，看看有没有生效了；按理来说，到这一步就可以生效，获得组件的类型提示；不过，如果你也像我一样没有生效，可以试试后面的步骤：

3. 遵照Vue[官方的说明](https://cn.vuejs.org/guide/typescript/overview.html#volar-takeover-mode)，关闭内置的Typescript支持（可选）

   ![](https://s.c.accr.cc/picgo/1700531731-79cd7d.png)

4. 切换Typescript版本至工作区版

   ![](https://s.c.accr.cc/picgo/1700531829-f42feb.png)

   ![](https://s.c.accr.cc/picgo/1700531869-06f0e9.png)

现在，大功告成！去看看有没有出现类型提示和自动补全吧～

![](https://s.c.accr.cc/picgo/1700531921-324e44.png)

![](https://s.c.accr.cc/picgo/1700532123-4d4638.png)

![](https://s.c.accr.cc/picgo/1700531956-29a1d6.png)

**这种方法的缺点**：对于从来没有使用过的组件，是不会触发组件名称的补全的：

![](https://s.c.accr.cc/picgo/1700532155-c32027.png)

解决方法是手动将组件库的typing.d.ts添加到`tsconfig.json`中——不过我个人认为，这种做法有点不太优雅；毕竟工具都帮你生成好`components.d.ts`了。具体加不加组件库的完整类型定义，读者见仁见智。

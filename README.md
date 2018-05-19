
#### 注意：本文有点标题党了，图很多很罗嗦，不想看图的同学可以直接看加粗的重点和最后的总结。

 最近在做一个项目，技术栈为vue全家桶 + element-ui + echarts，打包后发现有1.44M，首屏体验很差。这能忍？果断开始优化。下面说说我是如何将一个打包后**1.44MB**的项目变成打包后只有**0.42MB**，性能提升**70%** 的。

### 优化过程
0. 准备：

    `vue-cli`提供了一个很方便的查看代码打包后体积的命令，只需在正常的打包命令后加一个`--report`即可，这样打包完成后会自动开启一个页面，展示各个依赖包的大小。
    ```
    npm run build --report
    ```
1. 优化前：
    
    先看看优化前的大小吧

    ![](https://user-gold-cdn.xitu.io/2018/5/19/16378d158576c3d9?w=743&h=278&f=png&s=10398)
    这是打包前本地localhost中首屏加载的js文件，只有一个`app.js` （`3.2MB`）（注意是本地，未打包，未压缩）
    
    ![](https://user-gold-cdn.xitu.io/2018/5/19/16378d00a5128524?w=1912&h=944&f=png&s=245497)
    这是打包后的截图，体积为``1.44MB``,打包时间为``72s``
    

2. 第一次优化：**路由懒加载**

    说到优化，第一个肯定考虑的是懒加载啦，马上在vue和vue-router的[官方文档](https://router.vuejs.org/zh-cn/advanced/lazy-loading.html)里找到了解决方案
    > 结合 Vue 的异步组件和 Webpack 的代码分割功能，轻松实现路由组件的懒加载
    
    **具体做法**的话如下：
    
    首先要安装一个插件[Syntax Dynamic Import](https://babeljs.io/docs/plugins/syntax-dynamic-import/)使项目支持`动态import`
    ```
    cnpm install -S babel-plugin-syntax-dynamic-import
    ```
    然后修改`.babelrc`文件
    ```
    // .babelrc 中的plugins数组中多加一个"syntax-dynamic-import"
    {
      "plugins": ["syntax-dynamic-import"]
    }
    ```
    最后修改`router.js`,将所有路由都改为动态加载
    ```
    //router.js
    
    //原来的写法：import Home from '@/components/PC/Home'
    //改成下面这种形式（其他路由同理）
    const Home = () => import('@/components/PC/Home')       
    ```
    ＯＫ，第一次优化完成。让我们打包看看结果如何吧。
    ![](https://user-gold-cdn.xitu.io/2018/5/19/16378ebbef37cd4d?w=734&h=153&f=png&s=14986)
    
    ![](https://user-gold-cdn.xitu.io/2018/5/19/16378e67cd697ebc?w=1898&h=922&f=png&s=400358)
    
    
    
    上面两张图分别是本地打包前首屏加载的js资源（经计算大约为`3.1MB`）的截图和打包后的截图（`1.44MB`），打包时间为
    `55s`。注意我红色框出来的部分，和优化前相比较打包的结果多了几个以`0` `1` 等数字开头的js文件，这其实就是我们的路由文件被分离了出来，首屏只加载了需要的`0.js`和`3.js`文件，等到我们切换到其他路由的时候才会加载其他的`2.js`或者`4.js`，而不是像以前那样全部包含在了`app.js`中一次性全加载出来。
    
    和优化前相比，打包后大小没变，但是打包时间减少了，首屏加载的js资源也少了`0.1MB`（坑爹么不是！！）。
    > 打包体积没变，首屏才少了0.1MB？效果这么差，你特么在逗我？
    
    别着急打我，听我解释。打包体积没变是因为不管路由怎么懒加载，实质上需要的路由文件还是那么多，大小是不变的，所以体积没变。而首屏才少了`0.1MB`，是因为这个项目本来就是个很小的项目，只有4个页面，而且这个项目的首页引入了`echarts`本来就相对来说比较大。
    
    所以说这一步路由懒加载的优化是完全ok的，效果不好是因为是我项目的原因，少了的那`0.1MB`是剩余未加载的路由文件大小。
    
    **如果你的项目有很多个页面，那么路由懒加载的效果应该会不差。**
    
    我们再次看看这个图
    
    ![](https://user-gold-cdn.xitu.io/2018/5/19/16379039b034d5b5?w=1917&h=956&f=png&s=269832)
    发现左边黄色的框`echarts`和右边蓝色的框`element-ui`体积占了大头，我们先看`element-ui`占了`556KB`，现在开始针对`element-ui`进行第二次优化
    
    
3. 第二次优化：**element-ui组件按需加载**

    针对`element-ui`的优化，没啥好说的，具体做法，直接看[文档](http://element-cn.eleme.io/#/zh-CN/component/quickstart)里面的**按需引入**吧。
    照着文档优化了以后，再次打包查看结果：
    
    
    ![](https://user-gold-cdn.xitu.io/2018/5/19/163790a26fe03a00?w=1917&h=940&f=png&s=327523)
    
    这次优化后，打包用了`45s`，总大小由`1.44MB`变成了`1.16MB`
    ![](https://user-gold-cdn.xitu.io/2018/5/19/163790aba1332fb3?w=1917&h=949&f=png&s=322877)
    而且`element-ui`模块所占的大小也由`556kb`变成了`267kb`，效果还行。但是这点提升怎么满足的了我？解决了`element-ui`，我们看看另外一个模块`echarts`:
    
    ![](https://user-gold-cdn.xitu.io/2018/5/19/16379106a30fd0ac?w=1912&h=959&f=png&s=325545)
    比`element-ui`还要过分！！足足占了`606kb`，马上针对最大的bos----`echarts`进行优化。
    
4. 第三次优化：**使用 CDN 外部加载资源**
    
    这次优化主要是针对`echarts`，在其文档里也有提到[按需加载](http://echarts.baidu.com/tutorial.html#%E5%9C%A8%20webpack%20%E4%B8%AD%E4%BD%BF%E7%94%A8%20ECharts),但是这次我们不用按需加载了，我想把`echarts`彻底干掉！我们这次要使用`webpack`的[**externals**](https://webpack.docschina.org/configuration/externals/)

    > 防止将某些 import 的包(package)打包到 bundle 中，而是在运行时(runtime)再去从外部获取这些扩展依赖(external dependencies)。具有外部依赖(external dependency)的 bundle 可以在各种模块上下文(module context)中使用，例如 CommonJS, AMD, 全局变量和 ES2015 模块。

    **具体做法：**
    
    首先在`index.html`中引入`echarts`的外部CDN（如果需要地图组件，也需要一并引入）
    ```
    //index.html
    <script src="https://cdn.bootcss.com/echarts/4.1.0/echarts.min.js"></script>
    ```

    然后删除**所有**对`echarts`的引用的代码，如
    ```
    //删除下面这句代码
    import echarts from 'echarts'   //或者 const echarts = require('echarts')
    ```
    最后在`webpack.base.config.js`中,做如下改动
    ```
    //webpack.base.config.js   module.exports中增加externals对象
    module.exports = {
        externals: {
            "echarts": "echarts"
        },
    }
    ```
    查看优化结果:
    
    ![](https://user-gold-cdn.xitu.io/2018/5/20/1637921795f1f673?w=555&h=147&f=png&s=15934)
    这是打包前的本地首屏加载资源的截图，可算出这次一共加载了`1.31MB`（没有算上`echarts.min.js`，因为那是CDN资源）,相对于第一次优化后的`3.1MB`已经少了很多了。
    
    打包后的截图如下
    
    ![](https://user-gold-cdn.xitu.io/2018/5/20/1637926cd1470b76?w=1909&h=933&f=png&s=224576)
    可以看到打包后的体积只有`434.7KB`,而且这次打包只花了`34s`，最重要的是`echarts`也真的被干掉了！！
    
    惊不惊喜！！意不意外！！
    
    
    ### 各次优化的表格
    
    **懒得看图的同学可以直接看下面这张表格**
    
      <table>
        <thead>
          <tr>
            <th>#</th>
            <th>打包后体积</th>
            <th>压缩后体积</th>
            <th>首屏js资源</th>
            <th>打包耗时</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>优化前</td>
            <td>1.44M</td>
            <td>425K</td>
            <td>3.2M</td>
            <td>72s</td>
          </tr>
          <tr>
            <td>第一次优化（路由懒加载）</td>
            <td>1.44M</td>
            <td>434K</td>
            <td>3.11M</td>
            <td>55s</td>
          </tr>
          <tr>
            <td>第二次优化（element-ui按需加载）</td>
            <td>1.16M</td>
            <td>381K</td>
            <td>1.3M</td>
            <td>45s</td>
          </tr>
          <tr>
            <td>第三次优化（引入外部CDN）</td>
            <td> 434K</td>
            <td> 121K</td>
            <td>1.3M</td>
            <td>34s</td>
          </tr>
        </tbody>
      </table>
    
    
    可以看出，我们的优化还是很有成效的，各种体积和打包耗时差不多减少了**70%** 左右。
    
    ### 总结
    
    **这部分一定要看啊啊啊**
    
    
    * 遇到webpack打包性能问题，先去`npm run build --report`，然后根据分析结果来做相应的优化，谁占体积大就干谁
    * 路由很多的复杂页面，**路由懒加载**是肯定要做的
    * 现在很多库都有提供**按需加载**的功能，有需要的话可以按照官方文档的做法来按需加载
    * webpack提供的`externals`可以配合**外部资源CDN**轻松大幅度减少打包体积，尤其对于`echarts`、`jQuery`、`lodash`这种库来说
    * 千万不要忘了开启**Gzip**压缩
    * 本文讲的只是针对于`webpack`层面的优化，性能优化不只这些，还有其他方面的优化，比如[**页面渲染优化（减少重排）**](http://aizys.win/2017/12/17/%E5%89%8D%E7%AB%AF%E6%80%A7%E8%83%BD%E4%B9%8B%E9%A1%B5%E9%9D%A2%E6%B8%B2%E6%9F%93%E4%BC%98%E5%8C%96/)、[**网络加载优化**](http://aizys.win/2017/12/14/%E5%89%8D%E7%AB%AF%E6%80%A7%E8%83%BD%E4%B9%8B%E7%BD%91%E7%BB%9C%E5%8A%A0%E8%BD%BD%E4%BC%98%E5%8C%96/)等。

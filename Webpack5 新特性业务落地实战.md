# Webpack5 新特性业务落地实战



![](https://csdnimg.cn/release/blogv2/dist/pc/img/reprint.png)

[frontend_frank](https://blog.csdn.net/frontend_frank)![](https://csdnimg.cn/release/blogv2/dist/pc/img/newCurrentTime2.png)于 2021-03-05 08:02:00 发布![](https://csdnimg.cn/release/blogv2/dist/pc/img/articleReadEyes2.png)1110 ![](https://csdnimg.cn/release/blogv2/dist/pc/img/tobarCollect2.png) 收藏 3

文章标签： [编程语言](https://so.csdn.net/so/search/s.do?q=%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80&t=blog&o=vip&s=&l=&f=&viparticle=) [python](https://so.csdn.net/so/search/s.do?q=python&t=blog&o=vip&s=&l=&f=&viparticle=) [java](https://so.csdn.net/so/search/s.do?q=java&t=blog&o=vip&s=&l=&f=&viparticle=) [人工智能](https://so.csdn.net/so/search/s.do?q=%E4%BA%BA%E5%B7%A5%E6%99%BA%E8%83%BD&t=blog&o=vip&s=&l=&f=&viparticle=) [react](https://so.csdn.net/so/search/s.do?q=react&t=blog&o=vip&s=&l=&f=&viparticle=)

版权

> 本文作者：Wind、Skyler、ZRJ、ZJ

前言  

Webpack5 在 2020 年 10 月 10 日正式发布，并且在过去的几个月中快速演进和迭代，截止 1 月 28 日，Webpack5 已经更新了 18 个 minor 版本，带来了许多十分吸引人的新特性。据官网介绍[1]，Webpack5 整体的方向性变化有以下几点：

![](https://img-blog.csdnimg.cn/img_convert/a0ed8c42e035624b5f872ce0a6979e67.png)

- 通过持久化硬盘缓存能力来提升构建性能

- 通过更好的算法来改进长期缓存（降低产物资源的缓存失效率）

- 通过更好的 Tree Shaking 能力和代码的生成逻辑来优化产物的大小

- 改善 web 平台的兼容性能力

- 清除了内部结构中，在 Webpack4 没有重大更新而引入一些新特性时所遗留下来的一些奇怪的 state

- 通过引入一些重大的变更为未来的一些特性做准备，使得能够长期的稳定在 Webpack5 版本上

最后的两点介绍比较抽象，但整体可以看出，其实升级 Webpack5 对于开发者而言，能够感知到的就是构建时的效率和运行时的性能都有着明显的提升，包括一些新特性可能给我们带来的更舒适的“coding 姿势”。

# 团队实践

团队目前使用的是 Webpack4 来承接整套构建体系，迁移时我们参考了 [Webpack](https://so.csdn.net/so/search?q=Webpack&spm=1001.2101.3001.7020) 官方提供的升级手册[2]，这里我们针对一些对我们的业务现阶段有实际价值的点（新特性基本都有实际收益），集成到了团队的构建流程中，通过实践验证并记录了我们在基建落地的过程中遇到的一些问题和解决方法。

## 准备工作

### 前期调研

首先，通过官方文档给出的指引我们可以找出一些明确需要有改动的地方，其中就包括了如下几个方面：

- Webpack5 需要 Node 版本 ^10.13.0 的支持。本地使用的时候，需要将 Node 版本升级到 10.13.0 以上。对于使用了一些 CI 工具进行产物构建的时候，也需要同步升级对应镜像或者 CI 上配置的 Node 版本。

- 项目中的 html-webpack-plugin 需要升级的到支持 Webpack5 的版本[3]

- 项目中有使用到 workbox-webpack-plugin 需要升级到支持 Webpack5 的版本

- DevServer 需要升级 webpack-dev-server 至 ^4(next) 版本，否则 HMR 会有异常。

- Webpack5 移除了 Node.js Polyfill，将会导致一些包变得不可用（会在控制台输出 `'XXX' is not defined`），如果需要兼容 process 等 Nodejs Polyfill，则要安装相关的 Polyfill：process，并在 Plugin 中显式声明注入

```go
// webpack.config.js{    ...,    plugins: [        ...,        new webpack.ProvidePlugin({          process: 'process/browser',        }),    ]}
```

- 如果你在使用 Webpack5 的时候遇到一些 API 的弃用警告，可以果断搜索 Google，一般都是让你升级一些 Plugin 就完全能够解决问题了，并且这些 warning 现阶段并不会对你的构建产生什么破坏性的影响，以下我们提供了一些我们实践中沉淀的参考：

- - 升级 terser-webpack-plugin 能解决的 Warning:
  
  - - Compilation.cache was removed
    
    - optimizeChunkAssets
  
  - 升级 mini-css-extract-plugin 能够解决的 warning:
  
  - - MainTemplate.hooks.renderManifest
    
    - ChunkTemplate.hooks.renderManifest
    
    - MainTemplate.hooks.hashForChunk
    
    - Module.id: Use new ChunkGraph API
    
    - Module.updateHash
    
    - Chunk.modulesIterable
    
    - MainTemplate.outputOptions
    
    - MainTemplate.renderCurrentHashCode
    
    - MainTemplate.getAssetPath
    
    - MainTemplate.requireFn
    
    - ChunkGroup.getModuleIndex2 was renamed to getModulePostOrderIndex

做好了前面这些准备之后，我们就可以直接升级一把梭：

```go
npm install webpack@latest --dev
```

接下来，就可以开始尝试令人心动的 Webpack5 啦～

### 踩坑日记

#### 生态对齐 Webpack5

如果你的项目升级了 Webpack5 之后直接在项目中启动构建或者 devServer，你一般会收到类似下面这样的报错信息：

```go
TypeError: Cannot add property htmlWebpackPluginAlterChunks, object is not extensible    at /path/to/node_modules/html-webpack-plugin/index.js:59:56
```

如果是 Webpack 老司机应该就会一眼发现这个问题的关键所在 —— `html-webpack-plugin`，打开 Plugin 的 Github 首页[4]，解决方案就摆在你的眼前了：

![](https://img-blog.csdnimg.cn/img_convert/6b186c29f01c21dd56b40961f125ef3a.png)

同理，一般在控制台如果有抛出 error 或者 warning，大多都能通过升级对应依赖的版本解决问题，Webpack5 的升级还是要比当年 Webpack4 的升级来得丝滑许多，并且社区上的生态也在逐步跟进和完善。

这里再举几个比较常见的例子，比如我们常用来支撑 serviceWorker 的 workbox-webpack-plugin，因为它内部也依赖了 Webpack 的核心事件以及一些 API，因此也需要进行升级，这里可以参考：https://github.com/GoogleChrome/workbox/issues/2669；还有我们平时用来压缩 CSS 的 optimize-css-assets-webpack-plugin[5]，在 Plugin Github 首页中也明确表示了在 Webpack5 之后优先使用 Webpack 官方出品的 css-minimizer-webpack-plugin[6]。

![](https://img-blog.csdnimg.cn/img_convert/73a2e5e58b5ddc279a14a220af8fd61d.png)

当然，还有一些其他生态对齐的点就不在这个文档里再做赘述了，大部分内容你都可以在官网介绍[1]以及升级手册[2]这两份官方文档中找到，并且，如果有一些需要 case by case 去解决的通用问题，社区上大多也已经有 issue 或者其他文档中可以发现线索，当然，如果被你发现了一些“待解决”的问题，不也正是你向开源社区贡献力量的机会嘛～

基本上完成了上面这些依赖的升级，你的项目就能通过 Webpack5 跑起来了。

> PS：optimize-css-assets-webpack-plugin[5] 这插件既没有 cache 又没有 parallel，就我而言，我宁可在 loader 里使用 cssnano 做 CSS 逻辑优化，也不愿意在这个插件上无奈地浪费生命 ????，但是 css-minimizer-webpack-plugin[6] 解救了强迫症的我，它真的是再好不过的选择了。

#### 一碰就挂的 HMR

由于 webpack-dev-server 依赖了很多 Webpack 构建配置，所以在升级之后，不免有些地方没有对接地那么丝滑，HMR 就是一个最形象的例子，它变得十分脆弱，往往在不知不觉中，你会突然发现，这东西咋没反应了？！

我的第一直觉是认为自己是不是忽略了什么 devServer 的配置，但我不是照着官方文档 CV 的嘛？！冷静下来后我点开了 Google，果然，社区上早有前人遇到了同样的问题[7]。

这里我就长话短说了，这是 `webpack-dev-server` 遗留的一个 BUG，只有你在 Webpack 配置中显式地设置了 `target: 'web'`，HMR 才能够生效，哪怕你通过数组的方式去设置 target，例如：`target: \['web', 'es5'\]`，也会阻塞 HMR 的能力。

但好在官方发布了对接 Webpack5 的 v4 版本，你只需要通过升级 webpack-dev-server 至 next 版本（如果你在项目中通过它提供的 Node API 使用它的话），就能解决上述问题：

![](https://img-blog.csdnimg.cn/img_convert/9e034b0855566c976970cc4f1fbb77aa.png)

不过，因为截止 2020 年 1 月 28 日 webpack-dev-server v4 只发布了 beta0 一个 next 版本[8]，所以还有些能力不太稳定，比如你没法通过 `localhost` 启动你的项目[9]，但是这些都只是细节问题了，不影响整体的功能完整跑通。  

## 构建时新特性

### 内置静态资源构建能力 —— Asset Modules

> Webpack 只会打包 JS 文件，如果要打包其它文件就需要加上相应的 loader。

在 Webpack5 之前，我们一般都会使用以下几个 loader 来处理一些常见的静态资源，比如 PNG 图片、SVG 图标等等，他们的最终的效果大致如下所示：

- raw-loader：允许将文件处理成一个字符串导入

- file-loader：将文件打包导到输出目录，并在 import 的时候返回一个文件的 URI

- url-loader：当文件大小达到一定要求的时候，可以将其处理成 base64 的 URIS ，内置 file-loader

Webpack5 提供了内置的静态资源构建能力，我们不需要安装额外的 loader，仅需要简单的配置就能实现静态资源的打包和分目录存放。如下：满足规则匹配的资源就能够被存放在 assets 文件夹下面。

```go
// webpack.config.jsmodule.exports = {    ...,    module: {      rules: [          {            test: /\.(png|jpg|svg|gif)$/,            type: 'asset/resource',            generator: {                // [ext]前面自带"."                filename: 'assets/[hash:8].[name][ext]',            },        },      ],    },}
```

其中 type 取值如下几种：

- asset/source ——功能相当于 raw-loader。

- asset/inline——功能相当于 url-loader，若想要设置编码规则，可以在 generator 中设置 dataUrl。具体可参见官方文档[10]。

- asset/resource——功能相当于 file-loader。项目中的资源打包统一采用这种方式，得益于团队项目已经完全铺开使用了 HTTP2 多路复用的相关特性，我们可以将资源统一处理成文件的形式，在获取时让它们能够并行传输，避免在通过编码的形式内置到 js 文件中，而造成资源体积的增大进而影响资源的加载。

- asset—— 默认会根据文件大小来选择使用哪种类型，当文件小于 8 KB 的时候会使用 asset/inline，否则会使用 asset/resource。也可手动进行阈值的设定，具体可以参考官方文档[11]。

### 内置 FileSystem Cache 能力加速二次构建

Webpack5 之前，我们会使用 cache-loader[12] 缓存一些性能开销较大的 loader ，或者是使用 hard-source-webpack-plugin[13] 为模块提供一些中间缓存。在 Webpack5 之后，默认就为我们集成了一种自带的缓存能力（对 module 和 chunks 进行缓存[14]）。通过如下配置，即可在二次构建时提速。

```go
// webpack.config.jsmodule.exports = {    ...,    cache: {        type: 'filesystem',        // 可选配置        buildDependencies: {            config: [__filename],  // 当构建依赖的config文件（通过 require 依赖）内容发生变化时，缓存失效        },        name: '',  // 配置以name为隔离，创建不同的缓存文件，如生成PC或mobile不同的配置缓存        ...,    },}
```

生产环境下默认的缓存存放目录在 `node_modules/.cache/webpack/default-production` 中，如果想要修改，可通过配置 name，来实现分类存放。如设置 `name: 'production-cache'` 时生成的缓存存放位置如下。

![](https://img-blog.csdnimg.cn/img_convert/ebb18fe59700aa3e4bd85b91405c5b09.png)

PS：如果你直接通过调用 Webpack compiler 实例的 run 方法执行定制化构建操作时，你可能会遇到**构建缓存最终没有生成缓存文件**的情况，在搜索了 Webpack 的相关 Issues 后我们发现，你还需要手动调用 `compiler.close()` 来输出缓存文件。

![](https://img-blog.csdnimg.cn/img_convert/fcc4aa0fd15a5f53625c6cda621672f8.png)

### 内置 WebAssembly 编译及异步加载能力（sync/async）

WebAssembly[15] 被设计为一种面向 web 的二进制的格式文件，以其更接近于机器码而拥有着更小的文件体积和更快速的执行效率。c/c++ 等高级语言都能直接编译成 `.wasm` 文件而被 js 调用。Webpack4 本身就已经集成了 WebAssembly 的加载能力，只不过在 Webpack5 拓展了 WebAssembly 的异步加载能力，使得我们可以更灵活地借助 WebAssembly 做更多有意思的事情。

借此机会，我们可以简单介绍一下 WebAssembly 在实际项目开发中的快速上手流程：

以一个简单的求和函数为例，除了按照官方文档[16]生成 WebAssembly 文件外，我们可以通过 WasmFiddle[17] 这个在线网址来编写 c/c++ 程序，再直接转换为 WebAssembly 文件，并且该网站还为我们提供了即时的调试能力。

![](https://img-blog.csdnimg.cn/img_convert/51afc2e3b0a671c942a5b5ee194b944a.png)

通过上述方式，我们能够很快的得到一个 program.wasm 文件，将其下载到本地项目中，你便可以开始使用它了。

在 Webpack5 之前，我们会通过 wasm-loader 来进行 WebAssembly 文件的处理，同时在使用时还需要通过如下示例代码才能调用我们封装在 wasm 里的函数：

```go
import wasm from '/program.wasm'; wasm().then(instance => {  const sum = instance.exports.sum;  console.log(sum(1, 2));}
```

这个过程势必是重复繁琐的，但是，通过 Webpack5 内置的 WebAssembly 构建能力，我们仅需要配置如下：

```go
// webpack.config.jsmodule.exports = {    ...,    experiments: {        asyncWebAssembly: true,    },    module: {        rules: [            ...,           {                test: /\.wasm$/,                type: 'webassembly/async',            },         ],    },}
```

而当我们要使用 wasm 的时候，就是如此简单：

```go
import { sum } from './program.wasm'console.log(sum(1, 2))
```

### 内置 Web Worker 构建能力

Web Worker 为 Web 内容在后台线程中运行脚本提供了一种简单的方法。线程可以执行任务而不干扰用户界面。通常，我们可以将一些加解密或者图片处理等一些比较复杂的算法置于子线程中，当子线程执行完毕之后，再向主线程通信。

假如我们有一个比较耗时的计算逻辑存放在 calc.js 文件中，当我们执行完成之后，通过 `postMessage` 将文件信息通知给主线程。

```go
//  calc.worker.jslet num;for (let i = 0; i <= 20000000; i++) {  if (i === 20000000) {    num = 20000000;  }}postMessage({  value: num,}); 
```

master.js 主线程监听子线程的消息，当收到消息传输过来的时候，执行一些相应的操作。

```go
// master.jsworker.onmessage = e => {  console.log(e.data.value);};
```

对于以前在处理 Web Worker 的时候，我们需要借助于 worker-loader 来处理，通过如下配置

```go
// webpack.config.jsmodule.exports = {    ...,     module: {        rules: [            {                test: /\.worker\.js$/,                use: { loader: 'worker-loader' },            },        ],    },}
```

在使用的时候，直接引入 calc.worker.js 文件就能够构造一个 Worker 对象，因此主线程的处理如下：

```go
// master.jsimport Worker from './calc.worker.js';const worker = new Worker();worker.onmessage = e => {  console.log(e.data.value);};
```

在 Webpack5 中，我们不需要添加 loader 的处理方式，并且不需要针对 worker 配置特定的 .worker.js 之类的文件名，借助于 new URL， 便能实现 worker 的创建。如下，亦可参考官方示例[18]：

```go
// master.jsconst worker = new Worker(new URL('./calc.js', import.meta.url), {    name: "calc"  /* webpackEntryOptions: { filename: "workers/[name].js" } */});worker.onmessage = e => {  console.log(e.data.value);};
```

但因为考虑到开发者对于编程的习惯和方便性，我们此次并未将 worker-loader 全量下掉，你还是可以直接通过 `import` 一个 `.worker.js` 后缀的来使用 web worker ；同时，你也可以使用上述 Webpack5 的新特性，但是**请注意，在`new URL()`中不能使用`.worker.js`命名文件，否则会优先被 worker-loader 解析而导致最终你的 worker 无法正常运行**。

## 运行时新特性

### 移除了 Node.js Polyfills，Polyfill 交由开发者自由控制

由于移除了 Node.js Polyfills，如果前端包里使用了 process、path 这些依赖，需要手动添加 Polyfill 支持。如前面准备工作介绍的 process 等。

以 react-markdown 为例，当我们的项目升级到 Webpack5 之后，就会报错提示 `process.cwd is not a function`，如果你在项目里也遇到了类似的情况，比如某个你熟悉的 nodejs API[19] 在控制台报错 `is not defined`，这时候就需要我们自行添加相关的 Browser Polyfill，因为 Webpack5 不会再帮你处理这些 polyfill 了。process 的详细配置可参考准备工作。

![](https://img-blog.csdnimg.cn/img_convert/c1ec7e19e7237108f2a8370eb38cc3e6.png)

### 资源打包策略更优，构建产物更“轻量”

Prepack 是 Facebook 开源的一个 JavaScript 代码优化工具，运行在 “编译” 阶段，生成优化后的代码。下面是 Prepack 官网上的一个示例，我们可以看到，在对于任何输入，函数都能得到一个固定输出的时候，Prepack 就能在编译时，将结果帮我们计算出来。对于一些复杂且固定的计算逻辑而言，这种“预计算”能力，既能减小我们包的体积，又能加快运行时的速度。如官方示例[20]所示：

![](https://img-blog.csdnimg.cn/img_convert/1ecdb81a890e4e13e442cccec48cb93f.png)

Webpack5 内置了 Prepack 的部分能力，能够在极致之上，再度优化你的项目产物体积：  

![](https://img-blog.csdnimg.cn/img_convert/0c9d53441d590c790b59209d0636ff49.png)

### 深度 Tree Shaking 能力支持

Tree Shaking 能力，是指能够在打包的过程中移除 JavaScript 上下文中未被引用到的变量，借以次来减少打包后的体积。比如我们在开发阶段，引入某个文件时，被引用的文件中存在部分没有用到的代码、或是在开发的时候忘记删掉、亦或是前面开发者遗留下来的代码但是不便大胆的做一些删减工作，有了 Tree Shaking 这个功能之后，这些都将不在是问题，Webpack 在打包的时候能够自动帮我们完成没使用到的代码的删除工作，特别是 Webpack5 能够支持深层嵌套的 export 的 Tree Shaking，具体可参考官网[21]给出的示例：

![](https://img-blog.csdnimg.cn/img_convert/6474eb325854097b96d940a22967e5ca.png)

以上述为例，在 Webpack5 的能力下，配合上面我们提到的类 Prepack 能力，那些冗余的代码就自动被清除了：

![](https://img-blog.csdnimg.cn/img_convert/f1072ec7d7dd81acbad70218b1e5ecfa.png)

所以，你还需要担心项目里会有冗余代码对你的线上包体积造成负担吗？

### 更友好的 Long Term Cache 支持性，chunkid 不变

Webpack5 之前，文件打包后的名称是通过 ID 顺序排列的，一旦后续有一个文件进行了改动，那么必将造成后面的文件打包出来的文件名产生变化，即使文件内容没有产生改变。因此会造成资源的缓存失效。

Webpack5 有着更友好的长期缓存能力支持，其通过 [hash](https://so.csdn.net/so/search?q=hash&spm=1001.2101.3001.7020) 生成算法，为打包后的 modules 和 chunks 计算出一个短的数字 ID ，这样即使中间删除了某一个文件，也不会造成大量的文件缓存失效，具体介绍可参见官网[22]。

例如，我们在项目中新增一个页面，然后对比 Webpack4 和 Webpack5 打包后的产物名称的变化：

Webpack4：在首位产生的一个新的 chunk 将会导致所有 js 文件缓存失效，我们可以通过 sourcemap 内容看出，其实两个名称完全不同的 JS 文件，指向的始终是一份源文件。

![](https://img-blog.csdnimg.cn/img_convert/aa6d233eccb403da06a4d53058fa2fd5.png)

Webpack5：仅对有修改的文件失效

![](https://img-blog.csdnimg.cn/img_convert/5891213dcde9e68d7492b9a6bfc8c61a.png)

这里还需要额外拓展一下，Webpack5 还使用了真实的 contenthash[23] 来支持更友好的 Long term cache，啥意思呢，就是如果你的逻辑里只是删了下注释或者改了个变量名，那本质上你的代码逻辑是没有发生变化的，所以对于压缩后的文件这些内容的变更不会导致 contenthash 变化。

### 支持 Top Level Await，从此告别 async

Webpack5 还支持 Top Level Await。即允许开发者在 async 函数外部使用 await 字段。它就像巨大的 async 函数，原因是 import 它们的模块会等待它们开始执行它的代码，因此，这种省略 async 的方式只有在顶层才能使用。

通过开启以下配置：

```go
// webpack.config.jsmodule.exports = {    ...,    experiments: {        topLevelAwait: true,    },}
```

开启前后对比，以我们的国际化项目中经常用到的 i18n 拉取字典的逻辑为例：

```go
// 开启 top level await 之前import i18n from 'XXX/i18n-utils' (async () => {  // 国际化文案异步初始化逻辑  await i18n.init({/* ... */})  root.render(<AppContainer />)})() // 开启 top level await 之后import i18n from 'XXX/i18n-utils' await i18n.init({/* ... */})root.render(<AppContainer />)
```

我们不再需要为我们的异步逻辑包裹 async IIFE 了，在 JS 逻辑顶层，我们就可以直接使用 `await` 来进行异步逻辑的控制。

当然，我们也可以将此特性用于异步导出或者引入模块：

```go
// src/Home/index.jsximport React from 'react'; const Test = () => {  return <div>123</div>;}; let Home = null;await new Promise(resolve => {  Home = Test;  resolve();}); export default Home; // src/index.jsximport Home from './Home'
```

为了 eslint 语法检测的支持，我们还需要添加 babel 插件 `@babel/plugin-syntax-top-level-await` 来让我们的 babel 能够识别 top level await 语法。

但是，不同于我们处理其他 ES6 语法的方式，我们并不能将 top level await 语法的编译过程都收口在 Babel 中，就如下方的插件首页截图所述，此插件的作用仅仅是使得 babel 能够解析这个新语法，并将它转化为 AST，但并不会将其编译掉，因此真正的编译过程还是需要交给 Webpack 或者 Rollup 这样的构建工具来处理来处理的。

![](https://img-blog.csdnimg.cn/img_convert/375a20e2ef71734190170e55f69de524.png)

# 最终效果

为了展示 Webpack5 在打包构建的过程中对于项目的收益，我们做了如下对比：

## 构建效率对比

对于 filesystem 缓存开启与否，我们针对部门的项目模版做了如下的对比实验。可以看到，除了初次构建话费时间相对长一点外，后续的二次构建速度得到了大幅度的提高。

- 没有配置时的初次和二次构建：

![](https://img-blog.csdnimg.cn/img_convert/1bd608cb2c6a9c0e4737c22ee7d35e04.png)

- 配置了缓存的初次构建和二次构建：

![](https://img-blog.csdnimg.cn/img_convert/127e3ff436d339ce4191f45cd048318f.png)

![](https://img-blog.csdnimg.cn/img_convert/b91b27560a340073661499ac4a02bc22.png)

## Chunk 更新产物的缓存失效率对比

以我们在上述 Long Term Cache 特性中给出的打包后的 8 个 js chunk 资源为例。

Webpack4：在首位产生的一个新的 chunk 将会导致所有 js 文件缓存失效。

![](https://img-blog.csdnimg.cn/img_convert/5e3a3de2a913fb10fb9f9f94b49b8744.png)

Webpack5：仅对有修改的文件失效

![](https://img-blog.csdnimg.cn/img_convert/f33b0a8a09c9d23bd1ba2bab9cfdf5b2.png)

![](https://img-blog.csdnimg.cn/img_convert/9b669837af0be78149e5f52ab79b823d.png)

当然，Webpack5 升级给项目带来的最终增益还远远不止上面提到的这些，我们还需要收集更多的数据来验证 Webpack5 的升级对运行时阶段带来的更深层次的性能收益，但就目前而言，Webpack5 的许多特性已经能让我们在开发阶段舒适许多。  

# 总结

Webpack5 虽然说为我们提供了很多优秀的新特性，但是对于开发者在实际的开发过程中的改动感知还是比较弱的，我们可以很舒适地上手这些新特性，并且在升级过程中随着 Webpack 体系生态的逐步完善，当我们遇到问题时也都能够很快找到明确的解决方案，所以，赶快一起体验起来吧。

如果有同学也对 Webpack5 的相关能力感兴趣，咱们也可以随时交流讨论哈～

**招聘硬广**

我们团队招人啦！！！欢迎加入字节跳动商业变现前端团队，我们在做的技术建设有：前端工程化体系升级、团队 Node 基建搭建、前端一键式 CI 发布工具、组件服务化支持、前端国际化通用解决方案、重依赖业务系统微前端改造、可视化页面搭建系统、商业智能 BI 系统、前端自动化测试等等等等，拥有近百号人的北上杭大前端团队，一定会有你感兴趣的领域，如果你想要加入我们，欢迎点击我们的内推通道吧：

✨✨✨✨✨

校招专属入口：字节跳动校招内推码: **HTZYCHN**

如果你想了解我们部门的日常生 (dòu) 活 (bī) 以及工作环 (fú) 境 (lì)

也可以点击**阅读原文**了解噢～

✨✨✨✨✨

### 参考资料

[1] https://webpack.js.org/blog/2020-10-10-webpack-5-release/#general-direction

[2] https://webpack.js.org/migrate/5/

[3] https://github.com/jantimon/html-webpack-plugin#webpack-5

[4] https://github.com/jantimon/html-webpack-plugin

[5] https://github.com/NMFR/optimize-css-assets-webpack-plugin

[6] https://github.com/webpack-contrib/css-minimizer-webpack-plugin

[7] https://github.com/webpack/webpack-dev-server/issues/2758

[8] https://www.npmjs.com/package/webpack-dev-server

[9] https://github.com/webpack/webpack-dev-server/issues/2934

[10] https://webpack.js.org/guides/asset-modules/#custom-data-uri-generator

[11] https://webpack.js.org/guides/asset-modules/#general-asset-type

[12] https://github.com/webpack-contrib/cache-loader

[13] https://github.com/mzgoddard/hard-source-webpack-plugin

[14] https://webpack.js.org/configuration/other-options/#cache

[15] https://webassembly.org/

[16] https://webassembly.org/getting-started/developers-guide/

[17] https://wasdk.github.io/WasmFiddle//

[18] https://github.com/webpack/webpack/tree/master/examples/worker

[19] https://nodejs.org/dist/latest-v14.x/docs/api/

[20] https://prepack.io/

[21] https://webpack.js.org/blog/2020-10-10-webpack-5-release/#major-changes-optimization

[22] https://webpack.js.org/blog/2020-10-10-webpack-5-release/#major-changes-long-term-caching

[23] https://webpack.js.org/blog/2020-10-10-webpack-5-release/#real-content-hash
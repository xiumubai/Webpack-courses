缓存是一种应用非常广泛性能优化技术，在计算机领域几乎无处不在，例如：操作系统层面 CPU 高速缓存、磁盘缓存，网路世界中的 DNS 缓存、HTTP 缓存，以及业务应用中的数据库缓存、分布式缓存等等。

那自然而然的，我们也可以在 Webpack 使用各式各样的缓存技术，通过牺牲空间来提升构建过程的时间效率，在这篇文章中，我将从 Webpack5 的 持久化缓存 开始介绍用法、性能收益、基本原理；之后再过渡到 Webpack4 中如何借助第三方组件(Loader、Plugin)实现持久化缓存。

## Webpack5 中的持久化缓存

持久化缓存 算得上是 Webpack 5 最令人振奋的特性之一，它能够将首次构建的过程与结果数据持久化保存到本地文件系统，在下次执行构建时跳过解析、链接、编译等一系列非常消耗性能的操作，直接复用上次的 Module/ModuleGraph/Chunk 对象数据，迅速构建出最终产物。

持久化缓存的性能提升效果非常出众！以 Three.js 为例，该项目包含 362 份 JS 文件，合计约 3w 行代码，算得上中大型项目：

![iamge](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e1cc62bb30441eb8c50865f66b45f29~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

配置 babel-loader、eslint-loader 后，在我机器上测试，未使用 cache 特性时构建耗时大约在 11000ms 到 18000ms 之间；启动 cache 功能后，第二次构建耗时降低到 500ms 到 800ms 之间，两者相差接近 50 倍！

而这接近 50 倍的性能提升，仅仅需要在 Webpack5 中设置 cache.type = 'filesystem' 即可开启：

```js
module.exports = {
  //...
  cache: {
    type: 'filesystem',
  },
  //...
};
```

执行效果：

![image](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9dc585911d5d41bcbf70eee0faccb3be~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

此外，cache 还提供了若干用于配置缓存效果、缓存周期的配置项，包括：

- cache.type：缓存类型，支持 'memory' | 'filesystem'，需要设置为 filesystem 才能开启持久缓存；
- cache.cacheDirectory：缓存文件路径，默认为 node_modules/.cache/webpack ；
- cache.buildDependencies：额外的依赖文件，当这些文件内容发生变化时，缓存会完全失效而执行完整的编译构建，通常可设置为各种配置文件，如：

```js
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [
        path.join(__dirname, 'webpack.dll_config.js'),
        path.join(__dirname, '.babelrc'),
      ],
    },
  },
};
```

- cache.managedPaths：受控目录，Webpack 构建时会跳过新旧代码哈希值与时间戳的对比，直接使用缓存副本，默认值为 ['./node_modules']；
- cache.profile：是否输出缓存处理过程的详细日志，默认为 false；
- cache.maxAge：缓存失效时间，默认值为 5184000000 。

## 缓存原理

那么，为什么开启持久化缓存之后，构建性能会有如此巨大的提升呢？

Webpack5 会将首次构建出的 Module、Chunk、ModuleGraph 等对象序列化后保存到硬盘中，后面再运行的时候，就可以跳过许多耗时的编译动作，直接复用缓存数据。

Webpack 的构建过程，大致上可划分为三个阶段。

![iamge](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58feafdeed084eefa40f12f98b627262~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

- 初始化，主要是根据配置信息设置内置的各类插件。
- Make - 构建阶段，从 entry 模块开始，执行：
  - 读入文件内容；
  - 调用 Loader 转译文件内容；
  - 调用 acorn 生成 AST 结构；
  - 分析 AST，确定模块依赖列表；
  - 遍历模块依赖列表，对每一个依赖模块重新执行上述流程，直到生成完整的模块依赖图 —— ModuleGraph 对象。
- Seal - 生成阶段，过程：
  - 遍历模块依赖图，对每一个模块执行：
  - 代码转译，如 import 转换为 require 调用；
  - 分析运行时依赖。
  - 合并模块代码与运行时代码，生成 chunk；
  - 执行产物优化操作，如 Tree-shaking；
  - 将最终结果写出到产物文件。

过程中存在许多 CPU 密集型操作，例如调用 Loader 链加载文件时，遇到 babel-loader、eslint-loader、ts-loader 等工具时可能需要重复生成 AST；分析模块依赖时则需要遍历 AST，执行大量运算；Seal 阶段也同样存在大量 AST 遍历，以及代码转换、优化操作，等等。假设业务项目中有 1000 个文件，则每次执行 npx webpack 命令时，都需要从 0 开始执行 1000 次构建、生成逻辑。

而 Webpack5 的持久化缓存功能则将构建结果保存到文件系统中，在下次编译时对比每一个文件的内容哈希或时间戳，未发生变化的文件跳过编译操作，直接使用缓存副本，减少重复计算；发生变更的模块则重新执行编译流程。缓存执行时机如下图：

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc6ac3a471664560b3db676e73cb0c62~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

如图，Webpack 在首次构建完毕后将 Module、Chunk、ModuleGraph 三类对象的状态序列化并记录到缓存文件中；在下次构建开始时，尝试读入并恢复这些对象的状态，从而跳过执行 Loader 链、解析 AST、解析依赖等耗时操作，提升编译性能。

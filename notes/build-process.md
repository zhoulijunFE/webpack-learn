<img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/webpack.png" width="600" height="300"/>
webpack 是一个现代 JavaScript 应用程序的静态模块打包器。当 webpack 处理应用程序时，它会递归地构建一个依赖关系图，其中包含应用程序需要的每个模块，然后将所有这些模块打包成一个或多个文件。

## 术语介绍
* **Entry:**    webpack 构建的入口模块， 来作为构建其内部依赖的开始
* **Output:**     构建输出的文件目录及命名
* **Module:**   模块，在 webpack 里一切皆模块，一个模块对应着一个文件。webpack 会从配置的 Entry 开始递归找出所有依赖的模块
* **Chunk:**   coding split的产物，我们可以对一些代码打包成一个单独的chunk，比如某些公共模块，去重，更好的利用缓存。或者按需加载某些功能模块，优化加载时间
* **Loader:**   对模块的源代码进行转换为 webpack 能够处理的有效模块
* **Plugin:**   扩展插件，在 webpack 构建流程中的特定时机会广播出对应的事件，插件可以订阅这些事件的发生，在特定时机做对应的事情

## 构建流程
<img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/build-process.png" width="350" height="750"/>
**注： 构建流程 是基于 webpack:  4.28.4 版本**

### 配置解析、合并
  通过 yargs 解析 webpack.config.js 与 命令行参数，合并options对象
  ```
  // 配置解析入口：/webpack-cli/bin/cli.js
  const yargs = require("yargs")
  yargs.parse(process.argv.slice(2), (err, argv, output) => {
    // 命令行中读取config参数
    if (argv.config) {
      const configArgList = Array.isArray(argv.config) ? argv.config : [argv.config];
      const mapConfigArg = function mapConfigArg(configArg) {
        const resolvedPath = path.resolve(configArg);
      }
      configFiles = configArgList.map(mapConfigArg);
    } else {
      // 项目目录查找 webpack.config配置
      const defaultConfigFileNames = ["webpack.config", "webpackfile"].join("|");
      const webpackConfigFileRegExp = `(${defaultConfigFileNames})(${extensions.join("|")})`;
      const pathToWebpackConfig = findup(webpackConfigFileRegExp);
      if (pathToWebpackConfig) {
        const resolvedPath = path.resolve(pathToWebpackConfig);
        configFiles.push({
          path: resolvedPath,
        });
      }
    }

  // 配置文件转化options对象 /webpack-cli/bin/utils/convert-argv.js
  options = require(path.resolve(process.cwd(), filePath))
  ```

### 编译前的准备工作
  根据第一步options对象 生成 Compiler 对象，然后初始化 webpack 的默认配置options、及内置插件
   <img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/build-before.png" width="690" height="350"/>
#### Compiler对象
 * Compiler 继承Tapable事件流机制，定义了很多钩子, 实现插件订阅、广播 更多Tapable事件流机制
 * Compiler是webpack编译的控制流程中心，包含了所有的 webpack 环境信息，会在启动 webpack 的时候被一次性的初始化
 ```
 class Compiler extends Tapable {
  constructor(context) {
    super();
    this.hooks = {
      /** @type {SyncBailHook<string, Entry>} */
      entryOption: new SyncBailHook(["context", "entry"]
      /** @type {AsyncSeriesHook<Compiler>} */
      beforeRun: new AsyncSeriesHook(["compiler"]),
      /** @type {AsyncSeriesHook<Compiler>} */
      run: new AsyncSeriesHook(["compiler"]),
      /** @type {AsyncSeriesHook<CompilationParams>} */
      beforeCompile: new AsyncSeriesHook(["params"]),
      /** @type {SyncHook<CompilationParams>} */
      compile: new SyncHook(["params"]),
      /** @type {AsyncParallelHook<Compilation>} */
      make: new AsyncParallelHook(["compilation"])
   }
   this.options = /** @type {WebpackOptions} */ ({});
   this.context = context;
}
 ```
 
#### 代码详细拆解
 ```
 // webpack-cli/bin/cli.js
const webpack = require("webpack");
let compiler = webpack(options);
// 下一步开始编译主入口
compiler.run((err, stats) => {
  compilerCallback(err, stats);
}

// 调用/webpack/lib/webpack.js 生成Compiler对象
const webpack = (options, callback) => {
  // 初始化webpack默认配置options(自定义配置覆盖默认配置，webpack 4.0零配置)
  let options = new WebpackOptionsDefaulter().process(options);
  // 根据options初始化compiler
  let compiler = new Compiler(options.context);
  compiler.options = options;
  // 初始化配置文件中plugins
  if (options.plugins && Array.isArray(options.plugins)) {
    for (const plugin of options.plugins) {
      if (typeof plugin === "function") {
        plugin.call(compiler, compiler);
      } else {
        plugin.apply(compiler);
      }
    }
  }
  compiler.hooks.environment.call();
  compiler.hooks.afterEnvironment.call();
  // webpackOptionsApply初始化webpack默认plugins
  compiler.options = new WebpackOptionsApply().process(options, compiler);
}

// webpack/lib/WebpackOptionsApply.js初始化默认plugins
const EntryOptionPlugin = require("./EntryOptionPlugin");
class WebpackOptionsApply extends OptionsApply {
  constructor() {
    super();
  }
  process(options, compiler) {
    // 初始化入口插件EntryOptionPlugin
    new EntryOptionPlugin().apply(compiler);
    // 触发compiler的entryOption钩子
    compiler.hooks.entryOption.call(options.context, options.entry);
  }
}

// webpack/lib/EntryOptionPlugin.js插件
const SingleEntryPlugin = require("./SingleEntryPlugin");
module.exports = class EntryOptionPlugin {
  apply(compiler) {
    // 订阅compiler的entryOption钩子
    compiler.hooks.entryOption.tap("EntryOptionPlugin", (context, entry) => {
      // 初始化SingleEntryPlugin对象
      new SingleEntryPlugin(context, entry, "main")
    }
  }
}
 ```
### 开始编译主入口
  初始化Compilation对象，触发compilation.addEntry正式进入构建阶段
  <img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/build-entry.png" width="920" height="320"/>
#### Compilation对象
  继承Tapable，负责整个webpack的编译过程，包含了一次构建过程中所有的数据，一次构建过程对应一个 Compilation 实例
  ```
  class Compilation extends Tapable {
  constructor(compiler) {
    super();
  }
  const options = compiler.options;
  this.options = options;
  this.entries = []; // 入口
  this.modules = []; // 所有模块
  this.chunks = []; // 代码块
  this.assets = {}; // 所有资源
  this.mainTemplate = new MainTemplate(this.outputOptions);
  this.chunkTemplate = new ChunkTemplate(this.outputOptions);
  this.runtimeTemplate = new RuntimeTemplate(
    this.outputOptions,
    this.requestShortener
  );
  this.moduleTemplates = {
    javascript: new ModuleTemplate(this.runtimeTemplate, "javascript"),
    webassembly: new ModuleTemplate(this.runtimeTemplate, "webassembly")
  };
}
  ```
#### 代码详细拆解
```
// 编译入口webpack/lib/Compiler.js
run(callback) {
  this.hooks.beforeRun.callAsync(this, err => {
    this.hooks.run.callAsync(this, err => {
      // 调用compile编译入口方法
      this.compile(onCompiled)
    }
  })
})
 
// compile编译入口方法
compile(callback) {
  const params = this.newCompilationParams();
  // 触发compiler beforeRun钩子
  this.hooks.beforeCompile.callAsync(params, err => {
    // 触发compiler compile钩子
    this.hooks.compile.call(params);
    // 初始化compilation编译对象
    const compilation = this.newCompilation(params);
    // 触发compiler make钩子，进行modules生成
    this.hooks.make.callAsync(compilation, err => {
     // 触发compiler finish钩子
      compilation.finish()
     // 触发compiler seal钩子
      compilation.seal(err => {
     // 触发compiler afterCompile钩子
        this.hooks.afterCompile.callAsync(compilation, err => {
          return callback(null, compilation)
        })
      })
  })
}
// 初始化normalModuleFactory、contextModuleFactory对象
newCompilationParams() {
  const params = {
    normalModuleFactory: this.createNormalModuleFactory(),
    contextModuleFactory: this.createContextModuleFactory(),
    compilationDependencies: new Set()
  };
  return params;
}
// 初始化compilation对象
newCompilation(params) {
  // 初始化compilation对象
  const compilation = this.createCompilation();
  // 触发compilation 钩子
  this.hooks.compilation.call(compilation, params);
  return compilation;
}
 
 
// 订阅compiler compilation钩子 webpack/lib/SingleEntryPlugin.js
class SingleEntryPlugin {
  apply(compiler) {
    // 订阅compiler compilation钩子
    compiler.hooks.compilation.tap("SingleEntryPlugin", (compilation, { normalModuleFactory }) => {
      // 设置SingleEntryDependency生成module工厂对象NormalModuleFactory
      compilation.dependencyFactories.set(
        SingleEntryDependency,
        normalModuleFactory
      )
    }
    // 订阅compiler make钩子
    compiler.hooks.make.tapAsync("SingleEntryPlugin", (compilation, callback) => {
      const { entry, name, context } = this;
      const dep = SingleEntryPlugin.createDependency(entry, name);
      // 正式进入构建阶段
      compilation.addEntry(context, dep, name, callback);
    }
  }
  // 创建单入口 dependency
  static createDependency(entry, name) {
    const dep = new SingleEntryDependency(entry);
    dep.loc = { name };
    return dep;
  }
}
```

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
### 生成modules
  入口模块、模块依赖对象Dependence, 经过对应的工厂对象(**Factory )转化为对应的模块实例(Module)
  <img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/build-module.png" width="830" height="300"/>
#### Module模块实例
数据结构：
 [{
    //...属性
    dependencies:  [{
       //...属性
       module:  {
         //...属性
         dependencies: [{
           // ... 不断递归
         }
       }]
   }
  #### 代码详细拆解
  ```
  // webpack/lib/Compilation.js
// addEntry方法
addEntry(context, entry, name, callback) {
  // 调用_addModuleChain方法
  this._addModuleChain(
    context,
    entry,
    module => {
      this.entries.push(module);
    },
    (err, module) => {
       if (module) {
         slot.module = module;
       } else {
         const idx = this._preparedEntrypoints.indexOf(slot);
         if (idx >= 0) {
           this._preparedEntrypoints.splice(idx, 1);
         }
       }
      return callback(null, module);
   })
}
 
// _addModuleChain方法
_addModuleChain(context, dependency, onModule, callback) {
  const Dep = /** @type {DepConstructor} */ (dependency.constructor);
  // 根据依赖查找对应的工厂函数
  const moduleFactory = this.dependencyFactories.get(Dep);
    this.semaphore.acquire(() => {
      // 调用工厂函数NormalModuleFactory的create来生成NormalModule对象
      moduleFactory.create(
        {
          dependencies: [dependency]
        },
        (err, module) => {
           // 回调存入compilation modules中
           const addModuleResult = this.addModule(module);
           module = addModuleResult.module;
           onModule(module);
           dependency.module = module;
 
 
           const afterBuild = () => {
             if (addModuleResult.dependencies) {
               // 递归处理dependencies依赖
               this.processModuleDependencies(module）
             }
           }
 
 
           this.buildModule(module, false, null, null, err => {
             this.semaphore.release();
             // 调用afterBuild递归处理dependencies依赖
             afterBuild();
           })
        })
    }
 
// buildModule方法
buildModule(module, optional, origin, dependencies, thisCallback) {
   // 调用NormalModule.build()
   module.build(
     this.options,
     this,
     this.resolverFactory.get("normal", module.resolveOptions),
     this.inputFileSystem
   )
}
 
 
// webpack/lib/NormalModuleFactory.js
// 订阅NormalModuleFactory.hooks.factory
this.hooks.factory.tap("NormalModuleFactory", () => (result, callback) => {
  // 生成NormalModule对象
  createdModule = new NormalModule(result);
  return callback(null, createdModule);
})
 
 
create(data, callback) {
  // 触发NormalModuleFactory.hooks.factory
  const factory = this.hooks.factory.call(null);
  factory(result, (err, module) => {
    callback(null, module);
  });
}
  ```
  
  #### loader处理源文件、转化AST
 ```
 // webpack/lib/NormalModule.js
class NormalModule extends Module {
  // build方法
  build(options, compilation, resolver, fs, callback) {
    this.buildInfo = {
      cacheable: false,
      fileDependencies: new Set(),
      contextDependencies: new Set()
    }
    // 调用doBuild解析模块源码source
    return this.doBuild(options, compilation, resolver, fs, err => {
      // acorn将JS解析为AST，解析模块依赖关系，后续module递归遍历
      const result = this.parser.parse(
      this._ast || this._source.source(),
      {
         current: this,
         module: this,
         compilation: compilation,
         options: options
      }
   }
 
 
   // doBuild方法
   doBuild(options, compilation, resolver, fs, callback) {
     // loader转化为标准js模块
     runLoaders(
      {
        resource: this.resource,
        readResource: fs.readFile.bind(fs)
      }, (err, result) => {
        if (result) {
          this.buildInfo.cacheable = result.cacheable;
          this.buildInfo.fileDependencies = new Set(result.fileDependencies)
        }
        const resourceBuffer = result.resourceBuffer;
        const source = result.result[0];
        const sourceMap = result.result.length >= 1 ? result.result[1] : null;
        // loader模块转化后_source
        this._source = this.createSource(
          this.binary ? asBuffer(source) : asString(source),
          resourceBuffer,
          sourceMap
        );
        this._ast =
          typeof extraInfo === "object" &&
          extraInfo !== null &&
          extraInfo.webpackAST !== undefined
          ? extraInfo.webpackAST
          : null;
    }
 ```
### 生成chunks
  把modules生成chunk，生成最终源码存放在compilation.assets属性上
  <img src="https://github.com/zhoulijunFE/webpack-learn/blob/master/static/build-chunk.png" width="700" height="500" />
#### Chunk
  * 配置在entry入口模块， MainTemplate模板render方法渲染源码
  * 依赖引入的模块（require/import进来的），chunkTemplate模板render方法渲染源码
#### 代码详细拆解
 ```
 // webpack/lib/Compilation.js
seal(callback) {
  // 创建入口模块chunk
  for (const preparedEntrypoint of this._preparedEntrypoints) {
    const module = preparedEntrypoint.module;
    const name = preparedEntrypoint.name;
    const chunk = this.addChunk(name);
  }
  // 递归创建依赖模块chunk
  this.processDependenciesBlocksForChunkGroups(this.chunkGroups.slice());
  // 生成assets
  this.createChunkAssets()
}
 
 
// createChunkAssets方法
createChunkAssets() {
  for (let i = 0; i < this.chunks.length; i++) {
    const chunk = this.chunks[i];
    // 判断是入口模块chunk、依赖模块chunk 调用不用的模块类方法
    const template = chunk.hasRuntime()
      ? this.mainTemplate
      : this.chunkTemplate;
    // 调用模板MainTemplate or ChunkTemplate的getRenderManifest方法
    const manifest = template.getRenderManifest({
      chunk,
      moduleTemplates: this.moduleTemplates,
      dependencyTemplates: this.dependencyTemplates
    });
    for (const fileManifest of manifest) {
      source = fileManifest.render();
      // chunk源码存放assets中
      this.assets[file] = source;
    }
  }
}
 
// webpack/lib/MainTemplate.js
  // getRenderManifest方法
  getRenderManifest(options) {
    const result = [];
    // 触发 mainTemplate.hooks.renderManifest钩子
    this.hooks.renderManifest.call(result, options);
    return result;
  }
  // render方法
  render(hash, chunk, moduleTemplate, dependencyTemplates) {
    // 触发mainTemplate.hooks.render钩子
    let source = this.hooks.render.call(
      new OriginalSource(
        Template.prefix(buf, " \t") + "\n",
        "webpack/bootstrap"
      ),
      chunk,
      hash,
      moduleTemplate,
      dependencyTemplates
  );
 // 订阅mainTemplate.hooks.render钩子，生成源代码
 this.hooks.render.tap(
   "MainTemplate",
   (bootstrapSource, chunk, hash, moduleTemplate, dependencyTemplates) => {
     const source = new ConcatSource();
     source.add("/******/ (function(modules) { // webpackBootstrap\n");
     source.add(new PrefixSource("/******/", bootstrapSource));
     source.add("/******/ })\n");
     source.add("/************************************************************************/\n");
     source.add("/******/ (");
     source.add(
      this.hooks.modules.call(
        new RawSource(""),
        chunk,
        hash,
        moduleTemplate,
        dependencyTemplates
      )
    );
    source.add(")");
    return source;
  }
);
 
 
// webpack/lib/ChunkTemplate.js
   getRenderManifest(options) {
     const result = [];
     // 触发chunkTemplate.hooks.renderManifest钩子
     this.hooks.renderManifest.call(result, options);
     return result;
   }
 }
 
 
// webpack/lib/JavascriptModulesPlugin.js
// 订阅mainTemplate.hooks.renderManifest钩子
compilation.mainTemplate.hooks.renderManifest.tap("JavascriptModulesPlugin",
  (result, options) => {
     // 调用MainTemplate模板run方法
     compilation.mainTemplate.render(
       hash,
       chunk,
       moduleTemplates.javascript,
       dependencyTemplates
     )
  })
// 订阅chunkTemplate.hooks.renderManifest钩子
compilation.chunkTemplate.hooks.renderManifest.tap("JavascriptModulesPlugin",
  (result, options) => {
     this.renderJavascript(
       compilation.chunkTemplate,
       chunk,
       moduleTemplates.javascript,
       dependencyTemplates
  )
)}
renderJavascript(chunkTemplate, chunk, moduleTemplate, dependencyTemplates) {
  // Template.renderChunkModules
  const moduleSources = Template.renderChunkModules(
    chunk,
    moduleTemplate,
    dependencyTemplates   
 );
 
 
// webpack/lib/Template.js
static renderChunkModules(chunk, filterFn, moduleTemplate, dependencyTemplates) {
  const modules = chunk.getModules().filter(filterFn);
  const allModules = modules.map(module => {
    return {
      id: module.id,
      source: moduleTemplate.render(module, dependencyTemplates, {
        chunk
      })
    };
  });
}
 ```

### 文件输出
  按照output的配置，把构建结果输出到文件中
  ```
    // webpack/lib/Compiler.js
  this.emitAssets(compilation, err => {
    let content = source.source();
    this.outputFileSystem.writeFile(targetPath, content, callback);
  }
  ```
 

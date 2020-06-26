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

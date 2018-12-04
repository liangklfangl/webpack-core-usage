### 1.本章概述
到了本章节，本系列课程已经接近尾声了。在前面的章节中，我们论述了 webpack 的核心概念和使用， webpack-dev-server 的核心概念和使用， webpack 插件的编写， webpack 的 loader 的编写等等一系列 webpack 的核心内容。在本章节，我们主要讲述如何在此基础上写一个自己的打包工具，下面我们开始正文内容。

### 2.webpack的三种打包策略
#### 2.1 webpack的一次性打包模式
我们前面已经讲过，调用 Compiler 对象的 run 方法可以开始 webpack 的打包过程，所以一次性打包的代码是极好编写的，你只需要执行下面的代码即可:
```js
//defaultWebpackConfig表示webpack的配置，注意需要入口文件
 const compiler = webpack(defaultWebpackConfig);
 compiler.run(doneHandler);
 //调用run方法开始打包并监听打包结果
 function doneHandler(err, stats) {
  if(stats.hasErrors()){
  	printErrors(stats.compilation.errors,true);
  }
  const warnings =stats.warnings && stats.warnings.length==0;
  if(stats.hasWarnings()){
  	printErrors(stats.compilation.warnings);
  }
 console.log("Compilation finished!\n");
}
function printErrors(errors,isError=false) {
  console.log("Compilation Errors or Warnings as follows:\n");
	const strippedErrors = errors.map(function(error) {
		return stripAnsi(error);
	});
	for(let i = 0; i < strippedErrors.length; i++)
		isError ? console.error(strippedErrors[i]) : console.warn(strippedErrors[i]);
}
```
其中在 doneHandler 方法中得到的 Stats 在前面的 Compiler 和 Compilation 章节已经讲过，你可以再去看看这部分的内容。下面给出一些配置，通过这些配置你可以进一步细粒度的控制前面说的 Stats 的展示内容:
```js
stats: {
  // fallback value for stats options when an option is not defined (has precedence over local webpack defaults)
  all: undefined,
  //添加assets资源信息，和chunks区别前面章节已经说过
  assets: true,
  //通过一个字段来对assets资源排序，而!field表示反向排序
  assetsSort: "field",
  // Add information about cached (not built) modules
  //添加哪些模块是缓存的
  cached: true,
  //显示缓存的assets，如果设置为false表示只显示那些缓存的输出资源
  cachedAssets: true,
  //添加children的信息
  children: true,
  //添加chunks的信息
  chunks: true,
  //将编译的模块信息添加到chunk的信息中
  chunkModules: true,
  //添加chunk的来源信息
  chunkOrigins: true,
  //通过一个字段来对chunks资源排序，而!field表示反向排序
  chunksSort: "field",
  //模块解析的context目录
  context: "../src/",
  // 和`webpack --colors`命令一致
  colors: true,
  //显示某一个模块和入口模块的距离(层级)
  depth: false,
  //显示某一个相应的bundle的入口文件
  entrypoints: false,
  //添加--env信息
  env: false,
  //添加errors信息
  errors: true,
  //添加错误的相信信息
  errorDetails: true,
  // Exclude assets from being displayed in stats
  // This can be done with a String, a RegExp, a Function getting the assets name
  // and returning a boolean or an Array of the above.
  excludeAssets: "filter" | /filter/ | (assetName) => ... return true|false |
    ["filter"] | [/filter/] | [(assetName) => ... return true|false],
  // Exclude modules from being displayed in stats
  // This can be done with a String, a RegExp, a Function getting the modules source
  // and returning a boolean or an Array of the above.
  excludeModules: "filter" | /filter/ | (moduleSource) => ... return true|false |
    ["filter"] | [/filter/] | [(moduleSource) => ... return true|false],
  // See excludeModules
  exclude: "filter" | /filter/ | (moduleSource) => ... return true|false |
    ["filter"] | [/filter/] | [(moduleSource) => ... return true|false],
  //增加编译的哈希值
  hash: true,
  //添加最多显示的模块数量的限制
  maxModules: 15,
  //添加编译的模块信息
  modules: true,
  //通过指定的字段对模块进行排序，你可以使用 `!field` 来反转排序。默认是按照 `id` 排序。
  modulesSort: "field",
  //显示模块依赖以及warning/errors产生的原因(2.5.0以后引入)
  moduleTrace: true,
  //当文件大小超过`performance.maxAssetSize`指定的值以后输出提示信息
  performance: true,
  // Show the exports of the modules
  providedExports: false,
  //添加publicPath的信息
  publicPath: true,
  //添加某一个模块被引入的原因
  reasons: true,
  //添加模块的源代码
  source: true,
  //添加模块的时间信息
  timings: true,
  //显示哪一个模块的exports属性被使用
  usedExports: false,
  //添加webpack版本信息
  version: true,
  //添加warning信息
  warnings: true,
  //webpack 2.4.0以后引入，通过这个函数可以过滤输出的warning信息。可以是String,Regexp,或者函数，该函数可以返回一个boolean值。当然该值也可以同时指定String,Regexp,或者函数，将返回第一个匹配的值
  warningsFilter: "filter" | /filter/ | ["filter", /filter/] | (warning) => ... return true|false
};
```
而具体的配置你可以通过传入 stats.toString/stats.toJson 方法来完成:
```js
var webpack = require("webpack");
webpack({
    //webpack配置
}, function(err, stats) {
        if (err) { throw new gutil.PluginError('webpack:build', err); }
        gutil.log('[webpack:build]', stats.toString({
            chunks: false, 
            colors: true
        }));
});
```

#### 2.2 webpack的watch模式
webpack 的 watch 模式表示当 webpack 的打包完成以后会继续监听文件的变化，从而重新打包。其相对于上面的一次性打包方式不同之处在于完成打包后并不是立即退出，而是继续监听依赖的文件的变化。其代码就是直接调用下面的* compiler.watch *方法:
```js
//defaultWebpackConfig表示webpack的配置，注意需要入口文件
 const compiler = webpack(defaultWebpackConfig);
 compiler.watch(delay, doneHandler);
 //调用watch方法开始打包并监听打包结果
 function doneHandler(err, stats) {
  //stats.hasErrors()表示是否有errors
  if(stats.hasErrors()){
  	printErrors(stats.compilation.errors,true);
  }
  //stats.hasWarnings()表示是否有warnings
  if(stats.hasWarnings()){
  	printErrors(stats.compilation.warnings);
  }
 console.log("Compilation finished!\n");
}
function printErrors(errors,isError=false) {
  console.log("Compilation Errors or Warnings as follows:\n");
	const strippedErrors = errors.map(function(error) {
		return stripAnsi(error);
	});
	for(let i = 0; i < strippedErrors.length; i++)
		isError ? console.error(strippedErrors[i]) : console.warn(strippedErrors[i]);
}
```
其实上面的 watch 方法还可以接收第二个参数，比如下面的例子:
```js
compiler.watch({ // watch options:
    aggregateTimeout: 300, // wait so long for more changes
    poll: true // use polling instead of native watchers
    // pass a number to set the polling interval
}, function(err, stats) {
    // ...
});
```
- aggregateTimeout
  该参数表示当一个文件发生变化以后，不是立即开始一轮新的编译，而是会等待 aggregateTimeout 毫秒。这样 webpack 就可以将很多文件的变化放在一次编译中完成。该参数的默认值为300ms。

- poll
  该参数表示轮询。他会每隔一定时间去检查文件是否发生了变化，如果发生了变化就会重新编译。你可以通过下面的方式来指定:
```js
poll: 1000
```
上面的配置表示每隔1s会监听文件的变化并重新打包。注意：我们的 Watching 模式对于网络文件系统是不适用的，如果 Watching 模式不适用你可以使用下我们的轮询。

- ignored
  在很多系统中，监听所有文件变化会消耗大量的 CPU 和内存占用，因此很多情况下我们会排除一些文件的监听，比如常见的 node_modules 文件夹。此时，你可以使用这里的 ignored 配置。
```js
ignored: /node_modules/
```
当然，你也可以使用下面这种深度匹配策略：
```js
ignored: "files/**/*.js"
```
此时不再监听 files 文件夹下的任何以 *.js* 结尾的文件的变化。调用 watch 方法后会得到一个 Watching 对象，该对象上也含有很多常用的方法。比如:

- close方法
  通过调用 Watching 对象的这个方法，我们会结束文件的监听操作。注意：如果当前的 Watching 对象没有调用 close/Invalidate ，那么不允许调用新的一轮的打包，比如 watch 或者 run 方法。其调用方式如下:
```js
watching.close(() => {
  console.log("Watching Ended.");
});
```

- Invalidate方法
  调用该方法表示本轮编译失效，但是并不会直接退出当前的文件监听。调用方式如下:
```js
watching.invalidate();
```

其实上面的这个 close 方法和 Invalidate 方法在 [webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware) 插件中很常用，而我们的 webpack-dev-server 内部也是直接封装了 webpack-dev-middleware ,如下:
```js
this.sockets = [];
this.contentBaseWatchers = [];
const webpackDevMiddleware = require('webpack-dev-middleware');
this.middleware = webpackDevMiddleware(compiler, options);
//方法1:webpack-dev-server的middleware方法
middleware: () => {
    app.use(this.middleware);
}
//方法2:webpack-dev-server直接调用webpack-dev-middleware的invalidate方法
Server.prototype.invalidate = function () {
  if (this.middleware) this.middleware.invalidate();
};
//方法3:webpack-dev-server直接调用webpack-dev-middleware的close方法
Server.prototype.close = function (callback) {
  this.sockets.forEach((sock) => {
    sock.close();
  });
  this.sockets = [];
  this.contentBaseWatchers.forEach((watcher) => {
    watcher.close();
  });
  this.contentBaseWatchers = [];
  this.listeningApp.kill(() => {
    this.middleware.close(callback);
  });
};
Server.prototype._watch = function (watchPath) {
  const watcher = chokidar.watch(watchPath).on('change', () => {
   //如果文件变化，那么通知所有的this.sockets集合中的socket，通知类型为'content-changed'
    this.sockWrite(this.sockets, 'content-changed');
    //客户端会通过下面的方式进行监听
    // 'content-changed': function contentChanged() {
    //    log.info('[WDS] Content base changed. Reloading...');
    //    self.location.reload();
    // }
  });
  this.contentBaseWatchers.push(watcher);
};
Server.prototype.sockWrite = function (sockets, type, data) {
  sockets.forEach((sock) => {
    sock.write(JSON.stringify({
      type,
      data
    }));
  });
};
```
上面看了 webpack-dev-server 是如何使用 webpack-dev-middleware 的，下面我们看看 webpack-dev-middleware 具体开放的 API :
```js
 var webpackDevMiddlewareInstance = webpackMiddleware(/* see example usage */);
 app.use(webpackDevMiddlewareInstance);
 //10s以后不再监听文件的变化
 setTimeout(function(){
   webpackDevMiddlewareInstance.close();
 }, 10000);
```
上面这个例子展示了某一个时间后我们不再监听文件的变化。
```js
 var compiler = webpack(/* see example usage */);
 var webpackDevMiddlewareInstance = webpackMiddleware(compiler);
 app.use(webpackDevMiddlewareInstance);
 setTimeout(function(){
   // After a short delay the configuration is changed
   // in this example we will just add a banner plugin:
   compiler.apply(new webpack.BannerPlugin('A new banner'));
   // Recompile the bundle with the banner plugin:
   webpackDevMiddlewareInstance.invalidate();
 }, 1000);
```
而这个例子展示了，*假如*我们的配置文件发生变化以后，我们可以通过调用 invalidate 方法重新开始打包工作(这个功能很常用,所以在[wcf](https://github.com/liangklfangl/wcf/blob/master/src/webpackWatch.js#L25)中当文件变化以后不应该是直接退出)。关于 [webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware)的更多用法你可以官网查看。通过这个例子你可以知道了我们的 webpack-dev-middleware具有如下的构造函数:
```js
module.exports = function(compiler, options) {
}
```
他会接受一个 Compiler 对象和用户配置作为参数,同时返回了一个中间件，这个中间件的签名为 *webpackDevMiddleware(req, res, next)* 类型，因此可以直接作为 Express 服务器的中间件使用( webpack-dev-server 本身就是一个 Express 服务器)。而其内部其实就是直接调用 Watching 对象的 close 或者 invalidate 方法而已，比如下面的 close 方法和 invalidate 方法:
```js
close: function(callback) {
    callback = callback || function() {};
    if(context.watching) context.watching.close(callback);
    else callback();
}
```
下面是调用 Watching 方法的 invalidate 方法:
```js
invalidate: function(callback) {
    callback = callback || function() {};
    if(context.watching) {
        share.ready(callback, {});
        context.watching.invalidate();
    } else {
        callback();
  }
},
```
更多关于 [webpack-dev-server](https://webpack.js.org/configuration/dev-server/#devserver) 用法的内容你可以查看官网。

#### 2.3 webpack-dev-server模式
webpack-dev-server 模式需要我们首先添加 webpack 的 HMR 功能依赖的入口模块，并设置 server 启动的域名和端口号。接着，我们需要调用 webpack 方法获取到 Compiler 对象，接着把这个对象传入到我们的 webpack-dev-server 的实例中。这样，我们的服务器就可以监听文件的变化，并将打包的消息实时传递给前端页面实现 HMR 自动刷新。
```js
function startDevServer(wpOpt, options) {
  addDevServerEntrypoints(wpOpt, options);
  //第一步:添加webpack-dev-server的入口文件
  let compiler;
  try {
    compiler = webpack(wpOpt);
  } catch (e) {
    console.log("webpack compile error!");
    if (e instanceof webpack.WebpackOptionsValidationError) {
      console.error(colorError(options.stats.colors, e.message));
      process.exit(1); 
    }
    throw e;
  }
  //创建访问需要构建的域名
  const uri =
    createDomain(options) +
    (options.inline !== false || options.lazy === true
      ? "/"
      : "/webpack-dev-server/");
  let server;
  try {
      //第二步:获取Compiler对象并传入webpack-dev-server
    server = new WebpackDevServer(compiler, options);
  } catch (e) {
    const OptionsValidationError = require("webpack-dev-server/lib/OptionsValidationError");
    if (e instanceof OptionsValidationError) {
      console.error(colorError(options.stats.colors, e.message));
      process.exit(1);
    }
    throw e;
  }
  server.listen(options.port, options.host, function(err) {
    if (err) throw err;
    reportReadiness(uri, options);
  });
}
```
- 第一步:添加 webpack-dev-server 的入口文件

通过添加 "webpack/hot/only-dev-server" 或者 "webpack/hot/dev-server" 可以启动 HMR 等高级功能，这部分内容在前面章节已经详细论述过了。
```js
function createDomain(options) {
	const protocol = options.https ? "https" : "http";
	return options.public ? `${protocol}://${options.public}` : url.format({
		protocol: protocol,
		hostname: options.host,
		port: options.socket ? 0 : options.port.toString()
	});
};
module.exports = function addDevServerEntrypoints(webpackOptions, devServerOptions) {
  if(devServerOptions.inline !== false) {
    const domain = createDomain(devServerOptions);
    //创建启动的http服务器
    const devClient = [`${require.resolve("wds-hack")}?${domain}`];
     if(devServerOptions.hotOnly)
      devClient.push("webpack/hot/only-dev-server");
    else if(devServerOptions.hot)
      devClient.push("webpack/hot/dev-server");
    [].concat(webpackOptions).forEach(function(wpOpt) {
      if(typeof wpOpt.entry === "object" && !Array.isArray(wpOpt.entry)) {
        Object.keys(wpOpt.entry).forEach(function(key) {
          wpOpt.entry[key] = devClient.concat(wpOpt.entry[key]);
        });
      } else {
        wpOpt.entry = devClient.concat(wpOpt.entry);        
      }
    });
  }
};
```

- 第二步：获取Compiler对象并传入webpack-dev-server
```js
import WebpackDevServer from "webpack-dev-server/lib/Server";
//wpOpt表示webpack配置
var compiler = webpack(wpOpt);
var server = new WebpackDevServer(compiler, options);
  server.listen(options.port, options.host, function(err) {
    if (err) throw err;
    reportReadiness(uri, options);
  });
}
//启动服务器
```
通过上面的代码，我们就在 nodejs 中正常启动了 webpack-dev-server。而至于其他内容，通过上面的 watch 模式的分析你应该已经知道了，这里就不再说了。

### 3.本章小结
通过本章节的学习，你对于 webpack 的 watch 模式，webpack-dev-server 模式，和一次性打包模式的代码编写已经有了一个大概的认识。本章节的完整代码你可以[在这里](https://github.com/liangklfangl/wcf)获取，该脚手架虽然比较简单，但是牵涉的内容还是很多的，只要你弄懂了里面的内容，写一个自己的打包脚手架已经不是难事。而对于 webpack-dev-middleware 如果有更深的兴趣，你可以阅读我的[ webpack-dev-middle 源码分析](https://github.com/liangklfang/webpack-dev-middleware)文章，比如里面就包含了 webpack-dev-server 为什么可以将资源编译到内存中的分析:
```js
webpackDevMiddleware.fileSystem = context.fs;
setFs: function(compiler) {
    //compiler.outputPath必须提供一个绝对路径,其就是我们在output.path中配置的内容
    if(typeof compiler.outputPath === "string" && !pathIsAbsolute.posix(compiler.outputPath) && !pathIsAbsolute.win32(compiler.outputPath)) {
        throw new Error("`output.path` needs to be an absolute path or `/`.");
    }
    // store our files in memory
    var fs;
    var isMemoryFs = !compiler.compilers && compiler.outputFileSystem instanceof MemoryFileSystem;
    //是否是MemoryFileSystem实例
    if(isMemoryFs) {
        fs = compiler.outputFileSystem;
    } else {
        fs = compiler.outputFileSystem = new MemoryFileSystem();
    }
    context.fs = fs;
}
```
。最后，非常感谢你对于本系列课程的支持,如果有任何不对的地方欢迎在读者圈给我留言，我会及时修改。当然，如果有任何疑问，也欢迎讨论，共同进步！

### 1.本章概述
通过前面章节的内容，你对于 Webpack 的插件应该已经不陌生了,而且，对于 Webpack 很多高级的知识点应该都有了一定的了解。这包括 Webpack中的 Compiler 和 Compilation 对象，以及 Webpack 的插件原理。在本章节，我主要以官网提供的例子 FileListPlugin/HelloWorldPlugin 来说明如何写一个插件，而这部分内容在前面你应该已经都有了深入的了解，因此你应该会很轻松。同时，在本章节我也会给出 Webpack 中不同插件的类型与区别。但是，如果你想要写一个自己的 Webpack 的复杂插件，那么除了前面的内容以外，也要注意日常的积累。好了，下面开始我们的正文。

### 2.如何写一个 Webpack 的插件
Webpack 的插件机制将 Webpack 引擎的能力暴露给了开发者。使用 Webpack 内置的各种打包阶段钩子函数使得开发者能够引入他们自己的打包流程。写一个 Webpack 插件往往比写一个 Loader 复杂，因为你需要了解 Webpack 内部很多细节的部分。
#### 2.1 如何创建一个 Webpack 的插件
通过前面的章节你应该了解了，一个 Webpack 的插件其实包含以下几个条件:

- 一个 js 命名函数
- 在他的原型链上存在一个 apply 方法
- 为该插件指定一个 Webpack 的事件钩子函数
- 使用 Webpack 内部的实例对象( Compiler 或者 Compilation )具有的属性或者方法
- 当功能完成以后，我们需要执行 Webpack 的回调函数

比如下面的函数就具备了上面的条件，所以他是可以作为一个 Webpack 插件的:
```js
function MyExampleWebpackPlugin() {
};
MyExampleWebpackPlugin.prototype.apply = function(compiler) {
  //我们主要关注compilation阶段，即webpack打包阶段
  compiler.plugin('compilation', function(compilation , callback) {
    console.log("This is an example plugin!!!");
    //当该插件功能完成以后一定要注意回调callback函数
    callback();
  });
};
```

#### 2.2 Compiler和Compilation实例
在前面的章节，我已经深入的讲解了这部分的内容，我们下面总结性的给出两个对象的作用。

- *Compiler*对象
  这个 Compiler 对象代表了 Webpack 完整的可配置的的环境。这个对象在 Webpack 启动的时候会被创建，同时该对象也会被传入一些可控的配置，比如：Options, Loaders, Plugins。当插件被实例化的时候，会收到一个 Compiler 对象，通过这个对象你可以访问 Webpack 的内部环境。

- *Compilation*对象
  该对象在每次文件变化的时候都会被创建，因此会重新产生新的打包资源。我们的 Compilation 对象表示本次打包的模块，编译的资源，文件改变和监听的依赖文件的状态。而且该对象也会提供很多的回调点，我们的插件可以使用它来完成特定的功能。 而提供的钩子函数在前面的章节已经讲过了，此处不再赘述

#### 2.3 Hello World插件
比如下面是我们写的一个插件:
```js
//插件内部可以接受到该插件的配置参数
function HelloWorldPlugin(options) {
}
HelloWorldPlugin.prototype.apply = function(compiler) {
  //此处利用了Compiler提供的done钩子函数，作用前面已经说过
  compiler.plugin('done', function() {
    console.log('Hello World!');
  });
};
module.exports = HelloWorldPlugin;
```
那么在 Webpack 配置文件中你就可以通过下面的方式来进行配置:
```js
var HelloWorldPlugin = require('hello-world');
//已经发布到NPM
var webpackConfig = {
  plugins: [
    new HelloWorldPlugin({options: true})
  ]
};
```
我们前面已经说过，Webpack 插件最重要的就是 Compilation 和 Compiler 对象。我们看看在插件里面如何使用我们的 Compilation 对象:
```js
function HelloCompilationPlugin(options) {}

HelloCompilationPlugin.prototype.apply = function(compiler) {
  //使用Compiler对象的compilation钩子函数就可以获取Compilation对象
  compiler.plugin("compilation", function(compilation) {
   //使用Compilation注册回调
    compilation.plugin("optimize", function() {
      console.log("Assets are being optimized.");
    });
  });
};
module.exports = HelloCompilationPlugin;
```

#### 2.4 异步插件
上面你看到的 HelloWorld 插件是同步的，还有一种插件是异步的，我们看看异步插件如何编写:
```js
function HelloAsyncPlugin(options) {}

HelloAsyncPlugin.prototype.apply = function(compiler) {
  compiler.plugin("emit", function(compilation, callback) {
    // Do something async...
    setTimeout(function() {
      console.log("Done with async work...");
      callback();
    }, 1000);

  });
};
module.exports = HelloAsyncPlugin;
```
从这里你可以看出，异步插件和同步插件最大的不同在于，异步插件会传入一个 callback 参数，当你的插件完成的相应的功能以后，你必须回调这个 *callback* 函数。

当访问到 Webpack 的 Compiler 和 每次产生的 Compilation 对象的时候，你可以使用 Webpack 的引擎来完成任何事情。你可以重新处理已经存在的文件，创建自己的派生文件(你想要多产生的文件)，或者对将要产生的资源进行修改(HtmlWebpackPlugin)等等。比如我们在前面章节就已经讲述的下面的实例,该实例就是有效的利用了 Compiler 的文件输出 emit 阶段产生我们自己需要的文件:
```js
function FileListPlugin(options) {}
FileListPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {
    var filelist = 'In this build:\n\n';
    //compilation.assets和compilation.chunks前面已经说过
    for (var filename in compilation.assets) {
      filelist += ('- '+ filename +'\n');
    }
   //在compilation.assets中添加我们需要的资源
    compilation.assets['filelist.md'] = {
      source: function() {
        return filelist;
      },
      size: function() {
        return filelist.length;
      }
    };
    callback();
  });
};
module.exports = FileListPlugin;
```

### 3. Webpack 的插件类型
插件可以根据它注册的事件分成不同的类型。每一个特定的钩子函数决定了他会被如何执行,比如插件可以分为如下的类型:

- 同步插件
  
此时我们的 Tapable 实例通过下面的方式来执行插件
```js
applyPlugins(name: string, args: any...)
//或者
applyPluginsBailResult(name: string, args: any...)
```
这意味着每一个插件的回调函数将会被按照顺序*依次执行(观察者模式)*，并传入特定的参数*args*，这是插件的最简单的格式。很多有用的钩子函数比如`"compile"`, `"this-compilation"`都期望每一个插件同步执行。下面给出 Webpack 对于 compile 这个钩子函数的执行方式：
```js
Compiler.prototype.compile = function(callback) {
  self.applyPluginsAsync("before-compile", params, function(err) {
    self.applyPlugins("compile", params);
    //1.执行compile阶段，同步执行插件的方式
    var compilation = self.newCompilation(params);
    self.applyPluginsParallel("make", compilation, function(err) {
      compilation.finish();
      compilation.seal(function(err) {
        self.applyPluginsAsync("after-compile", compilation, function(err) {
        });
      });
    });
  });
};
```

- 瀑布流插件
  
这种类型的插件通过下面的方法来执行:
```js
applyPluginsWaterfall(name: string, init: any, args: any...)
```
此时，每一个插件都会将前一个插件的返回值作为参数输入，并传入自己的参数，这种插件必须考虑插件的执行顺序。第一个插件传入的第二个参数值为*init*，而最后一个插件的返回值作为applyPluginsWaterfall的返回值。这种插件的模式常用于 Webpack 的模板，比如
ModuleTemplate, ChunkTemplate。比如 ModuleTemplate 下就使用了如下的内容:
```js
const Template = require("./Template");
module.exports = class ModuleTemplate extends Template {
	constructor(outputOptions) {
		super(outputOptions);
	}
	render(module, dependencyTemplates, chunk) {
		const moduleSource = module.source(dependencyTemplates, this.outputOptions, this.requestShortener);
		const moduleSourcePostModule = this.applyPluginsWaterfall("module", moduleSource, module, chunk, dependencyTemplates);
		const moduleSourcePostRender = this.applyPluginsWaterfall("render", moduleSourcePostModule, module, chunk, dependencyTemplates);
    //1.必须考虑插件的执行顺序
		return this.applyPluginsWaterfall("package", moduleSourcePostRender, module, chunk, dependencyTemplates);
	}
	updateHash(hash) {
		hash.update("1");
		this.applyPlugins("hash", hash);
	}
};
```

- 异步插件
  
如果插件会被异步执行，那么应该使用下面的方式来完成:
```js
applyPluginsAsync(name: string, args: any..., callback: (err?: Error) -> void)
```
此时我们的插件处理函数调用的时候会传入args和签名为*(err?: Error) -> void*的回调函数。我们的处理函数将会按照*注册时候的顺序*被执行。而回调函数callback将会在所有的处理函数被调用以后调用。这种模式常常用于如"emit", "run"等钩子函数。比如下面的 Compiler 的 run 方法的具体逻辑。
```js
  //1.这里是异步的执行逻辑
 self.applyPluginsAsync("run", self, function(err) {
      if(err) return callback(err);
      self.readRecords(function(err) {
        if(err) return callback(err);
        //2.调用compile的回调函数
        self.compile(function onCompiled(err, compilation) {
         //其他代码逻辑
        });
      });
    });
```

- 异步瀑布流插件
 
此时所有的插件将会被异步执行，同时遵循瀑布流的方式。此时以下面的方式来调用:
```js
applyPluginsAsyncWaterfall(name: string, init: any, callback: (err: Error, result: any) -> void)
```
此时插件的回调函数在调用的时候传入当前的值，回调函数被调用的时候会有如下的签名*(err: Error, nextValue: any) -> void*。如果回调函数被调用了，那么*nextValue*就会成为下一个处理函数的当前值。第一个处理函数的当前值为*init*。当所有的处理函数都执行以后，回调函数会传入最后一个插件的返回值。如果任何一个处理函数传入了一个*err*，那么回调函数将会传入错误参数*err*，此时余下的所有的处理函数都不会被执行。这种模式常常用于如"before-resolve"或者 "after-resolve"。

- 异步序列化插件
  
这种模式和异步相同，但是区别在于只要一个插件失败，那么后续的插件都不会被调用。如果某一个插件传入了 Error 对象，那么回调函数的第一个参数将会是该 Error 对象，而且后续所有的回调函数都不会执行。这类插件的调用方式如下:
```js
applyPluginsAsyncSeries(name: string, args: any..., callback: (err: Error, result: any) -> void)
```
比如在 Compiler 的 seal 方法中将会有如下的调用逻辑:
```js
self.applyPluginsAsyncSeries("optimize-tree", self.chunks, self.modules, function sealPart2(err) {
 //"optimize-tree"是异步的，但是只要有一个插件失败那么后续都不会执行
);
```

- 平行插件
此时通过下面的方式来执行插件:
```js
applyPluginsParallel(name: string, args: any..., callback: (err?: Error) -> void)
applyPluginsParallelBailResult(name: string, args: any..., callback: (err: Error, result: any) -> void)
```
比如在 Compiler 的 compile 方法中将执行下面的代码:
```js
self.applyPluginsParallel("make", compilation, function(err) {
      compilation.finish();
      compilation.seal(function(err) {
        //其他逻辑
      });
    });
```
这个回调函数只有在*所有的插件注册的该钩子函数的回调没有抛出错误的时候*才会执行。如果任意一个插件注册的回调函数抛出了一个错误，那么这个 callback 也会被执行，同时传入该错误对象，同时其他插件注册的处理函数将会直接忽略。


### 4.Webpack插件调用顺序
Webpack 的源码中你经常会看到上面说的执行插件注册的方法，我们给出下面的seal方法的部分代码:
```js
seal(callback) {
  self.applyPlugins0("seal");
  self.applyPlugins0("optimize");
  while(self.applyPluginsBailResult1("optimize-modules-basic", self.modules) ||
    self.applyPluginsBailResult1("optimize-modules", self.modules) ||
    self.applyPluginsBailResult1("optimize-modules-advanced", self.modules));
  self.applyPlugins1("after-optimize-modules", self.modules);
  //这里是optimize module
  while(self.applyPluginsBailResult1("optimize-chunks-basic", self.chunks) ||
    self.applyPluginsBailResult1("optimize-chunks", self.chunks) ||
    self.applyPluginsBailResult1("optimize-chunks-advanced", self.chunks));
    //这里是optimize chunk
  self.applyPlugins1("after-optimize-chunks", self.chunks);
  //这里是optimize tree
  self.applyPluginsAsyncSeries("optimize-tree", self.chunks, self.modules, function sealPart2(err) {
    self.applyPlugins2("after-optimize-tree", self.chunks, self.modules);
    const shouldRecord = self.applyPluginsBailResult("should-record") !== false;
    self.applyPlugins2("revive-modules", self.modules, self.records);
    self.applyPlugins1("optimize-module-order", self.modules);
    self.applyPlugins1("advanced-optimize-module-order", self.modules);
    self.applyPlugins1("before-module-ids", self.modules);
    self.applyPlugins1("module-ids", self.modules);
    self.applyModuleIds();
    self.applyPlugins1("optimize-module-ids", self.modules);
    self.applyPlugins1("after-optimize-module-ids", self.modules);
    self.sortItemsWithModuleIds();
    self.applyPlugins2("revive-chunks", self.chunks, self.records);
    self.applyPlugins1("optimize-chunk-order", self.chunks);
    self.applyPlugins1("before-chunk-ids", self.chunks);
    self.applyChunkIds();
    self.applyPlugins1("optimize-chunk-ids", self.chunks);
    self.applyPlugins1("after-optimize-chunk-ids", self.chunks);
    self.sortItemsWithChunkIds();
    if(shouldRecord)
      self.applyPlugins2("record-modules", self.modules, self.records);
    if(shouldRecord)
      self.applyPlugins2("record-chunks", self.chunks, self.records);
    self.applyPlugins0("before-hash");
    self.createHash();
    self.applyPlugins0("after-hash");
    if(shouldRecord)
      self.applyPlugins1("record-hash", self.records);
    self.applyPlugins0("before-module-assets");
    self.createModuleAssets();
    if(self.applyPluginsBailResult("should-generate-chunk-assets") !== false) {
      self.applyPlugins0("before-chunk-assets");
      self.createChunkAssets();
    }
    self.applyPlugins1("additional-chunk-assets", self.chunks);
    self.summarizeDependencies();
    if(shouldRecord)
      self.applyPlugins2("record", self, self.records);
    self.applyPluginsAsync("additional-assets", err => {
      if(err) {
        return callback(err);
      }
      self.applyPluginsAsync("optimize-chunk-assets", self.chunks, err => {
        if(err) {
          return callback(err);
        }
        self.applyPlugins1("after-optimize-chunk-assets", self.chunks);
        self.applyPluginsAsync("optimize-assets", self.assets, err => {
          if(err) {
            return callback(err);
          }
          self.applyPlugins1("after-optimize-assets", self.assets);
          if(self.applyPluginsBailResult("need-additional-seal")) {
            self.unseal();
            return self.seal(callback);
          }
          return self.applyPluginsAsync("after-seal", callback);
        });
      });
    });
  });
}
```
而各个钩子函数执行的顺序你可以查看下面的内容:
```js
'before run'
  'run'
    compile:func//调用compile函数
        'before compile'
           'compile'//(1)compiler对象的第一阶段
               newCompilation:object//创建compilation对象
               'make' //(2)compiler对象的第二阶段 
                    compilation.finish:func
                       "finish-modules"
                    compilation.seal
                         "seal"
                         "optimize"
                         "optimize-modules-basic"
                         "optimize-modules-advanced"
                         "optimize-modules"
                         "after-optimize-modules"//首先是优化模块
                         "optimize-chunks-basic"
                         "optimize-chunks"//然后是优化chunk
                         "optimize-chunks-advanced"
                         "after-optimize-chunks"
                         "optimize-tree"
                            "after-optimize-tree"
                            "should-record"
                            "revive-modules"
                            "optimize-module-order"
                            "advanced-optimize-module-order"
                            "before-module-ids"
                            "module-ids"//首先优化module-order，然后优化module-id
                            "optimize-module-ids"
                            "after-optimize-module-ids"
                            "revive-chunks"
                            "optimize-chunk-order"
                            "before-chunk-ids"//首先优化chunk-order，然后chunk-id
                            "optimize-chunk-ids"
                            "after-optimize-chunk-ids"
                            "record-modules"//record module然后record chunk
                            "record-chunks"
                            "before-hash"
                               compilation.createHash//func
                                 "chunk-hash"//webpack-md5-hash
                            "after-hash"
                            "record-hash"//before-hash/after-hash/record-hash
                            "before-module-assets"
                            "should-generate-chunk-assets"
                            "before-chunk-assets"
                            "additional-chunk-assets"
                            "record"
                            "additional-assets"
                                "optimize-chunk-assets"
                                   "after-optimize-chunk-assets"
                                   "optimize-assets"
                                      "after-optimize-assets"
                                      "need-additional-seal"
                                         unseal:func
                                           "unseal"
                                      "after-seal"
                    "after-compile"//(4)完成模块构建和编译过程(seal函数回调)    
    "emit"//(5)compile函数的回调,compiler开始输出assets，是改变assets最后机会
    "after-emit"//(6)文件产生完成
```

### 5.本章小结
在本章节，我们给出了不同种类的 Webpack 的插件类型与区别，同时给出了 Webpack 中各个钩子函数调用的顺序与时机，也给出了官网的一个关于 HelloWorld/FileListPlugin 插件的用法。虽然这个例子比较简单，但是可以对前面的内容做一个总结和回顾。如果你需要写出复杂的插件，通过前面章节的论述应该也不难。如果你对本章节的内容有疑问，你也可以查看我写的[Compiler 和 Compilation](https://github.com/liangklfangl/webpack-compiler-and-compilation)文章。
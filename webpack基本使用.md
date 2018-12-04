#### 1.本章概述
在本章节，我们将简要的对webpack的基本使用做一个演示。通过本章节的学习，你应该能在你的项目中快速配置一个webpack打包文件并完成项目的打包工作。在本章节，我们不牵涉到webpack中那些深入的知识点，后续我会针对这些知识点做更加细致的讲解。但是，正如在上一个章节看到的那样，我很早就在webpack配置中引入了CommonChunkPlugin,因此在本章节，我将在webpack配置文件中继续使用这个插件。希望你能对该插件引起足够的重视，并学会如何在自己的项目中使用它。

其中CommonChunkPlugin是webpack用于创建一个独立的文件，即所谓的common chunk。这个chunk会包含多个入口文件中共同的模块。通过将多个入口文件公共的模块抽取出来可以在特定的时间进行缓存，这对于提升页面加载速度是很好的优化手段。
#### 2.webpack打包例子讲解
##### 2.1 CommonChunkPlugin参数详解
开始具体的例子之前我们看下这个插件支持的配置和详细含义。同时，我们也给出官网描述的几个例子：
```js
{
  name: string, // or
  names: string[],
  // The chunk name of the commons chunk. An existing chunk can be selected by passing a name of an existing chunk.
  // If an array of strings is passed this is equal to invoking the plugin multiple times for each chunk name.
  // If omitted and `options.async` or `options.children` is set all chunks are used, otherwise `options.filename`
  // is used as chunk name.
  // When using `options.async` to create common chunks from other async chunks you must specify an entry-point
  // chunk name here instead of omitting the `option.name`.
  filename: string,
  //指定该插件产生的文件名称，可以支持output.filename中那些支持的占位符，比如[hash],[chunkhash],[id]等。如果忽略这个这个属性，那么原始的文件名称不会被修改(一般是output.filename或者output.chunkFilename，你可以查看compiler和compilation部分第一个例子)。但是这个配置不允许和`options.async`一起使用
  minChunks: number|Infinity|function(module, count)  boolean,
  //至少有minChunks的chunk都包含指定的模块，那么该模块就会被移出到common chunk中。这个数值必须大于等于2，并且小于等于没有使用这个插件应该产生的chunk数量。如果你传入`Infinity`，那么只会产生common chunk，但是不会有任何模块被移到这个chunk中(没有一个模块会被依赖无限次)。通过提供一个函数，你也可以添加自己的逻辑，这个函数会被传入一个参数表示产生的chunk数量
  chunks: string[],
  // Select the source chunks by chunk names. The chunk must be a child of the commons chunk.
  // If omitted all entry chunks are selected.
  children: boolean,
  // If `true` all children of the commons chunk are selected
  deepChildren: boolean,
  // If `true` all descendants of the commons chunk are selected

  async: boolean|string,
  // If `true` a new async commons chunk is created as child of `options.name` and sibling of `options.chunks`.
  // It is loaded in parallel with `options.chunks`.
  // Instead of using `option.filename`, it is possible to change the name of the output file by providing
  // the desired string here instead of `true`.
  minSize: number,
  //所有被移出到common chunk的文件的大小必须大于等于这个值
}
```
上面的filename和minChunks已经在注释中说明了，下面我们重点说一下其他的属性。

- children属性

  其中在webpack中很多chunk产生都是通过require.ensure来完成的。我们看看下面的例子:
```js
//main.js为入口文件
if (document.querySelectorAll('a').length) {
    require.ensure([], () => {
        const Button = require('./Components/Button').default;
        const button = new Button('google.com');
        button.render('a');
    });
}
if (document.querySelectorAll('h1').length) {
    require.ensure([], () => {
        const Header = require('./Components/Header').default;
        new Header().render('h1');
    });
}
```
此时会产生三个chunk，分别为main和其他两个通过require.ensure产生的chunk,比如0.entry.chunk.js和1.entry.chunk.js。如果我们配置了多个入口文件(假如还有一个main1.js)，那么这些动态产生的chunk中可能也会存在相同的模块(此时main1,main会产生四个动态chunk)。而这个children配置就是为了这种情况而产生的。通过配置children，我们可以将动态产生的这些chunk的公共的模块也抽取出来。但是，很显然，以前是动态加载的文件现在都必须在页面初始的时候就加载完成，那么对于初始加载肯在时间上有一定的副作用。但是存在一种情况，比如进入主页面后，我们需要加载路由A，路由B....等一系列的文件(网站的核心模块都要提前一次性加载)，那么我们把路由A，路由B...这些公共的模块提取到公有模块中，然后和入口文件一起加载，在性能上还是有优势的。下面是官网提供的一个例子:
```js
new webpack.optimize.CommonsChunkPlugin({
  // names: ["app", "subPageA"]
  // (choose the chunks, or omit for all chunks)
  children: true,
  // (select all children of chosen chunks)
  // minChunks: 3,
  // (3 children must share the module before it's moved)
})
```
对于common-chunk-plugin不太明白的，可以[查看这里](https://github.com/liangklfangl/commonsChunkPlugin_Config#将公共业务模块与类库或框架分开打包)。

- async
 
上面这种children的方案会增加初始加载的时间，这种async的方式相当于创建了一个异步加载的common-chunk，其包含我们require.ensure动态产生的chunk中的公共模块。这样，当你访问特定路由的时候，我们会动态的加载这个common chunk，以及你特定路由包含的业务代码。下面也是官网给出的一个实例:
```js
new webpack.optimize.CommonsChunkPlugin({
  name: "app",
  // or
  names: ["app", "subPageA"]
  // the name or list of names must match the name or names
  // of the entry points that create the async chunks
  children: true,
  // (use all children of the chunk)
  async: true,
  // (create an async commons chunk)
  minChunks: 3,
  // (3 children must share the module before it's separated)
})
```

- names
  
  该参数用于指定common chunk的名称。如果指定的chunk名称在entry中有配置，那么表示选择特定的chunk。如果指定的是一个数组，那么相当于按照名称的顺序多次执行common-chunk-plugin插件。如果没有指定name属性，但是指定了`options.async` 或者`options.children`，那么表示抽取所有的chunk的公共模块，包括通过require.ensure动态产生的模块。其他情况下我们使用`options.filename`来作为chunk的名称。注意:如果你指定了`options.async`来创建一个异步加载的common chunk，那么你必须指定一个入口chunk名称，而不能忽略option.name参数。你可以[点击这个例子](https://github.com/liangklfangl/commonsChunkPlugin_Config#将公共业务模块与类库或框架分开打包)查看。

- chunks

  通过chunks参数来选择来源的chunk。这些chunk必须是common-chunk的子级chunk。如果没有指定，那么默认选中所有的入口chunk。下面给出一个例子:
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: process.cwd()+'/example6/main.js',
        main1: process.cwd()+'/example6/main1.js',
        jquery:["jquery"]
    },
    output: {
        path: process.cwd()  + '/dest/example6',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: "jquery",
            minChunks:2,
            chunks:["main","main1"]
        })
    ]
};
```
此时你会发现在我们的jquery.js的最后会打包进来我们的chunk1.js和chunk2.js
```js
/* 2 */
/***/ function(module, exports, __webpack_require__) {
  __webpack_require__(3);
  var chunk1=1;
  exports.chunk1=chunk1;

/***/ },
/* 3 */
/***/ function(module, exports) {
  var chunk2=1;
  exports.chunk2=chunk2;

/***/ }
```
关于chunks配置的使用你可以[点击这里查看](https://github.com/liangklfangl/commonsChunkPlugin_Config#参数chunks)。所以，chunks就是用于指定从哪些chunks来抽取公共的模块，而chunks的名称一般都是通过entry来指定的，比如上面的entry为:
```js
 entry: {
        main: process.cwd()+'/example6/main.js',
        main1: process.cwd()+'/example6/main1.js',
        jquery:["jquery"]
    },
```
而chunks指定的为:
```js
 chunks:["main","main1"]
```
表明从main,main1两个入口chunk中来找公共的模块。


- deepChildren
 
  如果将该参数设置为true,那么common-chunk下的所有的chunk都会被选中，比如require.ensure产生的chunk的子级chunk。从这些chunks中来抽取公共的模块。

- minChunks为函数

 你可以给*minChunks*传入一个函数。CommonsChunkPlugin将会调用这个函数并传入module和count参数。这个module参数用于指定某一个chunks中所有的模块，而这个chunk的名称就是上面你配置的name/names参数。这个module是一个[NormalModule](https://github.com/webpack/webpack/blob/master/lib/NormalModule.js)实例，他有如下的常用属性:

 *module.context*:表示存储文件的路径，比如'/my_project/node_modules/example-dependency'


*module.resource*: 表示被处理的文件名称。比如'/my_project/node_modules/example-dependency/index.js'

而我们的count参数表示指定的模块出现在多少个chunk中。这个函数对于你细粒度的操作CommonsChunk插件还是很有用的。你可自己决定将那些模块放在指定的common chunk中,下面是官网给出的一个例子:
```js
new webpack.optimize.CommonsChunkPlugin({
  name: "my-single-lib-chunk",
  filename: "my-single-lib-chunk.js",
  minChunks: function(module, count) {
    //如果一个模块的路径中存在somelib部分，而且这个模块出现在3个独立的chunk或者entry中，那么它就会被抽取到一个独立的chunk中，而且这个chunk的文件名称为"my-single-lib-chunk.js"，而这个chunk本身的名称为"my-single-lib-chunk"
    return module.resource && (/somelib/).test(module.resource) && count === 3;
  }
});
```
而官网下面的例子详细的展示了如何将node_modules下引用的模块抽取到一个独立的chunk中:
```js
new webpack.optimize.CommonsChunkPlugin({
  name: "vendor",
  minChunks: function (module) {
    // this assumes your vendor imports exist in the node_modules directory
    return module.context && module.context.indexOf("node_modules") !== -1;
  }
})
```
因为node_module下的模块一般都是来源于第三方，所以在本地很少修改，通过这种方式可以将第三方的模块抽取到公共的chunk中。

还有一种情况就是，如果你想把应用的css/scss和vendor的css(第三方类库的css)抽取到一个独立的文件中，那么你可以使用下面的minChunk函数，同时配合ExtractTextPlugin来完成。
```js
new webpack.optimize.CommonsChunkPlugin({
  name: "vendor",
  minChunks: function (module) {
    // This prevents stylesheet resources with the .css or .scss extension
    // from being moved from their original chunk to the vendor chunk
    if(module.resource && (/^.*\.(css|scss)$/).test(module.resource)) {
      return false;
    }
    return module.context && module.context.indexOf("node_modules") !== -1;
  }
})
```
这个例子在抽取node_modules下的模块的时候做了一个限制，即明确指定node_modules下的scss/css文件不会被抽取，所以我们最后生成的vendor.js不会包含第三方类库的css/scss文件，而只包含其中的js部分。 同时通过配置ExtractTextPlugin就可以将我们应用的css和第三方应用的css抽取到一个独立的css文件中，从而达到css和js分离。

其中CommonsChunkPlugin插件还有一个更加有用的配置，即用于将webpack打包逻辑相关的一些文件抽取到一个独立的chunk中。但是此时配置的name应该是entry中不存在的，这对于线上缓存很有作用。因为如果文件的内容不发生变化，那么chunk的名称不会发生变化，所以并不会影响到线上的缓存。比如下面的例子:
```js
new webpack.optimize.CommonsChunkPlugin({
  name: "manifest",
  //这个name必须不在entry中
  minChunks: Infinity
})
```
但是你会发现我们抽取manifest文件和配置vendor chunk的逻辑不一样，所以这个插件需要配置两次:
```js
[
  new webpack.optimize.CommonsChunkPlugin({
    name: "vendor",
    minChunks: function(module){
      return module.context && module.context.indexOf("node_modules") !== -1;
    }
  }),
  new webpack.optimize.CommonsChunkPlugin({
    name: "manifest",
    minChunks: Infinity
  }),
]
```
你可能会好奇，假如我们有如下的配置:
```js
  module.exports = {
    entry: './src/index.js',
    plugins: [
      new CleanWebpackPlugin(['dist']),
      new HtmlWebpackPlugin({
       title: 'Caching'
      })
    ],
    output: {
    filename: '[name].[chunkhash].js',
      path: path.resolve(__dirname, 'dist')
    }
  };
```
那么如果entry中文件的内容没有发生变化，你运行webpack命令多次，那么最后生成的文件名称应该是一样的，为什么会重新通过CommonsChunkPlugin来生成一个manifest文件呢？这个官网也有明确的说明：

 因为webpack在入口文件中会包含特定的样板文件，特别是runtime文件和manifest文件。而最后生成的文件的名称到底是否一致还与webpack的版本有关，在新版本中可能不存在这个问题，但是在老版本中可能会存在，所以为了安全起见我们一般都会使用它。那么问题又来了，样板文件和runtime文件指的是什么？你可以查[我的这个例子](https://github.com/liangklfangl/commonsChunkPlugin_Config#将公共业务模块与类库或框架分开打包)，把关注点放在文中说的为什么要提前加载最后一个chunk的问题上。下面我们就这部分做一下深入的分析:

 - Runtime
   
   当你的代码在浏览器中运行的时候，webpack使用Runtime和manifest来处理你应用中的模块化关系。其中包括在模块存在依赖关系的时候，加载和解析特定的逻辑，而解析的模块包括已经在浏览中加载完成的模块和那些需要懒加载的模块本身。

- Manifest
  
  一旦你的应用程序中，如index.html文件、一些 bundle 和各种静态资源被加载到浏览器中，会发生什么？你精心安排的 /src 目录的文件结构现在已经不存在，所以webpack如何管理所有模块之间的交互呢？这就是 manifest数据用途的由来……

  当编译器(compiler)开始执行、解析和映射应用程序时，它会保留所有模块的详细要点。这个数据集合称为 "Manifest"，当完成打包并发送到浏览器时，会在运行时通过Manifest来解析和加载模块。无论你选择哪种模块语法，那些 import 或 require 语句现在都已经转换为 __webpack_require__ 方法，此方法指向模块标识符(module identifier)。通过使用manifest中的数据，runtime 将能够查询模块标识符，检索出背后对应的模块。比如我提供的[这个例子](https://github.com/liangklfangl/commonsChunkPlugin_Config#单入口文件时候不能把引用多次的模块打印到commonchunkplugin中)，其中入口文件中有main.js，入口文件中加载chunk1.js和chunk2.js,而最后你看到的就是下面转化为__webpack_require__后的内容:
```js
  webpackJsonp([0,1],[
/* 0 */
/***/ function(module, exports, __webpack_require__) {
    __webpack_require__(1);
    __webpack_require__(2);
/***/ },
/* 1 */
/***/ function(module, exports, __webpack_require__) {

    __webpack_require__(2);
    var chunk1=1;
    exports.chunk1=chunk1;
/***/ },
/* 2 */
/***/ function(module, exports) {
    var chunk2=1;
    exports.chunk2=chunk2;
/***/ }
]);
```
而manifest文件的作用就是在运行的时候通过__webpack_require__后的模块标识(Module identifier)来加载指定的模块内容。比如[manifest例子](https://github.com/liangklfangl/commonsChunkPlugin_Config/blob/master/dest/example8/manifest.json)生成的manifest.json文件内容是如下的格式:
```js
{
  "common.js": "common.js",
  "main.js": "main.js",
  "main1.js": "main1.js"
}
```
这样就可以在源文件和目标文件之间有一个映射关系，而这个映射关系本身依然存在于我们打包后的输出目录，而不会因为src目录消失了而不知道具体的模块对应关系。而至于其中moduleId等的对应关系是由webpack自己维护的，通过打包后的[可视化](https://github.com/liangklfangl/commonchunkplugin-source-code/raw/master/7.png)你可以了解。

##### 2.2 CommonChunkPlugin无法抽取单入口文件公共模块
上面讲了webpack官网提供的例子以及原理分析，下面我们通过自己构造的几个例子来深入理解上面的概念。假如我们有如下的webpack配置文件:
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry: 
  {
    main:process.cwd()+'/example1/main.js',
  },
  output: {
    path:process.cwd()+'/dest/example1',
    filename: '[name].js'
  },
  devtool:'cheap-source-map',
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
       minChunks:2
   })
  ]
};
```
下面是我们的入口文件内容:
```js
//main.js
require("./chunk1");
require("./chunk2");
console.log('main1.');
```
其中chunk1.js内容如下:
```js
require("./chunk2");
var chunk1=1;
exports.chunk1=chunk1;
```
而chunk2.js内容如下:
```js
var chunk2=1;
exports.chunk2=chunk2;
```
我们引入了CommonsChunkPlugin，并将那些引入了两次以上的模块输出到chunk.js中。那么你肯定会认为，因为chunk2.js被引入了两次，那么它肯定会被插件抽取到chunk.js中，但是实际上并不是这样。你可以查看*main.js*，他的内容如下:
```js
webpackJsonp([0,1],[
/* 0 */
/***/ function(module, exports, __webpack_require__) {
    __webpack_require__(1);
    __webpack_require__(2);
/***/ },
/* 1 */
/***/ function(module, exports, __webpack_require__) {
    __webpack_require__(2);
    var chunk1=1;
    exports.chunk1=chunk1;

/***/ },
/* 2 */
/***/ function(module, exports) {
    var chunk2=1;
    exports.chunk2=chunk2;

/***/ }
]);
```
通过这个例子我需要告诉你:'单入口文件时候不能把引用多次的模块打印到CommonChunkPlugin中'。

##### 2.3 CommonChunkPlugin抽取多入口文件公共模块
假如我们有如下的webpack配置文件:
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
  entry: 
  {
      main:process.cwd()+'/example2/main.js',
      main1:process.cwd()+'/example2/main1.js',
  },
  output: {
    path:process.cwd()+'/dest/example2',
    filename: '[name].js'
  },
  plugins: [
   new CommonsChunkPlugin({
       name:"chunk",
       minChunks:2
   })
  ]
};
```
其中main1.js内容如下:
```js
require("./chunk1");
require("./chunk2");
```
而main.js内容如下:
```js
require("./chunk1");
require("./chunk2");
```
而chunk1.js内容如下:
```js
require("./chunk2");
var chunk1=1;
exports.chunk1=chunk1;
```
而chunk2.js内容如下:
```js
var chunk2=1;
exports.chunk2=chunk2;
```
此时，很显然我们采用的是多入口文件模式，在相应的目录下会生成main.js和main1.js，以及chunk.js，而chunk.js中抽取的是main.js和main1.js中被引入了两次以上的模块，很显然chunk1.js和chunk2.js都会被引入到chunk.js中,下面是chunk.js中的部分代码:
```js
/******/ ([
/* 0 */,
/* 1 */
/***/ function(module, exports, __webpack_require__) {
    __webpack_require__(2);
    var chunk1=1;
    exports.chunk1=chunk1;
/***/ },
/* 2 */
/***/ function(module, exports) {
    var chunk2=1;
    exports.chunk2=chunk2;

/***/ }
/******/ ]);
```

##### 2.4 CommonChunkPlugin分离业务代码与框架代码
假如我们有如下的webpack配置内容,同时chunk1,chunk2,main1,main的内容和上面保持一致。
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: process.cwd()+'/example3/main.js',
        main1: process.cwd()+'/example3/main1.js',
        common1:["jquery"],
        //只含有jquery.js
        common2:["vue"]
        //只含有vue.js和加载器代码
    },
    output: {
        path: process.cwd()+'/dest/example3',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: ["chunk",'common1','common2'],
            minChunks:2
            //引入两次以及以上的模块
        })
    ]
};
```
按照CommonsChunkPlugin的抽取公共代码的逻辑，我们会有如下的结果:
<pre>
1.chunk.js中保存的是main.js和main1.js的公共代码，即chunk1.js和chunk2.js 
2.common1.js中只有jquery.js
3.common2.js中只有vue.js，但是必须含有webpack的加载器代码 
</pre>
其实道理很简单，我们的chunk.js中只有chunk1.js和chunk2.js，而不存在被引入了两次的模块，最多引入次数的就是chunk2.js，所以common1.js只含有jquery.js。但是，正如前文所说，我们的common2.js必须最先加载。

##### 2.5 minChunks为Infinity配置
假如我们的webpack配置如下:
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: process.cwd()+'/example5/main.js',
        main1: process.cwd()+'/example5/main1.js',
        jquery:["jquery"]
        //minChunks: Infinity时候框架代码依然会被单独打包成一个文件
    },
    output: {
        path: process.cwd() + '/dest/example5',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: "jquery",
            minChunks:2//被引用两次及以上
        })
    ]
};
```
上面的文件输出将会是如下内容:
<pre>
1.main.js包含去掉的公共代码部分
2.main1.js包含去掉的公共代码部分
3.main1.js和main2.js的公共代码将会被打包到jquery.js中，即jquery.js包含jquery+公共的业务代码
</pre>

其实，这个配置稍微晦涩难懂一点,假如我们将上面的minChunks配置修改为"Infinity",那么结果将截然不同：
<pre>
1.main.js原样打包
2.main1.js原样打包
3.jquery包含jquery.js和webpack模块加载器    
</pre>
因为将minChunks设置为Infinity，也就是无穷大，那么main.js和main1.js中不存在任何模块被依赖的次数这么大，因此chunk.js和chunk1.js都不会被抽取出来。

##### 2.6 chunks指定那些入口文件中的公共模块会被抽取
我们继续修改webpack配置如下:
```js
var CommonsChunkPlugin = require("webpack/lib/optimize/CommonsChunkPlugin");
module.exports = {
    entry: {
        main: process.cwd()+'/example6/main.js',
        main1: process.cwd()+'/example6/main1.js',
        jquery:["jquery"]
    },
    output: {
        path: process.cwd()  + '/dest/example6',
        filename: '[name].js'
    },
    plugins: [
        new CommonsChunkPlugin({
            name: "jquery",
            minChunks:2,
            chunks:["main","main1"]
            //main.js和main1.js中都引用的模块才会被打包的到公共模块
        })
    ]
};
```
此时我们的chunks设置为*["main","main1"]*，表示只有main.js和main1.js中都引用的模块才会被打包的到公共模块，而且必须是依赖次数为2次以上的模块。因此结果将会如下:
<pre>
1.jquery.js中包含main1.js和main.js中公共的模块，即chunk1.js和chunk2.js,以及jquery.js本身
2.main1.js表示是去掉公共模块后的文件内容
3.main.js表示是去掉公共模块后的文件内容    
</pre>
我们也可以通过查看打包后的jquery.js看到结果验证，即jquery.js包含了jquery.js本身以及公共的业务代码:
```js
/* 2 */
/***/ function(module, exports, __webpack_require__) {
  __webpack_require__(3);
  var chunk1=1;
  exports.chunk1=chunk1;

/***/ },
/* 3 */
/***/ function(module, exports) {
  var chunk2=1;
  exports.chunk2=chunk2;

/***/ }
```

#### 3.本章小结
本章节主要通过7个例子展示了webpack配合CommonsChunkPlugin的打包结果，但是为什么结果是这样，我会在webpack常见插件原理分析章节进行深入的剖析。本章节所有的例子代码你可以[点击这里](https://github.com/liangklfangl/commonsChunkPlugin_Config)查看。文中的配置都是参考webpack2的，如果你使用的是webpack1，请升级。如果需要查看上面的例子的运行结果，请执行下面的命令:
```js
npm install webpack -g
git clone https://github.com/liangklfangl/commonsChunkPlugin_Config.git
webpack
//修改webpack.config.js并运行webpack命令
```


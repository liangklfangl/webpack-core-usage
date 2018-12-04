### 1.本章概述
通过前面章节的内容，你对于 Webpack 的基础知识应该有了一个总体的了解。下面我将以一个具体的实例来教你如何写一个 Webpack 的 Loader。这个 Loader 本身复用性并不高，他是我在开发中遇到的一个实际问题。通过这个例子的论述，我想你对于如何写一个 Webpack 的 Loader应该会有一个整体的把握。下面我们开始本章节的内容

### 2.写一个Webpack的Loader
假如我们如下的markdown文件，文件主要内容为如下(完整内容点击[这里](https://github.com/liangklfangl/astexample/blob/master/demo.md)):
```jsx
import { Button } from 'antd';
ReactDOM.render(
  <div>
    <Button type="primary">Primary</Button>
    <Button>Default</Button>
    <Button type="dashed">Dashed</Button>
    <Button type="danger">Danger</Button>
  </div>
, mountNode);
```
此时，我们希望使用 Webpack 的 Loader来加载这个markdown文件的内容。那么很显然，我们就是要写一个相应的Loader，比如我们在webpack.config.js中添加如下的配置:
```js
module.exports = {
  module:{
    rule:[
     {
          test: /\.md$/,
          loaders: [
            require.resolve("babel-loader"),
            require.resolve("./markdownDemoLoader.js")
          ]
        }]
  }
}
```
其中markdownDemoLoader.js就是我们需要完成的 Webpack 的 Loader。在这个Loader中我们有如下的代码:
```js
const loaderUtils =  require('loader-utils');
const Grob = require('grob-files');
const {p2jsonml} = require('./utils/pc2jsonml');
const transformer = require('./utils/transformer');
const Smangle = require('string-mangle');
const generator = require('babel-generator').default;
const pwd = process.cwd();
const fs = require('fs');
const path = require('path');
const util = require('util');
/**
 *第一个参数是markdown的内容
 */
module.exports = function markdown2htmlPreview (content){
  //缓存该模块
  if (this.cacheable) {
    this.cacheable();
  }
  const loaderIndex = this.loaderIndex;
  //打印this可以得到所有的信息，这里得到md处理文件的loader数组中，当前loader所在的下标
  const query = loaderUtils.getOptions(this);
  const lang = query&&query.lang || 'react-demo';
  //获取Loader的query字段
  const processedjsonml=Smangle.stringify(p2jsonml(content));
  //得到jsonml
  const astProcessed = `module.exports = ${processedjsonml}`;
  //每一个Loader导出的内容都是module.exports
  const res = transformer(astProcessed,lang);
  //将得到的jsonml内容进一步的处理
  const inputAst = res.inputAst;
  const imports = res.imports;
  for (let k = 0; k < imports.length; k++) {
    inputAst.program.body.unshift(imports[k]);
  const code = generator(inputAst, null, content).code;
  //回到ES6代码
  const processedCode= 'const React =  require(\'react\');\n' +
        'const ReactDOM = require(\'react-dom\');\n'+
        code;
  return processedCode;
  }
}
```
我们现在看看Loader编写中常用的方法:
```js
  if (this.cacheable) {
    this.cacheable();
  }
```
我们知道Loader加载的结果默认是缓存的，如果你不想缓存可以使用*this.cacheable(false);*去阻止缓存。一个可以缓存的Loader必须满足一定的条件，即当输入和模块依赖关系没有发生变化的情况下，输出默认是确定的。也就是说，这个模块除了通过*this.addDependency*添加的模块依赖以外没有任何其他的模块依赖。

```js
 const loaderIndex = this.loaderIndex;
```
这个表示当前Loader在加载特定文件的时候所在的下标。

在上面这个Loader中，我们首先原样传入markdown文件的内容，然后将它转化为jsonml，我们看看上面的jsonml.js的内容:
```js
const markTwain = require('mark-twain');
const path = require('path');
function p2jsonml(fileContent){
  const markdown = markTwain(fileContent);
  return markdown;
};
module.exports = {
  p2jsonml
}
```
转化为jsonml格式以后，我们将会得到如下的内容:
```js
module.exports = { "content": [ "article", [ "h3", "1.mark-twain解析出来的无法解析成为ast" ], [ "pre", { "lang": "jsx" }, [ "code", "import { Button } from 'antd';\nReactDOM.render(\n

\n <Button type="primary" shape="circle" icon="search" />\n <Button type="primary" icon="search">Search\n <Button shape="circle" icon="search" />\n <Button icon="search">Search\n 
\n <Button type="ghost" shape="circle" icon="search" />\n <Button type="ghost" icon="search">Search\n <Button type="dashed" shape="circle" icon="search" />\n <Button type="dashed" icon="search">Search\n
,\n mountNode\n);" ] ] ], "meta": {
} }
```
但是这并不是我们希望的结果，我们需要继续如下的处理，其中目的只有一个:将我们的ReactDOM.render中第一个参数的值放到一个独立的函数中，函数的名字为jsonmlReactLoader:
```js
const babylon = require('babylon');
const types = require('babel-types');
const traverse = require('babel-traverse').default;
function parser(content) {
  return babylon.parse(content, {
    sourceType: 'module',
    plugins: [
      'jsx',
      'flow',
      'asyncFunctions',
      'classConstructorCall',
      'doExpressions',
      'trailingFunctionCommas',
      'objectRestSpread',
      'decorators',
      'classProperties',
      'exportExtensions',
      'exponentiationOperator',
      'asyncGenerators',
      'functionBind',
      'functionSent',
    ],
  });
}
module.exports = function transformer(content, lang) {
  let imports = [];
  const inputAst = parser(content);
  traverse(inputAst, {
    ArrayExpression: function(path) {
      const node = path.node;
      const firstItem = node.elements[0];
      //tagName
      const secondItem = node.elements[1];
      //attributes or child element
      let renderReturn;
      if (firstItem &&
        firstItem.type === 'StringLiteral' &&
        firstItem.value === 'pre' &&
        secondItem.properties[0].value.value === lang) {
        let codeNode = node.elements[2].elements[1];
        let code = codeNode.value;
        //得到代码的内容了，也就是demo的代码内容
        const codeAst = parser(code);
        //继续解析代码内容~~~
        traverse(codeAst, {
          ImportDeclaration: function(importPath) {
            imports.push(importPath.node);
            importPath.remove();
          },
          CallExpression: function(CallPath) {
            const CallPathNode = CallPath.node;
            if (CallPathNode.callee &&
              CallPathNode.callee.object &&
              CallPathNode.callee.object.name === 'ReactDOM' &&
              CallPathNode.callee.property &&
              CallPathNode.callee.property.name === 'render') {
              //we focus on ReactDOM.render method
              renderReturn = types.returnStatement(
                 CallPathNode.arguments[0]
              );
              //we focus on first parameter of ReactDOM.render method
              CallPath.remove();
            }
          },
        });
        const astProgramBody = codeAst.program.body;
        const codeBlock = types.BlockStatement(astProgramBody);
        if (renderReturn) {
          astProgramBody.push(renderReturn);
        }
        const coceFunction = types.functionExpression(
          types.Identifier('jsonmlReactLoader'),
          [],
        );
        path.replaceWith(coceFunction);
      }
    },
  });
  return {
    imports: imports,
    inputAst: inputAst,
  };
};
```
经过上面的代码处理，你会清楚的看到我们的markdown文件内容变成了如下的格式了:
```js
const React =  require('react');
const ReactDOM = require('react-dom');
import { Button } from 'antd';
module.exports = {
  "content": ["article", ["h3", "1.mark-twain解析出来的无法解析成为ast"], function jsonmlReactLoader() {
    return <div>
    <Button type="primary" shape="circle" icon="search" />
    <Button type="primary" icon="search">Search</Button>
    <Button shape="circle" icon="search" />
    <Button icon="search">Search</Button>
    <br />
    <Button type="ghost" shape="circle" icon="search" />
    <Button type="ghost" icon="search">Search</Button>
    <Button type="dashed" shape="circle" icon="search" />
    <Button type="dashed" icon="search">Search</Button>
  </div>;
  }],
  "meta": {}
};
```
此时的模块依然是ES6格式与jsx混合的代码，我们需要进一步配合babel来处理将它转化为ES5代码，所以我们的webpack.config.js中才会在该插件后引入babel-loader来对代码进行进一步的打包。那么你可能会想，就算babel打包后，得到上面这样的代码会有什么用?我给你看看，在前端我是如何将这样的代码转化为React类型的:
```js
import ReactDOM from "react-dom";
import React from "react";
const content =  require('../../demos/basic.md');
const converters = [
      [
        function(node) { return typeof node === 'function'; },
        function(node, index) {
          return React.cloneElement(node(), { key: index });
        }
      ]
    ];
//(2)converters可以引入一个库来完成
const JsonML = require('jsonml.js/lib/utils');
const toReactComponent = require('jsonml-to-react-component');
ReactDOM.render(toReactComponent(content.content,converters), document.getElementById('react-content'));

```
是不是很容易理解了，我们Loader处理后的代码，最后会被我原样转化为React的组件并在页面中展示,当然这个过程必须经过[jsonml-to-react-element](https://github.com/benjycui/jsonml-to-react-element)的转化。所以说，我们的Loader完成了markdown文件类型到我们最后的javascript模块的转化。这就是 Webpack 中 Loader 的强大作用。

### 3.Webpack的Loader常见配置
在 Webpack 中 Loader 就是一个模块，该模块导出一个函数。我们的 Webpack 的 Loader 机制会调用这些函数，并将前一个函数的处理结果传递给下一个处理函数，而第一个函数接受到的就是文件的原始内容，比如上面的这个例子就是 markdown 文件的原样内容(与通过 Nodejs 中 fs 模块读取的内容一致)。在这个函数中的 *this* 对象会有各种有用的方法，你可以通过这些方法将 Loader 的调用形式转化为异步的(this.async方法)，或者得到该 Loader 的配置参数等等。

第一个 Loader 会被传入文件的原始内容，最后的一个 Loader 必须返回一个结果，这个结果可以是 String 或者 Buffer 类型，他们代表 JavaScript 模块的源代码。同时，一个可选的返回值就是 SourceMap 。如果只要返回一个值，那么可以是同步模式，如果需要返回多个值，那么必须调用*this.callback()*方法。在异步模式下，*this,async()*必须调用来通知 Webpack 的 Loader 执行器等待异步返回的结果。它返回*this.callback()*，同时该 Loader 必须返回 undefined同时调用该回调。

#### 3.1 同步的Loader
```js
module.exports = function(content) {
  return someSyncOperation(content);
};
```
下面是同步的Loader并返回多个值:
```js
module.exports = function(content) {
  this.callback(null, someSyncOperation(content), sourceMaps, ast);
  //如果要返回多个值必须通过this.callback方法
  return; 
  // always return undefined when calling callback()
  // 当调用this.callback时候必须返回undefined
};
```
####  3.2 异步Loader
```js
module.exports = function(content) {
    var callback = this.async();
    //*this,async()*必须调用来通知Webpack的Loader执行器等待异步返回的结果
    someAsyncOperation(content, function(err, result) {
        //必须返回undefined或者回调该callback函数
        if(err) return callback(err);
        callback(null, result);
    });
};

```
下面是异步的Loader并返回多个值的情况:
```js
module.exports = function(content) {
    var callback = this.async();
    //异步Loader
    someAsyncOperation(content, function(err, result, sourceMaps, ast) {
        if(err) return callback(err);
        callback(null, result, sourceMaps, ast);
    });
};
```

#### 3.3 "Raw" Loader
默认情况下，源文件的内容会被转化为UTF-8的字符串并传给我们的 Loader 。通过设置 *raw* 这个标志，那么我们的 Loader 会接受到一个 Buffer 对象。每一个Loader 都允许将他的结果以 String 或者 Buffer 的类型传递给下一个 Loader ，而 Webpack 可以将它在两者之间正常转化:
```js
module.exports = function(content) {
    assert(content instanceof Buffer);
    return someSyncOperation(content);
    // return value can be a `Buffer` too
    // This is also allowed if loader is not "raw"
};
module.exports.raw = true;
```

#### 3.4 Pitching Loader
我们的Loader默认都是从右边向左边执行的，但是在很多情况下，我们可能并不关心前一个Loader的执行结果或者输入资源。我们仅仅关系元数据，我们的Loader上的*pitch*方法就是在 Loader 被调用之前从左边往右边执行的。如果某一个Loader的pitch方法输出一个结果，那么打包过程就是逆转，同时跳过其他的Loader，并继续执行左侧的 Loader（左侧的Loader最后执行）。同时*data*可以在 pitch 方法和常规调用之间传递:
```js
module.exports = function(content) {
    return someSyncOperation(content, this.data.value);
};
module.exports.pitch = function(remainingRequest, precedingRequest, data) {
    if(someCondition()) {
        // fast exit
        // 此时允许我们只执行左侧的Loader而忽略后续的Loader
        return "module.exports = require(" + JSON.stringify("-!" + remainingRequest) + ");";
    }
    data.value = 42;
};
```
比如我们的[style-loader](https://github.com/liangklfang/style-loader/blob/master/index.js#L8)就指定了该pitch方法。

### 4.Webpack的Loader配置
一个 Webpack 的 Loader 的上下文表示在 Loader 中的 this 对象具有的那些属性，假如有如下的例子:
```js
require("./loader1?xyz!loader2!./resource?rrr");
```
假如我们在*/abc/file.js*这个文件中调用了上面的 require 方法。我们分析下常用的属性:

- this.version

  这表示 Loader 的 API 的版本。当前版本是2，该参数可用于向后兼容。使用 this.version 你可以指定自定义逻辑。

- this.context

  表示当前模块所在的目录。通过这个参数你可以获取该目录下的其他内容。在上面的例子中就是*/abc*这个目录。

- this.request

 已经解析后的请求字符串，比如上面的例子就是*"/abc/loader1.js?xyz!/abc/node_modules/loader2/index.js!/abc/resource.js?rrr"*。即，特定的Loader 已经转化为绝对路径。

- this.query
  
  如果一个 Loader 配置了 Options 对象，那么该参数就是指向这个对象。如果该 Loader 没有 Options 参数，但是配置了 query 字符串，那么该参数就是查询字符串，并以?开头。比如开头的例子可以通过 Options 来配置 Loader 具备的参数:
```js
module.exports = {
  module:{
    rule:[
      { test: /\.md$/, use:[{
        loader:'babel-loader'
      },{
        loader:"./markdownDemoLoader.js",
        options:{
         //指定该Loader的Options参数
        }
      }]
    }]
  }
}
```
- this.callback

 使用这个函数可以给我们的 Loader 返回多个结果，可以在同步或者异步的情况下调用。默认的参数类型是如下格式:
```js
this.callback(
    err: Error | null,
    content: string | Buffer,
    sourceMap?: SourceMap,
    abstractSyntaxTree?: AST
);
```
第一个参数可以是 Error 对象或者null;第二个参数是一个String或者Buffer;第三个参数可选，表示可以被该模块解析的SourceMap;第四个参数也是可选的，是一个AST抽象语法树，该参数Webpack本身会忽略，但是在不同的Loader之间共享AST可以提升打包的速度。调用该方法后必须返回undefined！关于抽象语法树的内容你可以继续阅读[这里](https://github.com/liangklfangl/astexample)的内容。

- this.async

调用该方法相当于告诉我们的 Loader 执行器我们需要调用异步的结果，返回的内容就是*this.callback*。比如下面的例子:
```js
module.exports = function(content) {
    var callback = this.async();
    //*this,async()*必须调用来通知Webpack的Loader执行器等待异步返回的结果
    someAsyncOperation(content, function(err, result) {
        //必须返回undefined或者回调该callback函数
        if(err) return callback(err);
        callback(null, result);
    });
};
```

- this.data
 
  表示在 Loader 的 pitch 方法和正常打包阶段共享的数据。

- this.cacheable

  默认情况下，每一个Loader加载的结果都是可以缓存的。你可以在调用该方法的时候传入false显示要求Loader不要缓存结果。一个可以缓存的Loader必须满足一定的条件，即当输入和依赖关系没有发生变化的情况下，输出必须是确定的。也就是说，该Loader除了*this.addDependency*指定的依赖以外，不能有其他的依赖模块。

- this.loaders

表示一个Loader数组，在pitch阶段是可以修改的。如:
```js
loaders = [{request: string, path: string, query: string, module: function}]
```
比如下面的例子:
```js
[
  {
    request: "/abc/loader1.js?xyz",
    path: "/abc/loader1.js",
    query: "?xyz",
    module: [Function]
  },
  {
    request: "/abc/node_modules/loader2/index.js",
    path: "/abc/node_modules/loader2/index.js",
    query: "",
    module: [Function]
  }
]
```

- this.loaderIndex

  表示当前Loader所在Loaders数组中的下标，比如上面的例子中*loader1*就是0，而*loader2*就是1。

- this.resource

 Loader加载的资源部分，包含query字段。如上面的例子就是:

```js
 "/abc/resource.js?rrr"
```

- this.resourcePath

  表示资源文件本身，比如上面的例子就是*"/abc/resource.js"*。

- this.resourceQuery

 表示资源的query部分。比如上面的例子就是* "?rrr"*。

- this.target

  表示将当前代码打包成的文件格式。可以是"web"或者"node"。

- this.webpack

  如果当前模块是被Webpack打包，那么值就是true。

- this.sourceMap

  表示是否应该产生sourceMap。因为产生sourceMap的花销是很昂贵的，所以你需要确定是否有必要产生。

- this.emitWarning

  产生一个警告消息。

- this.emitError

 产生一个错误信息。

- this.loadModule

比如下面的形式:
```js
loadModule(request: string, callback: function(err, source, sourceMap, module))
```
解析一个特定的加载请求成为对某一个模块的加载，同时使用产生的资源，sourceMap,模块实例(同行是NormalModule)来调用所有的Loader和回调函数。这个函数可以用于获取其他模块的内容并产生结果

- this.resolve
其中使用方法如下:
```js
resolve(context: string, request: string, callback: function(err, result: string))
```
相当于通过require方法来加载一个模块。

- this.addDependency
  
  可以通过下面的方法来完成
```js
addDependency(file: string)
dependency(file: string) // shortcut
```
将某一个文件作为该Loader的依赖，进而使得该文件任何变化可以被监听。比如[html-loader](https://github.com/webpack-contrib/html-loader)使用该技术来查看解析的html文件中依赖的其他含有*src*和*src-set*属性的资源，然后为这些属性添加url。

- this.addContextDependency

  将某一个目录作为Loader的依赖。用法如下：
```js
addContextDependency(directory: string)
```
此时，如果目录中资源发生变化，那么Loader本身的输出将会更新，上一次加载的资源的缓存失效。

- this.clearDependencies

  移除 Loader 所有的依赖。甚至自己和其它 Loader 的初始依赖。考虑使用 pitch。用法如下:
```js
clearDependencies()
```
但是，建议在pitch方法中完成这个功能。

- emitFile

用于输出一个文件。这是 webpack 特有的方法。用法如下:
```js
emitFile(name: string, content: Buffer|string, sourceMap: {...})
```
你可以查看[file-loader](https://github.com/webpack-contrib/file-loader/blob/master/src/index.js#L66)是如何输出一个特定的文件的。

- this.fs

  使用这个属性可以获取到*Compilation*实例上的*inputFileSystem*属性,其实际上是一个CachedInputFileSystem。其主要属性如下:
```js
CachedInputFileSystem {
  fileSystem: NodeJsInputFileSystem {},
  //NodeJsInputFileSystem
  _statStorage: 
  //_statStorage属性，保存加载的所有的模块资源
   Storage {
     duration: 60000,
     running: {},
     data: 
      { '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main1.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main1.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/main1.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/node_modules': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules': [Object],
        '/Users/qinliang.ql/Desktop/node_modules': [Object],
        '/Users/qinliang.ql/node_modules': [Object],
        '/Users/node_modules': [Object],
        '/node_modules': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk1': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk1.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk1.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk2': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk2.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/chunk-module-assets/chunk2.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/map.png': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/map.png.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/map.png.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/_url-loader@0.6.2@url-loader/index.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/_url-loader@0.6.2@url-loader/index.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/_url-loader@0.6.2@url-loader/index.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/index': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/index.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/index.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/index': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/index.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/index.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/dist/vue.runtime.common.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/dist/vue.runtime.common.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/vue/dist/vue.runtime.common.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/dist/jquery.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/dist/jquery.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/jquery/dist/jquery.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/process/browser.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/process/browser.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/process/browser.js.json': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/webpack/buildin/global.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/webpack/buildin/global.js.js': [Object],
        '/Users/qinliang.ql/Desktop/commonsChunkPlugin_Config/node_modules/webpack/buildin/global.js.json': [Object] },
     levels: 
      [ [Object] ],
     count: 48,
     interval: 
      Timeout {
        _called: false,
        _idleTimeout: 530,
        _idlePrev: [Object],
        _idleNext: [Object],
        _idleStart: 685,
        _onTimeout: [Function: bound ],
        _timerArgs: undefined,
        _repeat: 530 },
     needTickCheck: false,
     nextTick: null,
     passive: false,
     tick: [Function: bound ] },
  //_readdirStorage
  _readdirStorage: 
   Storage {
     duration: 60000,
     running: {},
     data: {},
     levels: 
      [],
     count: 0,
     interval: null,
     needTickCheck: false,
     nextTick: null,
     passive: true,
     tick: [Function: bound ] },
  //_readFileStorage
  _readFileStorage: 
   Storage {
     duration: 60000,
     running: {},
     data: 
      { 
        //已经删除
       },
     levels: 
      [ [Object]],
     count: 32,
     interval: 
      Timeout {
        _called: false,
        _idleTimeout: 530,
        _idlePrev: [Object],
        _idleNext: [Object],
        _idleStart: 678,
        _onTimeout: [Function: bound ],
        _timerArgs: undefined,
        _repeat: 530 },
     needTickCheck: false,
     nextTick: null,
     passive: false,
     tick: [Function: bound ] },
  //_statStorage与_readdirStorage,_readFileStorage
  _readJsonStorage: 
   Storage {
     duration: 60000,
     running: {},
     data: 
      { 
        //已经删除
      },
     levels: 
      [ [Object] ],
     count: 23,
     interval: 
      Timeout {
        _called: false,
        _idleTimeout: 530,
        _idlePrev: [Object],
        _idleNext: [Object],
        _idleStart: 679,
        _onTimeout: [Function: bound ],
        _timerArgs: undefined,
        _repeat: 530 },
     needTickCheck: false,
     nextTick: null,
     passive: false,
     tick: [Function: bound ] },
  //_readlinkStorage
  _readlinkStorage: 
   Storage {
     duration: 60000,
     running: {},
     data: 
      {
        //已经删除
      },
     levels: 
      [ [Object] ],
     count: 55,
     interval: 
      Timeout {
        _called: false,
        _idleTimeout: 530,
        _idlePrev: [Object],
        _idleNext: [Object],
        _idleStart: 684,
        _onTimeout: [Function: bound ],
        _timerArgs: undefined,
        _repeat: 530 },
     needTickCheck: false,
     nextTick: null,
     passive: false,
     tick: [Function: bound ] },
  _stat: [Function: bound bound ],
  _statSync: [Function: bound bound ],
  _readdir: [Function: bound readdir],
  _readdirSync: [Function: bound readdirSync],
  _readFile: [Function: bound bound readFile],
  _readFileSync: [Function: bound bound ],
  _readJson: [Function: bound ],
  _readJsonSync: [Function: bound ],
  _readlink: [Function: bound bound ],
  _readlinkSync: [Function: bound bound ] }
```
所以该对象其实就包含了_statStorage,_readdirStorage,_readFileStorage,_readJsonStorage,_readlinkStorage等几个存储相关的字段。而至于compiler.outputFileSystem你可以查看[webpack-dev-middleware](https://github.com/liangklfang/webpack-dev-middleware#4该插件的compileroutputfilesystem是一个memoryfilesystem实例)是如何使用它来将输出资源保存到内存中而不是文件系统中的。上面这个输出实例来自于[这个文件](https://github.com/liangklfangl/commonsChunkPlugin_Config/blob/master/chunk-module-assets/FileListPlugin.js)，你可以自己运行并查看结果。

### 5.本章总结
本章节，我们通过一个 markdown 的 loader 的具体事例展了如何写一个 Webpack 的 loader。同时也给出了 loader 常见的配置和用法。通过本章节的学习，你应该能够写一个基础的 Webpack 的 loader。本章节的完整实例代码你可以查看[Webpack 操作 AST](https://github.com/liangklfangl/astexample)，但是因为这个 Loader 牵涉到了如何操作我们的 AST 语法树，如果你对于这部分内容比较陌生，那么你可以查看我推荐给你的这个文章。
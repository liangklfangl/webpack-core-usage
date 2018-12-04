#### 1.本章概述
在本章节，我们通过一个引入 ladash 特定模块的实例来展示了 tree-shaking 在 Webpack 中的重要作用。通过合理的使用 tree-shaking 功能可以有效的减少打包后文件的大小，通过本实例我们也可以知道 tree-shaking 的作用条件和范围。这样对于 webpack 优化策略你又掌握了一部分核心知识。

#### 2.我们是否需要引入tree-shaking?
下面是ladash中对外导出的对象:
```js
   lodash.isFunction = isFunction;
    lodash.isInteger = isInteger;
    lodash.isLength = isLength;
    lodash.isMap = isMap;
    lodash.isMatch = isMatch;
    lodash.isMatchWith = isMatchWith;
    lodash.isNaN = isNaN;
    lodash.isNative = isNative;
    lodash.isNil = isNil;
    lodash.isNull = isNull;
    lodash.isNumber = isNumber;
    lodash.isObject = isObject;
    lodash.isObjectLike = isObjectLike;
    lodash.isPlainObject = isPlainObject;
    lodash.isRegExp = isRegExp;
    lodash.isSafeInteger = isSafeInteger;
    lodash.isSet = isSet;
    lodash.isString = isString;
    lodash.isSymbol = isSymbol;
    lodash.isTypedArray = isTypedArray;
    lodash.isUndefined = isUndefined;
    lodash.isWeakMap = isWeakMap;
    lodash.isWeakSet = isWeakSet;
```
这是为什么我们可以通过如下方式引入方法的原因:
```js
import { concat, sortBy, map, sample } from 'lodash';
//lodash其实是一个对象
```
但是还有一种常见的方法就是只引入我们需要的函数，如下:
```js
import sortBy from 'lodash/sortBy';
import map from 'lodash/map';
import sample from 'lodash/sample';
```
之所以可以通过这种方法引用是因为在lodash的npm包中，我们每一个方法都对应于一个独立的文件，并导出了该方法，如下面就是sortBy.js方法的源码:
```js
var sortBy = baseRest(function(collection, iteratees) {
  if (collection == null) {
    return [];
  }
  var length = iteratees.length;
  if (length > 1 && isIterateeCall(collection, iteratees[0], iteratees[1])) {
    iteratees = [];
  } else if (length > 2 && isIterateeCall(iteratees[0], iteratees[1], iteratees[2])) {
    iteratees = [iteratees[0]];
  }
  return baseOrderBy(collection, baseFlatten(iteratees, 1), []);
});
module.exports = sortBy;
```
注意一点就是:通过后者来导入我们需要的文件比前者全部导入的文件要小的多。上面我已经说了原因，即后者将每一个方法都存放在一个独立的文件中，从而可以按需导入，所以文件也就比较小了。具体你可以[查看这里](https://lacke.mn/reduce-your-bundle-js-file-size/)来学习如何减少bundle.js的大小。当然，如果你使用了webpack3的tree-shaking，那么就不需要考虑这个情况了。tree-shaking会让没用的代码在打包的时候直接被剔除。但是，请注意，tree-shaking 的功能要生效必须满足一定的条件，即必须是 ES6 模块。


#### 3.webpack引入tree-shaking功能
##### 3.1 webpack如何使用tree-shaking
为了让 webpack2 支持tree-shaking功能，我们需要对[wcf的 babel 配置进行修改](https://github.com/liangklfangl/wcf/blob/master/src/getBabelDefaultConfig.js)，其中修改最重要的一点就是去掉babel-preset-es2015 ，而采用 plugin 处理。在 plugin 处理的时候还需要去掉下面的插件:
```js
require.resolve("babel-plugin-transform-es2015-modules-amd"),
//转化为amd格式，define类型
require.resolve("babel-plugin-transform-es2015-modules-commonjs"),
//转化为commonjs规范，得到:exports.default = 42,export.name="罄天"
require.resolve("babel-plugin-transform-es2015-modules-umd"),
//umd规范
```
采用 babel-plugin-transform-es2015-modules-commonjs 以后，我们的代码:
```js
//imported.js
export function foo() {
    return 'foo';
}
export function bar() {
    return 'bar';
}
//下面是index.js
import {foo} from './imported';
let elem = document.getElementById('output');
elem.innerHTML = `Output: ${foo()}`;
```
会被webpack转化为如下的形式:
```js
Object.defineProperty(exports, "__esModule", {
    value: true
});
exports.foo = foo;
exports.bar = bar;
//都转化为commonjs规范了
function foo() {
    return 'foo';
}
function bar() {
    return 'bar';
}
/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

"use strict";
var _imported = __webpack_require__(0);

var elem = document.getElementById('output');
elem.innerHTML = 'Output: ' + (0, _imported.foo)();
/***/ })
/******/ ]);
```
所以，我们没有用到的bar方法也被引入了。而如果引入babel-plugin-transform-es2015-modules-amd，我们的打包代码就会得到如下的内容:
```js
/******/ ([
/* 0 */
/***/ (function(module, exports, __webpack_require__) {
var __WEBPACK_AMD_DEFINE_ARRAY__, __WEBPACK_AMD_DEFINE_RESULT__;!(__WEBPACK_AMD_DEFINE_ARRAY__ = [exports], __WEBPACK_AMD_DEFINE_RESULT__ = function (exports) {
    'use strict';
    Object.defineProperty(exports, "__esModule", {
        value: true
    });
    exports.foo = foo;
    exports.bar = bar;
    //我们的没有用到的bar方法也被导出了
    function foo() {
        return 'foo';
    }
    function bar() {
        return 'bar';
    }
}.apply(exports, __WEBPACK_AMD_DEFINE_ARRAY__),
        __WEBPACK_AMD_DEFINE_RESULT__ !== undefined && (module.exports = __WEBPACK_AMD_DEFINE_RESULT__));
/***/ }),
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

var __WEBPACK_AMD_DEFINE_ARRAY__, __WEBPACK_AMD_DEFINE_RESULT__;!(__WEBPACK_AMD_DEFINE_ARRAY__ = [__webpack_require__(0)], __WEBPACK_AMD_DEFINE_RESULT__ = function (_imported) {
  'use strict';

  var elem = document.getElementById('output');
  elem.innerHTML = 'Output: ' + (0, _imported.foo)();
}.apply(exports, __WEBPACK_AMD_DEFINE_ARRAY__),
        __WEBPACK_AMD_DEFINE_RESULT__ !== undefined && (module.exports = __WEBPACK_AMD_DEFINE_RESULT__));
/***/ })
/******/ ]);
```
而如果引入 babel-plugin-transform-es2015-modules-umd 也会面临同样的问题，所以我们应该去掉上面三个插件，即不再使用 `amd/cmd/umd` 规范打包，而使用我们的 ES6 原生模块打包策略。让 ES6 模块不受 Babel 预设（preset）的影响。Webpack 认识 ES6 模块，只有当保留 ES6 模块语法时才能够应用 tree-shaking。如果将其转换为 CommonJS 语法，Webpack 不知道哪些代码是使用过的，哪些不是（就不能应用 tree-shaking 了）。最后，Webpack将把它们转换为 CommonJS语法。最终得到的 babel 默认配置就是如下的内容:
```js
function getDefaultBabelConfig() {
  return {
    cacheDirectory: tmpdir(),
    //We must set!
    presets: [
      require.resolve('babel-preset-react'),
      // require.resolve('babel-preset-es2015'),
      //(1)这个必须去掉
      require.resolve('babel-preset-stage-0'),
    ],
    plugins: [
      require.resolve("babel-plugin-transform-es2015-template-literals"),
      require.resolve("babel-plugin-transform-es2015-literals"),
      require.resolve("babel-plugin-transform-es2015-function-name"),
      require.resolve("babel-plugin-transform-es2015-arrow-functions"),
      require.resolve("babel-plugin-transform-es2015-block-scoped-functions"),
      require.resolve("babel-plugin-transform-es2015-classes"),
      //这里会转化class
      require.resolve("babel-plugin-transform-es2015-object-super"),
      require.resolve("babel-plugin-transform-es2015-shorthand-properties"),
      require.resolve("babel-plugin-transform-es2015-computed-properties"),
      require.resolve("babel-plugin-transform-es2015-for-of"),
      require.resolve("babel-plugin-transform-es2015-sticky-regex"),
      require.resolve("babel-plugin-transform-es2015-unicode-regex"),
      require.resolve("babel-plugin-syntax-object-rest-spread"),
      require.resolve("babel-plugin-transform-es2015-parameters"),
      require.resolve("babel-plugin-transform-es2015-destructuring"),
      require.resolve("babel-plugin-transform-es2015-block-scoping"),
      require.resolve("babel-plugin-transform-es2015-typeof-symbol"),
      [
        require.resolve("babel-plugin-transform-regenerator"),
        { async: false, asyncGenerators: false }
      ],
      // require.resolve("babel-plugin-add-module-exports"),
      // 交给webpack2处理，可以删除
      require.resolve("babel-plugin-check-es2015-constants"),
      require.resolve("babel-plugin-syntax-async-functions"),
      require.resolve("babel-plugin-syntax-async-generators"),
      require.resolve("babel-plugin-syntax-class-constructor-call"),
      require.resolve("babel-plugin-syntax-class-properties"),
      require.resolve("babel-plugin-syntax-decorators"),
      require.resolve("babel-plugin-syntax-do-expressions"),
      require.resolve("babel-plugin-syntax-dynamic-import"),
      require.resolve("babel-plugin-syntax-exponentiation-operator"),
      require.resolve("babel-plugin-syntax-export-extensions"),
      require.resolve("babel-plugin-syntax-flow"),
      require.resolve("babel-plugin-syntax-function-bind"),
      require.resolve("babel-plugin-syntax-jsx"),
      require.resolve("babel-plugin-syntax-trailing-function-commas"),
      require.resolve("babel-plugin-transform-async-generator-functions"),
      require.resolve("babel-plugin-transform-async-to-generator"),
      require.resolve("babel-plugin-transform-class-constructor-call"),
      require.resolve("babel-plugin-transform-class-properties"),
      require.resolve("babel-plugin-transform-decorators"),
      require.resolve("babel-plugin-transform-decorators-legacy"),
      require.resolve("babel-plugin-transform-do-expressions"),
      require.resolve("babel-plugin-transform-es2015-duplicate-keys"),
      require.resolve("babel-plugin-transform-es2015-spread"),
      require.resolve("babel-plugin-transform-exponentiation-operator"),
      require.resolve("babel-plugin-transform-export-extensions"),
      // require.resolve("babel-plugin-transform-es2015-modules-amd"),
      // require.resolve("babel-plugin-transform-es2015-modules-commonjs"),
      // require.resolve("babel-plugin-transform-es2015-modules-umd"),
      // (2)去掉这个
      require.resolve("babel-plugin-transform-flow-strip-types"),
      require.resolve("babel-plugin-transform-function-bind"),
      require.resolve("babel-plugin-transform-object-assign"),
      require.resolve("babel-plugin-transform-object-rest-spread"),
      require.resolve("babel-plugin-transform-proto-to-assign"),
      require.resolve("babel-plugin-transform-react-display-name"),
      require.resolve("babel-plugin-transform-react-jsx"),
      require.resolve("babel-plugin-transform-react-jsx-source"),
      require.resolve("babel-plugin-transform-runtime"),
      require.resolve("babel-plugin-transform-strict-mode"),
    ]
  };
}
```
具体文件内容你可以点击[wcf](https://github.com/liangklfangl/wcf/blob/master/src/getBabelDefaultConfig.js)打包 babel 配置。当然你也可以使用下面方式告诉 babel 预设不转换模块：
```js
{
  "presets": [
    ["env", {
      "loose": true,
      "modules": false
    }]
  ]
}
```
这种方式要简单的多。但是，正如[如何在 Webpack 2 中使用 tree-shaking](https://mp.weixin.qq.com/s?__biz=MjM5MTA1MjAxMQ==&mid=2651226843&idx=1&sn=8ce859bb0ccaa2351c5f8231cc016052&chksm=bd495b5f8a3ed249bb2d967e27f5e0ac20b42f42698fdfd0d671012782ce0074a21129e5f224&mpshare=1&scene=1&srcid=08241S5UYwTpLwk1N2s51tXG&key=adf9313632dd72f547280f783810492f9adb79ab0d4163835d8f16b9ef1ba0b666c3253ebf73fcbd10842f39091c3775a8bcb7ebf2f1613b0baadc517bd3a3f871c02aa3495fa42b3e960fd7f99357e0&ascene=0&uin=MTkwNTY4NjMxOQ%3D%3D&devicetype=iMac+MacBookAir7%2C2+OSX+OSX+10.12+build(16A323)&version=12020810&nettype=WIFI&fontScale=100&pass_ticket=Kwkar2P9YwiWaPYmrcaqYmEqAigrP8I305SDCp6p05cCbna5znl6Uz%2FMx75BskRL)文章本身所说，这种方式会存在副作用，即无法移除多余的类声明。在使用 ES6 语法定义类时，类的成员函数会被添加到属性 prototype ，没有什么方法能完全避免这次赋值，所以 webpack 会`认为我们添加到prototype 上方法的操作也是对类的一种使用，导致无法移除多余的类声明`,编译过程阻止了对类进行 tree-shaking ,它仅对函数起作用。UglifyJS 不能够分辨它仅仅是类声明，还是其它有副作用的操作，因为UglifyJS 不能做控制流分析。

##### 3.2 webpack的tree-shaking标记 vs rollup标记区别
<pre>
移除未使用代码（Dead code elimination） vs 包含已使用代码（live code inclusion）
</pre>
Webpack 仅仅标记未使用的代码而不移除，并且不将其导出到模块外。它拉取所有用到的代码，将剩余的（未使用的）代码留给像 UglifyJS 这类压缩代码的工具来移除。UglifyJS 读取打包结果，在压缩之前移除未使用的代码。而 Rollup 不同，它的打包结果只包含运行应用程序所必需的代码。打包完成后的输出并没有未使用的类和函数，压缩仅涉及实际使用的代码。

##### 3.3 基于babel-minify-webpack-plugin(即babili-webpack-plugin)移除多余的类声明
[babel-minify-webpack-plugin](https://github.com/webpack-contrib/babel-minify-webpack-plugin)能将 ES6 代码编译为 ES5，移除未使用的类和函数，这就像 UglifyJS 已经支持 ES6 一样。babel-minify 会在编译前`删除未使用的代码`。在编译为 ES5 之前，很容易找到未使用的类，因此tree-shaking 也可以用于类声明，而不再仅仅是函数。如果你去看 babili-webpack-plugin 的代码，你会看到下面两句:
```js
import { transform } from 'babel-core';
import babelPresetMinify from 'babel-preset-minify';
```
首先是就是[babel-preset-minify](https://github.com/babel/minify/blob/master/packages/babel-preset-minify/src/index.js),你可以看到他内部会调用如 babel-plugin-minify-dead-code-elimination , babel-plugin-minify-type-constructors 等来判断哪些代码没有被引用，进而可以在代码没有被编译为 ES5 之前把它移除掉。而 babel-core 就是负责把处理后的 ES6 代码继续编译为 ES5 代码。

所以，我们只需用 babel-minify-webpack-plugin 替换 UglifyJS ，然后删除 babel-loader (该 plugin 自己会处理 ES6 代码，但是 jsx 处理需要自己添加 preset ) 即可。另一种方式是将[babel-preset-minify](https://github.com/babel/minify/tree/master/packages/babel-preset-minify)作为 Babel 的预设，仅使用 babel-loader（移除 UglifyJS 插件,因为 babel-preset-minify 已经压缩完成）。推荐使用第一种（插件的方式），因为当编译器不是 Babel（比如 Typescript）时，它也能生效。
```js
module: {
  rules: []
},
plugins: [
  new BabiliPlugin()
  //替代uglifyjs，它可以移除es6的多余类声明
]
```
我们需要将 ES6+ 代码传给 babel-minify ，否则它不会移除（未使用的）类。所以，这种方式就要求所有的第三方包都必须有es6的代码发布，否则无法移除。

##### 3.4 目前[wcf](https://github.com/liangklfangl/wcf)没有引入babili-webpack-plugin
这种情况下我们依然会对类的代码打包成为ES5，然后交给我们的uglifyjs处理,比如下面的例子：
```js
//imported.js
export function foo() {
    return 'foo';
}
export function bar() {
    return 'bar';
}
export function ql(){
  return 'ql'
}
export class Test{
 toString(){
   return 'test';
 }
}
export class Test1{
 toString(){
   return 'test1';
 }
}
//index.js
import {foo} from './imported';
let elem = document.getElementById('app');
elem.innerHTML = `Output: ${foo()}`;
```
打包后的结果如下:
```js
/***/ (function(module, __webpack_exports__, __webpack_require__) {
"use strict";
/* harmony export (immutable) */ 
__webpack_exports__["a"] = foo;
/* unused harmony export bar */
/* unused harmony export ql */
/* unused harmony export Test */
/* unused harmony export Test1 */
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0_babel_runtime_helpers_classCallCheck__ = __webpack_require__(8);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0_babel_runtime_helpers_classCallCheck___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_0_babel_runtime_helpers_classCallCheck__);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1_babel_runtime_helpers_createClass__ = __webpack_require__(9);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1_babel_runtime_helpers_createClass___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_1_babel_runtime_helpers_createClass__);
function foo() {
  return 'foo';
}
function bar() {
  return 'bar';
}
function ql() {
  return 'ql';
}
var Test = function () {
  function Test() {
    __WEBPACK_IMPORTED_MODULE_0_babel_runtime_helpers_classCallCheck___default()(this, Test);
  }
  __WEBPACK_IMPORTED_MODULE_1_babel_runtime_helpers_createClass___default()(Test, [{
    key: 'toString',
    value: function toString() {
      return 'test';
    }
  }]);

  return Test;
}();
var Test1 = function () {
  function Test1() {
    __WEBPACK_IMPORTED_MODULE_0_babel_runtime_helpers_classCallCheck___default()(this, Test1);
  }

  __WEBPACK_IMPORTED_MODULE_1_babel_runtime_helpers_createClass___default()(Test1, [{
    key: 'toString',
    value: function toString() {
      return 'test1';
    }
  }]);
  return Test1;
}();
})
```
此时通过查看 `harmony export` 部分，我们知道 webpack 导出的仅仅是我们用到的 foo 模块而已，而其他的不管是多余的函数声明还是多余的类声明都是被标记为无用代码(`'unused'`)。我以为，通过这种方式打包，经过uglifyjs处理就会将类 Test1,Test2 的代码移除，其实事实并不是这样，经过 uglifyjs 处理后多余的函数是没有了，但是多余的类声明打包成的函数代码依然存在!依然存在!依然存在!

终极解决方法:[使用babel-minify-webpack-plugin](https://github.com/webpack-contrib/babel-minify-webpack-plugin),即babili-webpack-plugin。完整实例代码可以[参考这里](https://github.com/blacksonic/babel-webpack-tree-shaking),而目前[wcf](https://github.com/liangklfangl/wcf)没有采用这种策略，所以多余的 class 是无法去除的。目前，我觉得这种策略是可以接受的，因为我们第三方发布的包很少是使用 class 发布的，而都是编译为 ES5 代码后发布的，所以通过 uglifyjs 这种策略已经足够了。

当然，你也可以使用[babel-preset-minify](https://github.com/babel/minify/tree/master/packages/babel-preset-minify)来将代码压缩作为你的预设，我觉的这种方式在独立封装自己的打包工具的时候比较有用，他是所有 babel 代码压缩插件的集合。

### 4.tree-shaking的局限性
这一部分都是自己的理解，但是是基于这样一个事实：
```js
import {sortBy} from "lodash";
```
我通过 import 引入 sortBy 方法以后，以为仅仅是引入了该方法而已，但是实际上把 concat 等函数都引入了。因为 import 是基于 ES6 的静态语法分析，而我们的 lodash 第三方包导出的时候并不是基于 ES6 的 import/export 机制，代码如下：
```js
 var _ = runInContext();
  if (typeof define == 'function' && typeof define.amd == 'object' && define.amd) {
    root._ = _;
    define(function() {
      return _;
    });
  }
  else if (freeModule) {
    // Export for Node.js.
    (freeModule.exports = _)._ = _;
    // Export for CommonJS support.
    freeExports._ = _;
  }
  else {
    // Export to the global object.
    root._ = _;
  }
}.call(this));
```
所以，实际上 tree-shaking 是无法完成的!所以，上面说的 babel-minify-webpack-plugin 其实在这里根本不能起作用，因为他的第一步就是去掉 ES6 中没有用到的代码然后打包成为 ES5 ，但是这里的代码压根就不是 ES6，所以它也就没有魔力了。对第三方包来说也是，应当使用 ES6 模块。幸运的是，越来越多的包作者同时发布 CommonJS 格式和ES6格式的模块。ES6 模块的入口由 package.json的字段 module 指定。

对 ES6 模块，未使用的函数会被移除，但 class 并不一定会。只有当包内的 class 定义也为 ES6 格式时，babili-webpack-plugin才能将其移除。很少有包能够以这种格式发布，但有的做到了（比如说 lodash 的 lodash-es）(本段文字来自于[如何在 Webpack 2 中使用 tree-shaking](https://mp.weixin.qq.com/s?__biz=MjM5MTA1MjAxMQ==&mid=2651226843&idx=1&sn=8ce859bb0ccaa2351c5f8231cc016052&chksm=bd495b5f8a3ed249bb2d967e27f5e0ac20b42f42698fdfd0d671012782ce0074a21129e5f224&mpshare=1&scene=1&srcid=08241S5UYwTpLwk1N2s51tXG&key=adf9313632dd72f547280f783810492f9adb79ab0d4163835d8f16b9ef1ba0b666c3253ebf73fcbd10842f39091c3775a8bcb7ebf2f1613b0baadc517bd3a3f871c02aa3495fa42b3e960fd7f99357e0&ascene=0&uin=MTkwNTY4NjMxOQ%3D%3D&devicetype=iMac+MacBookAir7%2C2+OSX+OSX+10.12+build(16A323)&version=12020810&nettype=WIFI&fontScale=100&pass_ticket=Kwkar2P9YwiWaPYmrcaqYmEqAigrP8I305SDCp6p05cCbna5znl6Uz%2FMx75BskRL)),你可以可以参考[如何评价 Webpack 2 新引入的 Tree-shaking 代码优化技术？](https://www.zhihu.com/question/41922432)尤雨溪的回答:
<p>
ES6的模块设计虽然使得灵活性不如 CommonJS 的 require ，但却保证了 ES6 modules 的依赖关系是确定 (deterministic) 的，和运行时的状态无关，从而也就保证了 ES6 modules 是可以进行可靠的静态分析的。对于主要在服务端运行的 Node 来说，所有的代码都在本地，按需动态 require 即可，但对于要下发到客户端的 web 代码而言，要做到高效的按需使用，不能等到代码执行了才知道模块的依赖，必须要从模块的静态分析入手。这是 ES6 modules 在设计时的一个重要考量，也是为什么没有直接采用 CommonJS。
</p>

所以，我们在引入一个lodash模块的时候应该使用下面的模式:
```js
import sortBy from 'lodash/sortBy';
```

### 5.本章小结
在本章节，我们通过 lodash 的一个简单实例说明了 webpack 引入 tree-shaking 的重要意义。同时，我们也通过例子说明了如何在 webpack2 中引入 tree-shaking，以及要实现 tree-shaking需要满足的条件。通过本章节的学习，你不仅能够了解如何实现 tree-shaking，也将知道如何优化自己的代码，以及如何为未来 npm 包的优化做出选择。
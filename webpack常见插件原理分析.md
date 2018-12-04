#### 1.本章说明
主要讲解一下 Webpack 几个稍微简单的一点的插件原理，通过本章节的学习，你对前面的知识应该会有一个更加深入的理解。

#### 2.prepack-webpack-plugin的说明
今年facebook开源了一个[prepack](https://github.com/facebook/prepack)，当时就很好奇。它到底和webpack之间的关系是什么？于是各种google,最后还是去官网上看了下各种例子。例子都很好理解，但是对于其和webpack的关系还是有点迷糊。最后找到了一个好用的插件，即[prepack-webpack-plugin](https://github.com/gajus/prepack-webpack-plugin),这才恍然大悟~

#### 2.1 解析prepack-webpack-plugin源码
下面我们直接给出这个插件的apply源码，因为webpack的plugin的所有逻辑都是在apply方法中处理的。内容如下:
```js
import ModuleFilenameHelpers from 'webpack/lib/ModuleFilenameHelpers';
import {
  RawSource
} from 'webpack-sources';
import {
  prepack
} from 'prepack';
import type {
  PluginConfigurationType,
  UserPluginConfigurationType
} from './types';
const defaultConfiguration = {
  prepack: {},
  test: /\.js($|\?)/i
};
export default class PrepackPlugin {
  configuration: PluginConfigurationType;
  constructor (userConfiguration?: UserPluginConfigurationType) {
    this.configuration = {
      ...defaultConfiguration,
      ...userConfiguration
    };
  }
  apply (compiler: Object) {
    const configuration = this.configuration;
    compiler.plugin('compilation', (compilation) => {
      compilation.plugin('optimize-chunk-assets', (chunks, callback) => {
        for (const chunk of chunks) {
          const files = chunk.files;
          //chunk.files获取该chunk产生的所有的输出文件，记住是输出文件
          for (const file of files) {
            const matchObjectConfiguration = {
              test: configuration.test
            };
            if (!ModuleFilenameHelpers.matchObject(matchObjectConfiguration, file)) {
              // eslint-disable-next-line no-continue
              continue;
            }
            const asset = compilation.assets[file];
            //获取文件本身
            const code = asset.source();
            //获取文件的代码内容
            const prepackedCode = prepack(code, {
              ...configuration.prepack,
              filename: file
            });
            //所以，这里是在webpack打包后对ES5代码的处理
            compilation.assets[file] = new RawSource(prepackedCode.code);
          }
        }
        callback();
      });
    });
  }
}
```
首先对于webpack各种钩子函数时机不了解的可以[点击这里](https://github.com/liangklfangl/webpack-compiler-and-compilation)。如果对于webpack中各个对象的属性不了解的可以[点击这里](https://github.com/liangklfangl/webpack-common-sense)。接下来我们对上面的代码进行简单的剖析:

(1)首先看for循环的前面那几句
```js
  const files = chunk.files;
  //chunk.files获取该chunk产生的所有的输出文件，记住是输出文件
  for (const file of files) {
   //这里我们只会对该chunk包含的文件中符合test规则的文件进行后续处理
    const matchObjectConfiguration = {
      test: configuration.test
    };
    if (!ModuleFilenameHelpers.matchObject(matchObjectConfiguration, file)) {
      // eslint-disable-next-line no-continue
      continue;
    }
}
```
我们这里也给出ModuleFilenameHelpers.matchObject的代码:
```js
//将字符串转化为regex
function asRegExp(test) {
    if(typeof test === "string") test = new RegExp("^" + test.replace(/[-[\]{}()*+?.,\\^$|#\s]/g, "\\$&"));
    return test;
}
ModuleFilenameHelpers.matchPart = function matchPart(str, test) {
    if(!test) return true;
    test = asRegExp(test);
    if(Array.isArray(test)) {
        return test.map(asRegExp).filter(function(regExp) {
            return regExp.test(str);
        }).length > 0;
    } else {
        return test.test(str);
    }
};
ModuleFilenameHelpers.matchObject = function matchObject(obj, str) {
    if(obj.test)
        if(!ModuleFilenameHelpers.matchPart(str, obj.test)) 
        return false;
    //获取test，如果这个文件名称符合test规则返回true，否则为false
    if(obj.include)
        if(!ModuleFilenameHelpers.matchPart(str, obj.include)) return false;
    if(obj.exclude)
        if(ModuleFilenameHelpers.matchPart(str, obj.exclude)) return false;
     return true;
};
```
这几句代码是一目了然的，如果这个产生的文件名称符合test规则返回true，否则为false。

(2)我们继续看后面对于符合规则的文件的处理
```js
 //如果满足规则我们继续处理~
 const asset = compilation.assets[file];
//获取编译产生的资源
const code = asset.source();
//获取文件的代码内容
const prepackedCode = prepack(code, {
  ...configuration.prepack,
  filename: file
});
//所以，这里是在webpack打包后对ES5代码的处理
compilation.assets[file] = new RawSource(prepackedCode.code);
```
其中asset.source表示的是模块的内容,你可以[点击这里](https://github.com/liangklfangl/webpack-common-sense)查看。假如我们的模块是一个html,内容如下:
```html
<header class="header">{{text}}</header>
```
最后打包的结果为:
```js
module.exports = "<header class=\\"header\\">{{text}}</header>";' }
```
这也是为什么我们会有下面的代码:
```js
compilation.assets[basename] = {
      source: function () {
        return results.source;
      },
      //source是文件的内容，通过fs.readFileAsync完成
      size: function () {
        return results.size.size;
        //size通过 fs.statAsync(filename)完成
      }
    };
    return basename;
  });
```
前面两句代码我们都分析过了，我们继续看下面的内容:
```js
const prepackedCode = prepack(code, {
  ...configuration.prepack,
  filename: file
});
//所以，这里是在webpack打包后对ES5代码的处理
compilation.assets[file] = new RawSource(prepackedCode.code);
```
此时才真正的对webpack打包后的代码进行处理，prepack的nodejs用法可以[查看这里](https://prepack.io/getting-started.html)。最后一句代码其实就是操作我们的输出资源，在输出资源中添加一个文件，文件的内容就是prepack打包后的代码。其中webpack-source的内容你可以[点击这里](https://github.com/webpack/webpack-sources)。按照官方的说明，该对象可以获取源代码，hash,内容大小，sourceMap等所有信息。我们给出对RowSourceMap的说明:
<pre>
RawSource
Represents source code without SourceMap.
new RawSource(sourceCode: String)
</pre>

很显然，就是显示源代码而不包含sourceMap。

#### 2.2 prepack-webpack-plugin总结
所以，从我的理解来说，prepack作用于webpack的时机在于：将源代码转化为ES5以后。从上面的html的编译结果就可以知道了，至于它到底做了什么，以及如何做的，还请查看[官网](https://prepack.io/getting-started.html)


### 3.BannerPlugin插件分析
我们现在讲述一下BannerPlugin内部的原理。他的主要用法如下：
```js
{
  banner: string, 
	// the banner as string, it will be wrapped in a comment
  raw: boolean, 
	//如果配置了raw，那么banner会被包裹到注释当中
  entryOnly: boolean, 
	//如果设置为true，那么banner仅仅会被添加到入口文件产生的chunk中
  test: string | RegExp | Array,
  include: string | RegExp | Array,
  exclude: string | RegExp | Array,
}
```
我们看看他的内部代码:
```js
"use strict";
const ConcatSource = require("webpack-sources").ConcatSource;
const ModuleFilenameHelpers = require("./ModuleFilenameHelpers");
//'This file is created by liangklfangl' =>/*! This file is created by liangklfangl */
function wrapComment(str) {
	if(!str.includes("\n")) return `/*! ${str} */`;
	return `/*!\n * ${str.split("\n").join("\n * ")}\n */`;
}
class BannerPlugin {
	constructor(options) {
		if(arguments.length > 1)
			throw new Error("BannerPlugin only takes one argument (pass an options object)");
		if(typeof options === "string")
			options = {
				banner: options
			};
		this.options = options || {};
		//配置参数
		this.banner = this.options.raw ? options.banner : wrapComment(options.banner);
	}
	apply(compiler) {
		let options = this.options;
		let banner = this.banner;
		compiler.plugin("compilation", (compilation) => {
			compilation.plugin("optimize-chunk-assets", (chunks, callback) => {
				chunks.forEach((chunk) => {
					//入口文件都是默认首次加载的,即isInitial为true，和require.ensure按需加载是完全不一样的
					if(options.entryOnly && !chunk.isInitial()) return;
					chunk.files
						.filter(ModuleFilenameHelpers.matchObject.bind(undefined, options))
						//只要满足test正则表达式的文件才会被处理
						.forEach((file) =>
							compilation.assets[file] = new ConcatSource(
								banner, "\n", compilation.assets[file]
								//在原来的输出文件头部添加我们的banner信息
							)
						);
				});
				callback();
			});
		});
	}
}
module.exports = BannerPlugin;
```

### 4.EnvironmentPlugin插件分析
该插件的使用方法如下：
```js
new webpack.EnvironmentPlugin(['NODE_ENV', 'DEBUG'])
```
此时相当于以以下方式使用DefinePlugin插件
```js
new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
  'process.env.DEBUG': JSON.stringify(process.env.DEBUG)
})
```
当然，该插件也可以传入一个对象:
```js
new webpack.EnvironmentPlugin({
  NODE_ENV: 'development', 
	// use 'development' unless process.env.NODE_ENV is defined
  DEBUG: false
})
```
假如我们有如下的entry文件:
```js
if (process.env.NODE_ENV === 'production') {
  console.log('Welcome to production');
}
if (process.env.DEBUG) {
  console.log('Debugging output');
}
```
如果我们执行*NODE_ENV=production webpack*命令，那么你会发现输出文件为如下内容:
```js
if ('production' === 'production') { // <-- 'production' from NODE_ENV is taken
  console.log('Welcome to production');
}
if (false) { // <-- default value is taken
  console.log('Debugging output');
}
```
那么上面讲述了这个插件如何使用，那么我们看看他的内部原理是什么?
```js
"use strict";
const DefinePlugin = require("./DefinePlugin");
//1.EnvironmentPlugin内部直接调用DefinePlugin
class EnvironmentPlugin {
	constructor(keys) {
		this.keys = Array.isArray(keys) ? keys : Object.keys(arguments);
	}
	apply(compiler) {
		//2.这里直接使用compiler.apply方法来执行我们的DefinePlugin插件
		compiler.apply(new DefinePlugin(this.keys.reduce((definitions, key) => {
			const value = process.env[key];
			//获取process.env中的参数
			if(value === undefined) {
				compiler.plugin("this-compilation", (compilation) => {
					const error = new Error(key + " environment variable is undefined.");
					error.name = "EnvVariableNotDefinedError";
					//3.我们可以往compilation.warning里面填充我们自己的编译warning信息
					compilation.warnings.push(error);
				});
			}
			definitions["process.env." + key] = value ? JSON.stringify(value) : "undefined";
			//4.将我们的所有的key都封装到process.env上面了并返回(注意这里是向process.env上赋值)
			return definitions;
		}, {})));
	}
}
module.exports = EnvironmentPlugin;
```

### 5.MinChunkSizePlugin插件分析
这个插件的作用在于，如果产生的某个 Chunk 的大小小于阈值，那么直接和其他的 Chunk 合并，其主要使用方法如下:
```js
new webpack.optimize.MinChunkSizePlugin({
  minChunkSize: 10000 
})
```
我们看看他的内部原理是如何实现的:
```js
class MinChunkSizePlugin {
	constructor(options) {
		if(typeof options !== "object" || Array.isArray(options)) {
			throw new Error("Argument should be an options object.\nFor more info on options, see https://webpack.github.io/docs/list-of-plugins.html");
		}
		this.options = options;
	}
	apply(compiler) {
		const options = this.options;
		const minChunkSize = options.minChunkSize;
		compiler.plugin("compilation", (compilation) => {
			compilation.plugin("optimize-chunks-advanced", (chunks) => {
				let combinations = [];
				chunks.forEach((a, idx) => {
					for(let i = 0; i < idx; i++) {
						const b = chunks[i];
						combinations.push([b, a]);
					}
				});
				const equalOptions = {
					chunkOverhead: 1,
					// an additional overhead for each chunk in bytes (default 10000, to reflect request delay)
					entryChunkMultiplicator: 1
					//a multiplicator for entry chunks (default 10, entry chunks are merged 10 times less likely)
					//入口文件乘以的权重，所以如果含有入口文件，那么更加不容易小于minChunkSize，所以入口文件过小不容易被集成到别的chunk中
				};
				combinations = combinations.filter((pair) => {
					return pair[0].size(equalOptions) < minChunkSize || pair[1].size(equalOptions) < minChunkSize;
				});
        //对数组中元素进行删选，至少有一个chunk的值是小于minChunkSize的
				combinations.forEach((pair) => {
					const a = pair[0].size(options);
					const b = pair[1].size(options);
					const ab = pair[0].integratedSize(pair[1], options);
					//得到第一个chunk集成了第二个chunk后的文件大小
					pair.unshift(a + b - ab, ab);
					//这里的pair是如[0,1],[0,2]等这样的数组元素，前面加上两个元素:集成后总体积的变化量;集成后的体积
				});
				//此时combinations的元素至少有一个的大小是小于minChunkSize的
				combinations = combinations.filter((pair) => {
					return pair[1] !== false;
				});
				if(combinations.length === 0) return;
				//如果没有需要优化的，直接返回
				combinations.sort((a, b) => {
					const diff = b[0] - a[0];
					if(diff !== 0) return diff;
					return a[1] - b[1];
				});
				//按照我们的集成后变化的体积来比较，从大到小排序
				const pair = combinations[0];
				//得到第一个元素
				pair[2].integrate(pair[3], "min-size");
				//pair[2]是我们的chunk,pair[3]也是chunk
				chunks.splice(chunks.indexOf(pair[3]), 1);
				//从chunks集合中删除集成后的chunk
				return true;
			});
		});
	}
}
module.exports = MinChunkSizePlugin;
```
下面说明一下具体的其中主要的代码:
```js
var combinations = [];
var chunks=[0,1,2,3]
chunks.forEach((a, idx) => {
	for(let i = 0; i < idx; i++) {
		const b = chunks[i];
		combinations.push([b, a]);
	}
});
```
变量combinations是组合形式，把自己和前面比自己小的元素组合成为一个元素。之所以是选择比自己的小的情况是为了减少重复的个数，如[0,2]和[2,0]必须只有一个。

#### 6.本章小结
在本章节，我们主要讲了几个稍微简单一点的 Webpack 的 Plugin ， 如果你对于 Plugin 的原理比较感兴趣，我想前面我介绍的那些基础知识已经够用了。至于很多复杂的 Plugin 就需要在平时开发的时候多关注和学习了。更多 Webpack 插件的分析你也可以[点击这里](https://github.com/liangklfang/webpack/tree/master/lib/optimize)， 而至于插件本身的用法，我觉得[官网](https://webpack.js.org/plugins/min-chunk-size-plugin/)就已经足够了。

#### 1.本章概述
本章节我们将会详细论述webpack对于HMR的支持，当然因为更多的涉及原理的东西，所以代码也比较多。如果你有任何不懂的地方，记得在读者圈给我提问，我会及时回答。

#### 2.webpack 是如何实现 HMR 的以及实现的原理如何

##### 2.1 webpack的HMR的实现
其实webpack实现HMR是依赖于webpack-dev-server的，webpack官方文档也写的非常清楚，我们只需要参考它的做法来完成即可，首先假如我们有如下的webpack.config.js配置文件:
```js
  const path = require('path');
  const HtmlWebpackPlugin = require('html-webpack-plugin');
  const CleanWebpackPlugin = require('clean-webpack-plugin');
const webpack = require('webpack');
  module.exports = {
    entry: {
      app: './src/index.js'
    },
    devtool: 'inline-source-map',
    devServer: {
      contentBase: './dist',
      hot: true
      //支持HMR
    },
    plugins: [
     new webpack.NamedModulesPlugin(),
     new webpack.HotModuleReplacementPlugin()
     //该插件来自于webpack本身的支持
    ],
    output: {
      filename: '[name].bundle.js',
      path: path.resolve(__dirname, 'dist')
    }
  };
```
很显然，我们的入口文件中只有app.js,同时在webpack的devServer配置中我们设置了hot为true,而且在webpack的plugin中我们添加了new webpack.HotModuleReplacementPlugin()这个插件。
假如我们需要在index.js中实现HMR，我们写一个index.js的代码实例:
```js
     import printMe from './print.js';
     if(module.hot){
        //此时accept方法第一个参数表示只有当print.js发生改变我们才会热加载
        module.hot.accept('./print.js', function() {
        console.log('Accepting the updated printMe module!');
        printMe();
    })
}
```
而我们的print.js为如下内容:
```js
  export default function printMe() {
      console.log('I get called from print.js!');
     }
```
此时，当你修改print.js的时候，我们的index.js也会重新加载，而且在控制台中也会输出如下内容:
<pre>
 [HMR] Waiting for update signal from WDS... main.js:4395 [WDS] Hot
 Module Replacement enabled.
 + 2main.js:4395 [WDS] App updated. Recompiling...
 + main.js:4395 [WDS] App hot update...
 + main.js:4330 [HMR] Checking for updates on the server...
 + main.js:10024 Accepting the updated printMe module!
 + 0.4b8ee77….hot-update.js:10 Updating print.js...
 + main.js:4330 [HMR] Updated modules:
 + main.js:4330 [HMR]  - 20
 + main.js:4330 [HMR] Consider using the NamedModulesPlugin for module names.
 </pre>

这就是HMR热加载的方式，而且它不是采用完全reload页面的方式，所以在开发中是一个很重要的特性。当然，只有模块本身支持热加载并开启了webpack的热加载特性以后才会支持，这一点一定要注意。

##### 2.2 webpack的HMR的实现原理
看看下面的方法你就知道了，在hot模式下，我们的entry最后都会被添加两个文件，而这两个文件和你配置的entry一起作为webpack最终的入口文件，进而构建最终的模块依赖图谱。
```js
module.exports = function addDevServerEntrypoints(webpackOptions, devServerOptions) {
    if(devServerOptions.inline !== false) {
        //表示是inline模式而不是iframe模式
        const domain = createDomain(devServerOptions);
        const devClient = [`${require.resolve("../../client/")}?${domain}`];
        //客户端内容
        if(devServerOptions.hotOnly)
            devClient.push("webpack/hot/only-dev-server");
        else if(devServerOptions.hot)
            devClient.push("webpack/hot/dev-server");
        //配置了不同的webpack而文件到客户端文件中
        [].concat(webpackOptions).forEach(function(wpOpt) {
            if(typeof wpOpt.entry === "object" && !Array.isArray(wpOpt.entry)) {
                //这里是我们自己在webpack.config.js中配置的entry对象，对entry中的每一个入口文件都添加我们webpack/hot/only-dev-server或者webpack/hot/dev-server用于实现HMR
                Object.keys(wpOpt.entry).forEach(function(key) {
                    wpOpt.entry[key] = devClient.concat(wpOpt.entry[key]);
                });
            } else if(typeof wpOpt.entry === "function") {
                wpOpt.entry = wpOpt.entry(devClient);
                //如果entry是一个函数那么我们把devClient数组传入函数，由开发者自己构建自己的entry，但是只有在HMR开启的情况下适用
            } else {
                wpOpt.entry = devClient.concat(wpOpt.entry);
                //如果用户的entry是数组，那么我们直接将webpack/hot/only-dev-server或者webpack/hot/dev-server传入用于实现HMR
            }
        });
    }
};
```
请仔细理解上面的注释，因为它蕴含着在HMR模式下，webpack-dev-server对于我们自己配置的entry的一种进一步处理。下面我们将会进一步深入分析webpack/hot/only-dev-server和webpack/hot/dev-server，看看他们是如何实现HMR的。我们来看看"webpack/hot/only-dev-server"的文件内容，它是实现HMR的关键:
```js
if(module.hot) {
    var lastHash;
    var upToDate = function upToDate() {
        return lastHash.indexOf(__webpack_hash__) >= 0;
      //(1)如果两个hash相同那么表示没有更新，其中lastHash表示上一次编译的hash，记住是compilation的hash
      //只有在HotModuleReplacementPlugin开启的时候存在。任意文件变化后compilation都会发生变化
    };
    //(2)下面是检查更新的模块
    var check = function check() {
        module.hot.check().then(function(updatedModules) {
            //(2.1)没有更新的模块直接返回,通知用户无需HMR
            if(!updatedModules) {
                console.warn("[HMR] Cannot find update. Need to do a full reload!");
                console.warn("[HMR] (Probably because of restarting the webpack-dev-server)");
                return;
            }
            //(2.2)开始更新
            return module.hot.apply({
                ignoreUnaccepted: true,
                //和accept函数指定热加载那些模块
                ignoreDeclined: true,
                //decline表示不支持这个模块热加载
                ignoreErrored: true,
                //error表示出错的模块
                onUnaccepted: function(data) {
                    console.warn("Ignored an update to unaccepted module " + data.chain.join(" -> "));
                },
                onDeclined: function(data) {
                    console.warn("Ignored an update to declined module " + data.chain.join(" -> "));
                },
                onErrored: function(data) {
                    console.warn("Ignored an error while updating module " + data.moduleId + " (" + data.type + ")");
                }
             //(2.2.1)renewedModules表示哪些模块已经更新了
            }).then(function(renewedModules) {
                //(2.2.2)如果有模块没有更新完成，那么继续检查
                if(!upToDate()) {
                    check();
                }
                //(2.2.3)更新的模块updatedModules，renewedModules表示哪些模块已经更新了
                require("./log-apply-result")(updatedModules, renewedModules);
                //通知已经热加载完成
                if(upToDate()) {
                    console.log("[HMR] App is up to date.");
                }
            });
        }).catch(function(err) {
        //(2.3)更新异常，输出HMR信息
            var status = module.hot.status();
            if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot check for update. Need to do a full reload!");
                console.warn("[HMR] " + err.stack || err.message);
            } else {
                console.warn("[HMR] Update check failed: " + err.stack || err.message);
            }
        });
    };
    var hotEmitter = require("./emitter");
    //(3)emitter模块内容，也就是导出一个events实例
    /*
    var EventEmitter = require("events");
    module.exports = new EventEmitter();
     */
    hotEmitter.on("webpackHotUpdate", function(currentHash) {
        lastHash = currentHash;
        //(3.1)表示本次更新后得到的hash值
        if(!upToDate()) {
            //(3.1.1)有更新
            var status = module.hot.status();
            if(status === "idle") {
                console.log("[HMR] Checking for updates on the server...");
                check();
            } else if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot apply update as a previous update " + status + "ed. Need to do a full reload!");
            }
        }
    });
    console.log("[HMR] Waiting for update signal from WDS...");
} else {
    throw new Error("[HMR] Hot Module Replacement is disabled.");
}
```
上面看到了log-apply-result模块，我们看到这个模块是在所有的内容已经更新完成后调用的，下面继续看一下它到底做了什么事情:
```js
module.exports = function(updatedModules, renewedModules) {
    //(1)renewedModules表示哪些模块需要更新，剩余的模块unacceptedModules表示，哪些模块由于 ignoreDeclined，ignoreUnaccepted配置没有更新
    var unacceptedModules = updatedModules.filter(function(moduleId) {
        return renewedModules && renewedModules.indexOf(moduleId) < 0;
    });
    //(2)unacceptedModules表示该模块无法HMR，打印log
    if(unacceptedModules.length > 0) {
        console.warn("[HMR] The following modules couldn't be hot updated: (They would need a full reload!)");
        unacceptedModules.forEach(function(moduleId) {
            console.warn("[HMR]  - " + moduleId);
        });
    }
    //(2)没有模块更新，表示模块是最新的
    if(!renewedModules || renewedModules.length === 0) {
        console.log("[HMR] Nothing hot updated.");
    } else {
        console.log("[HMR] Updated modules:");
        //(3)打印那些模块被热更新。每一个moduleId都是数字那么建议使用NamedModulesPlugin(webpack2建议)
        renewedModules.forEach(function(moduleId) {
            console.log("[HMR]  - " + moduleId);
        });
        var numberIds = renewedModules.every(function(moduleId) {
            return typeof moduleId === "number";
        });
        if(numberIds)
            console.log("[HMR] Consider using the NamedModulesPlugin for module names.");
    }
};
```

所以"webpack/hot/only-dev-server"的文件内容就是检查哪些模块更新了(通过webpackHotUpdate事件完成，而该事件依赖于`compilation`的hash值)，其中哪些模块更新成功，而哪些模块由于某种原因没有更新成功。至于模块什么时候接受到需要更新是和webpack的打包过程有关的，这里也给出触发更新的时机:

```js
 ok: function() {
        sendMsg("Ok");
        if(useWarningOverlay || useErrorOverlay) overlay.clear();
        if(initial) return initial = false;
        reloadApp();
    },
    warnings: function(warnings) {
        log("info", "[WDS] Warnings while compiling.");
        var strippedWarnings = warnings.map(function(warning) {
            return stripAnsi(warning);
        });
        sendMsg("Warnings", strippedWarnings);
        for(var i = 0; i < strippedWarnings.length; i++)
            console.warn(strippedWarnings[i]);
        if(useWarningOverlay) overlay.showMessage(warnings);

        if(initial) return initial = false;
        reloadApp();
    },
   function reloadApp() {
    //(1)如果开启了HMR模式
    if(hot) {
        log("info", "[WDS] App hot update...");
        var hotEmitter = require("webpack/hot/emitter");
        hotEmitter.emit("webpackHotUpdate", currentHash);
        //重新启动webpack/hot/emitter，同时设置当前hash,通知上面的webpack-dev-server的webpackHotUpdate事件，告诉它打印那些模块的更新信息
        if(typeof self !== "undefined" && self.window) {
            // broadcast update to window
            self.postMessage("webpackHotUpdate" + currentHash, "*");
        }
    } else {
       //(2)如果不是Hotupdate那么我们直接reload我们的window就可以了
        log("info", "[WDS] App updated. Reloading...");
        self.location.reload();
    }
}

```
也就是说当客户端(*打包到我们的entry中的webpack-dev-server提供的websocket的客户端代码*)接受到服务器(*webpack-dev-server接受到webpack提供的compiler对象可以知道webpack什么时候打包完成，通过webpack-dev-server提供的websocket服务端代码通知websocket客户端*)发送的ok和warning信息的时候会要求更新。如果支持HMR的情况下就会要求检查更新，同时发送过来的还有服务器端本次编译的*compilation的hash*值。如果不支持HMR，那么我们要求刷新页面。我们继续深入一步，看看服务器什么时候发送'ok'和'warning'消息：
```js
Server.prototype._sendStats = function(sockets, stats, force) {
    if(!force &&
        stats &&
        (!stats.errors || stats.errors.length === 0) &&
        stats.assets &&
        stats.assets.every(function(asset) {
            return !asset.emitted;
            //(1)每一个asset都是没有emitted属性，表示没有发生变化。如果发生变化那么这个assets肯定有emitted属性
        })
    )
    return this.sockWrite(sockets, "still-ok");
    //(1)将stats的hash写给socket客户端
    this.sockWrite(sockets, "hash", stats.hash);
    //设置hash
    if(stats.errors.length > 0)
        this.sockWrite(sockets, "errors", stats.errors);
    else if(stats.warnings.length > 0)
        this.sockWrite(sockets, "warnings", stats.warnings);
    else
        this.sockWrite(sockets, "ok");
}
```
也就是说更新是通过上面这个方法完成的，我们看看上面这个方法什么时候调用就可以了：
```js
compiler.plugin("done", function(stats) {
         //clientStats表示需要保存stats中的那些属性，可以允许配置，参见webpack官网
        this._sendStats(this.sockets, stats.toJson(clientStats));
        this._stats = stats;
    }.bind(this));
```
是不是豁然开朗了，也就是每次compiler的'done'钩子函数被调用的时候就会要求客户端去检查模块更新，如果客户端不支持HMR，那么就会全局加载。整个过程就是:`webpack-dev-server在用户的入口文件中添加热加载的客户端websocket代码=>webpack-dev-server拿到webpack的compiler对象=>判断是否需要更新=>通过websocket服务端代码通知客户端，并发送compilation的hash值=>客户端判断compilation的hash值是否发生变化并实现热加载以及log打印`。而有一点你需要弄清楚，那就是:我们的webpack-dev-server必须拿着webpack提供的compiler对象才行，具体你可以查看我对webpack-dev-server的一个[封装实例](https://github.com/liangklfangl/wcf/blob/master/src/devServer.js#L136)。

接下来我们来看看"webpack/hot/dev-server":
```js
if(module.hot) {
    var lastHash;
    //__webpack_hash__是每次编译的hash值是全局的
    var upToDate = function upToDate() {
        return lastHash.indexOf(__webpack_hash__) >= 0;
    };
    var check = function check() {
        module.hot.check(true).then(function(updatedModules) {
            //检查所有要更新的模块，如果没有模块要更新那么回调函数就是null
            if(!updatedModules) {
                console.warn("[HMR] Cannot find update. Need to do a full reload!");
                console.warn("[HMR] (Probably because of restarting the webpack-dev-server)");
                window.location.reload();
                return;
            }
            //如果还有更新
            if(!upToDate()) {
                check();
            }
            require("./log-apply-result")(updatedModules, updatedModules);
            //已经被更新的模块都是updatedModules
            if(upToDate()) {
                console.log("[HMR] App is up to date.");
            }

        }).catch(function(err) {
            var status = module.hot.status();
            //如果报错直接全局reload
            if(["abort", "fail"].indexOf(status) >= 0) {
                console.warn("[HMR] Cannot apply update. Need to do a full reload!");
                console.warn("[HMR] " + err.stack || err.message);
                window.location.reload();
            } else {
                console.warn("[HMR] Update failed: " + err.stack || err.message);
            }
        });
    };
    var hotEmitter = require("./emitter");
    //获取MyEmitter对象
    hotEmitter.on("webpackHotUpdate", function(currentHash) {
        lastHash = currentHash;
        if(!upToDate() && module.hot.status() === "idle") {
            //调用module.hot.status方法获取状态
            console.log("[HMR] Checking for updates on the server...");
            check();
        }
    });
    console.log("[HMR] Waiting for update signal from WDS...");
} else {
    throw new Error("[HMR] Hot Module Replacement is disabled.");
}
```
两者的主要代码区别在于check函数的调用方式:

<pre>
check([autoApply], callback: (err: Error, outdatedModules: Module[]) => void
</pre>

如果我们的autoApply设置为true,那么我们回调函数传入的就是所有被自己[dispose处理](https://webpack.js.org/api/hot-module-replacement/#dispose-or-adddisposehandler-)过的模块，同时apply方法也会自动调用，而不需要向`webpack/hot/only-dev-server`一样手动调用`module.hot.apply`。如果auApply设置为false，那么所有的模块更新都会通过手动调用apply来完成。而所说的被自己dispose处理就是通过如下的方式来完成的:
```js
if (module.hot) {
    module.hot.accept();
    //支持热更新
    //当前模块代码更新后的回调，常用于移除持久化资源或者清除定时器等操作，如果想传递数据到更新后的模块，可以通过传入data参数，后续参数可以通过module.hot.data获取
    module.hot.dispose(() => {
        window.clearInterval(intervalId);
    });
}
```
而一般我们调用webpack-dev-server只会添加--hot而已，即内部不需要调用apply，而传入的都是被dispose处理过的模块:
```js
 if(devServerOptions.hotOnly)
        devClient.push("webpack/hot/only-dev-server");
    else if(devServerOptions.hot)
        devClient.push("webpack/hot/dev-server");
```

不管是那种方式，webpack-dev-server实现热加载都具有如下的流程:

![](http://images.gitbook.cn/5f2d93b0-b0ae-11e7-a56a-2b0687e97e1c)


##### 2.3 如何写出支持HMR的代码

这里就是一个例子,你也可以查看[这个仓库](https://github.com/liangklfangl/wcf),然后克隆下来，执行下面命令(注意，这个仓库的代码已经发布到npm的[webpackcc](https://www.npmjs.com/package/webpackcc)):
```js
npm install webpackcc -g
npm run test
```
你就会发现访问localhost:8080的时候代码是可以支持HMR(你可以修改test目录下的所有的文件)，而不会出现页面刷新的情况。下面也给出实例代码:
```js
//time.js
let moduleStartTime = getCurrentSeconds();
//(1)得到当前模块加载的时间，是该模块的一个全局变量，首次加载模块的时候获取到,热加载
//   时候会重新赋值
function getCurrentSeconds() {
    return Math.round(new Date().getTime() / 1000);
}
export function getElapsedSeconds() {
    return getCurrentSeconds() - moduleStartTime;
}
//(2)开启HMR，如果添加HotModuleReplacement插件，webpack-dev-server添加--hot
if (module.hot) {
    const data = module.hot.data || {};
    //(3)如果module.hot.dispose将当前的数据放到了data中可以通过module.hot.data获取
    if (data.moduleStartTime)
        moduleStartTime = data.moduleStartTime;
    //(4)我们首次会将当前模块加载的时间传递到热加载后的模块中，从而热加载后的moduleStartTime
    //   会一直是首次加载模块的时间
    module.hot.dispose((data) => {
        data.moduleStartTime = moduleStartTime;
    });
}
```
在time.js中我们会在每次热加载的时候保存模块首次加载的时间，这是实现热加载后页面time不改变的关键代码。下面再给出index.js的代码:
```js
import * as dom from './dom';
import * as time from './time';
import pulse from './pulse';
require('./styles.scss');
const UPDATE_INTERVAL = 1000; // milliseconds
const intervalId = window.setInterval(() => {
    dom.writeTextToElement('upTime', time.getElapsedSeconds() + ' seconds');
    dom.writeTextToElement('lastPulse', pulse());
}, UPDATE_INTERVAL);
// Activate Webpack HMR
if (module.hot) {
    module.hot.accept();
    // dispose handler
    module.hot.dispose(() => {
        window.clearInterval(intervalId);
    });
}
```
你可能有这样的疑问:"如果我们修改index.js后，我们页面的时间是否就会刷新呢？"答案是:`"不会"`!这是因为:当你改变了index.js的代码，虽然我们会调用clearInterval，但是该模块也是支持热加载的，所以热加载后又会执行window.setInterval，而我们time.js返回的依然是正确的时间。

关于module.hot.dispose有一点需要注意:
```js
module.hot.dispose(function(){
  console.log('1');
    window.clearInterval(intervalId);
})
```
假如在修改index.js之前，我们的代码如上，此时我们修改代码为如下:
```js
module.hot.dispose(function(){
  console.log('2');
    window.clearInterval(intervalId);
})
```
此时你会发现打印出来的结果为1而不是2，即打印的结果是HMR完成之前的代码。这可能是因为这个函数是为了清除持久化资源或者清除定时器等操作而设计的。完整的代码逻辑你一定要[查看这里](https://github.com/liangklfangl/wcf/blob/master/test/index.js)并运行一下，这样可能更好的了解HMR的逻辑。

#### 2.4 HMR牵涉到其他函数与概念

##### 2.4.1 accept函数
```js
accept(dependencies: string[], callback: (updatedDependencies) => void) => void
accept(dependency: string, callback: () => void) => void
```
此时表示，我们这个模块支持HMR，任何其依赖的模块变化都会被捕捉到。当依赖的模块更新后回调函数被调用。当然，如果是下面这种方式：
```js
accept([errHandler]) => void
```
那么表示我们接受当前模块`所有`依赖的模块的代码更新，而且这种更新不会冒泡到父级中去。这当我们模块没有导出任何东西的情况下有用(因为没有导出，所以也就没有父级调用)。

##### 2.4.2 decline函数
上面的例子中我们的dom.js是如下方式写的：
```js
import $ from 'jquery';
export function writeTextToElement(id, text) {
    $('#' + id).text(text);
}
if (module.hot) {
    module.hot.decline('jquery');//不接受jquery更新
}
```
其中decline方法签名如下:
```js
decline(dependencies: string[]) => void
decline(dependency: string) => void
```
这表明我们不会接受特定模块的更新，如果该模块更新了，那么更新失败同时失败代码为"decline"。而上面的代码表明我们不会接受jquery模块的更新。当然也可以是如下模式：
```js
decline() => void
```
这表明我们当前的模块是不会更新的，也就是不会HMR。如果更新了那么错误代码为"decline";

##### 2.4.3其中dispose函数
函数签名如下:

```js
dispose(callback: (data: object) => void) => void
addDisposeHandler(callback: (data: object) => void) => void
```
这表示我们会添加一个一次性的处理函数，这个函数在当前模块更新后会被调用。此时，你需要移除或者销毁一些持久的资源，如果你想将当前的状态信息转移到更新后的模块中，此时可以添加到data对象中，以后可以通过module.hot.data访问。如下面的time.js例子用于保存指定模块实例化的时间，从而防止模块更新后数据丢失(刷新后还是会丢失的)。
```js
if (module.hot) {
    module.hot.accept();
    // dispose handler
    module.hot.dispose(() => {
        window.clearInterval(intervalId);
        //这是更新之前的模块的intervalId，而不是更新后的新的模块
    });
}
```
##### 2.4.4 HMR的module.hot.status() => string
该函数可以获取到HMR当前所处的状态，可以是*idle, check, watch, watch-delay, prepare, ready, dispose, apply, abort or fail*中的任意个:
- idle
  这个状态表明，当然HMR处于空闲状态，可以调用check。调用后状态为check
- check
  HMR在检查模块更新。如果没有模块更新，那么重新回到idle状态。如果有更新那么会依次经过prepare,dispose,apply然后重新回到idle状态
- watch
  HMR当前处于监听模式，他可以自动接收到更新。如果接受到更新，那么就会进入*watch-delay*模式，然后等待机会开始更新操作。如果开始更新，那么会依次经过prepare, dispose 和 apply状态。如果在更新的时候又监听到文件更新，那么重新回到watch或者watch-delay状态
- prepare
  表明HMR在准备更新。比如在下载一些资源，如webpack更新后的*资源下载*过程。
- ready
  可以开始更新了，需要手动调用apply方法去继续更新操作
- dispose
   HMR在调用模块自己的dispose方法，并开始更新后的模块替换操作
- apply
   HMR在调用*被替换后(dispose)*的模块的父级模块的accept方法，当然模块自己必须能够被dispose
- abort
  更新无法被进一步apply，但是文件处于更新之前的一致状态
- fail
  在更新过程中抛出了异常，当前的文件状态处于不一致状态。系统需要重启

上面的源码分析中你也看到了调用方式:
```js
module.hot.status()
```

##### 2.4.5 HMR的apply方法
其中调用的方式如下:
```js
module.hot.apply(options).then(outdatedModules => {
  // outdated modules...
}).catch(error => {
  // catch errors
});
```
其中options可以包含下面的这些参数:
<pre>
 1.ignoreUnaccepted表示调用accept时没有指定的模块。如果accpet没有参数，接受任何模块更新
 2.ignoreDeclined表示调用decline明确指定不需要检查的模块
 3.ignoreErrored忽略在调用accept时候抛出的错误
 4.onDeclined函数，接受那些decline指定的模块
 5.onUnaccepted函数，接受accept中没有指定的模块
 6.onAccepted函数，接受accept中指定的模块
 7.onDisposed函数，接受那些被dispose的模块
 8.onErrored函数，接受那些出错的模块   
</pre>
每一个函数接受到的参数为如下类型:
```js
{
  type: "self-declined" | "declined" | 
        "unaccepted" | "accepted" | 
        "disposed" | "accept-errored" | 
        "self-accept-errored" | "self-accept-error-handler-errored",
  moduleId: 4, 
  // The module in question.
  dependencyId: 3, 
  // For errors: the module id owning the accept handler.
  chain: [1, 2, 3, 4], 
  // For declined/accepted/unaccepted: the chain from where the update was propagated.
  // 这个chain表示更新冒泡的顺序
  parentId: 5, 
  // For declined: the module id of the declining parent
  outdatedModules: [1, 2, 3, 4],
  // For accepted: the modules that are outdated and will be disposed
  outdatedDependencies: { 
  // For accepted: The location of accept handlers that will handle the update
    5: [4]
  },
  error: new Error(...), 
  // For errors: the thrown error
  originalError: new Error(...) 
  // For self-accept-error-handler-errored: 
   // the error thrown by the module before the error handler tried to handle it.
}
```
##### 2.4.6 hotUpdateChunkFilename vs hotUpdateMainFilename
当你修改了test目录下的文件的时候，比如修改了scss文件，此时你会发现在页面中多出了一个script元素，内容如下:
```html
<script type="text/javascript" charset="utf-8" src="0.188304c98f697ecd01b3.hot-update.js"></script>
```
其中内容是：
```js
webpackHotUpdate(0,{
/***/ 15:
/***/ (function(module, exports, __webpack_require__) {
exports = module.exports = __webpack_require__(46)();
// imports
// module
exports.push([module.i, "html {\n  border: 1px solid yellow;\n  background-color: pink; }\n\nbody {\n  background-color: lightgray;\n  color: black; }\n  body div {\n    font-weight: bold; }\n    body div span {\n      font-weight: normal; }\n", ""]);
// exports

/***/ })
})
//# sourceMappingURL=0.188304c98f697ecd01b3.hot-update.js.map
```
从内容你也可以看出，只是将我们修改的模块push到exports对象中！而hotUpdateChunkFilename就是为了让你能够执行script的src中的值的！而同样的hotUpdateMainFilename是一个json文件用于指定哪些模块发生了变化，在output目录下。

##### 2.5 less/scss/css的热加载
要实现less/scss/css的热加载是非常容易的,我们可以直接使用style-loader来完成(在开发模式下，生产模式下不建议使用)。比如在开发模式下对于css的加载可以配置如下的loader:
```js
   module: {
     rules: [
       {
         test: /\.css$/,
         use: ['style-loader', 'css-loader']
       }
     ]
   }
```
对于style-loader热加载的你可以直接[点击这里](https://github.com/webpack-contrib/style-loader/blob/master/index.js#L24)，其中原理上面都说过了，如果不懂，请仔细阅读上面的HMR的部分。而至于less/scss因为最终都会打包成为css，所以其实和css的热加载是一样的道理。


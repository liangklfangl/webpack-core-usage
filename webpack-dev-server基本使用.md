#### 1.本章概述
在本章节，我们将简要的对webpack-dev-server的基本使用做一个演示。通过这个章节，你可以学会如何在自己的项目中使用webpack-dev-server。

#### 2.webpack-dev-server使用
第一步:安装webpack-dev-server
```js
npm install webpack-dev-server -g
```
第二步:在项目根目录下配置webpack.config.js
```js
var path = require("path");
module.exports = {
  entry: {
    app: ['./src/main.js']
  },
  output: {
    path: path.resolve(__dirname, "public"),
    publicPath: "",
    filename: "bundle.js"
  }
};
```
其中里面的各项配置，通过前面的章节你应该能够理解，此处不再赘述，如果不懂，可以仔细阅读前面的章节。

第三步:配置package.json中的scripts部分
```js
  "scripts": {
    "build": "webpack",
    "start": "webpack-dev-server --inline --hot --port 3000 --content-base public",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```
其中对于script部分不太了解的可以查看我写的[package.json中的scripts部分深入讲解](https://github.com/liangklfangl/npm-command#4packagejson中script部分深入讲解)。通过配置scripts你可以使用如下命令:
```js
npm start
//或者
npm run start
```
来替换掉:
```js
webpack-dev-server --inline --hot --port 3000 --content-base public
```
当然，如果你不想做替换，依然可以在目录下运行你全局安装的webpack-dev-server命令。此时需要的所有配置都已经完成了，启动你的*npm run start*就会启动一个Express服务器，端口号是3000，同时支持HMR，contentBase为我们指定的/public。

#### 3.webpack-dev-server的inline模式与iframe模式
上面的打包实例中你看到我在cli中传入了*--inline*，表示这是内联的打包方式，那么什么是内联的打包方式呢?
##### 3.1 iframe模式
我们首先看看Express服务器是如何处理iframe模式的:
```js
  app.get('/webpack-dev-server/*', (req, res) => {
    res.setHeader('Content-Type', 'text/html');
    fs.createReadStream(path.join(__dirname, '..', 'client', 'live.html')).pipe(res);
  });
```
所以当我们在URL中加入webpack-dev-server的路径以后，我们就会返回live.html，我们再看看live.html的内容:
```js
<!DOCTYPE html>
<html>
 <head>
  <meta http-equiv="X-UA-Compatible" content="IE=edge" />
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, height=device-height, initial-scale=1.0, user-scalable=no, minimum-scale=1.0, maximum-scale=1.0" />
  <script type="text/javascript" charset="utf-8" src="/__webpack_dev_server__/live.bundle.js"></script>
 </head>
 <body> 
 </body>
</html>
```
其直接返回了*/__webpack_dev_server__/live.bundle.js*。我们看看这个请求Express服务器返回的资源内容:
```js
 app.get('/__webpack_dev_server__/live.bundle.js', (req, res) => {
    res.setHeader('Content-Type', 'application/javascript');
    fs.createReadStream(path.join(__dirname, '..', 'client', 'live.bundle.js')).pipe(res);
  });
```
所以我们最终返回的还是client/live.bundle.js，我们就来看看live.bundle.js的具体内容。首先它会在页面中创建一个iframe并注册了客户端代码:
```js

  const onSocketMsg = {
    hot() {
      hot = true;
      iframe.attr('src', contentPage + window.location.hash);
    },
    invalid() {
      okness.text('');
      status.text('App updated. Recompiling...');
      header.css({
        borderColor: '#96b5b4'
      });
      $errors.hide();
      if (!hot) iframe.hide();
    },
    hash(hash) {
      currentHash = hash;
    },
    'still-ok': function stillOk() {
      okness.text('');
      status.text('App ready.');
      header.css({
        borderColor: ''
      });
      $errors.hide();
      if (!hot) iframe.show();
    },
    ok() {
      okness.text('');
      $errors.hide();
      reloadApp();
    },
    warnings() {
      okness.text('Warnings while compiling.');
      $errors.hide();
      reloadApp();
    },
    errors(errors) {
      status.text('App updated with errors. No reload!');
      okness.text('Errors while compiling.');
      $errors.text(`\n${stripAnsi(errors.join('\n\n\n'))}\n\n`);
      header.css({
        borderColor: '#ebcb8b'
      });
      $errors.show();
      iframe.hide();
    },
    close() {
      status.text('');
      okness.text('Disconnected.');
      $errors.text('\n\n\n  Lost connection to webpack-dev-server.\n  Please restart the server to reestablish connection...\n\n\n\n');
      header.css({
        borderColor: '#ebcb8b'
      });
      $errors.show();
      iframe.hide();
    }
  };
  socket('/sockjs-node', onSocketMsg);
```
这样，当webpack的资源发生变化以后可以通过websocket通知到我们这个主页面。我们看看主页面最重要的一个方法，即*reloadApp*方法，方法内容如下:
```js
  function reloadApp() {
    //如果开启了HMR功能
    if (hot) {
      status.text('App hot update.');
      try {
        iframe[0].contentWindow.postMessage(`webpackHotUpdate${currentHash}`, '*');
      } catch (e) {
        console.warn(e); // eslint-disable-line
      }
      iframe.show();
    } else {
      status.text('App updated. Reloading app...');
      header.css({
        borderColor: '#96b5b4'
      });
      try {
        let old = `${iframe[0].contentWindow.location}`;
        if (old.indexOf('about') === 0) old = null;
        iframe.attr('src', old || (contentPage + window.location.hash));
        if (old) {
            //强制刷新
          iframe[0].contentWindow.location.reload();
        }
      } catch (e) {
        iframe.attr('src', contentPage + window.location.hash);
      }
    }
  }
});
```
这个主页面接受到事件以后，通过postMessage将当前打包的hash值发送到我们的内部的iframe。如果开启了HMR，那么iframe会检查资源更新，如果没有开启HMR，那么强制iframe进行刷新。上面讲了很多原理的知识，我们下面给出一个日常实例:

我们的页面被嵌套在一个iframe中，当资源改变的时候会重新加载。只需要在路径中加入webpack-dev-server就可以了,不需要其他的任何处理:
```js
http://localhost:8080/webpack-dev-server/index.html
```
从而在页面中就会产生如下的一个iframe标签并注入css/js/DOM:

![](https://github.com/liangklfangl/webpack-dev-server/blob/master/iframe.png)

这个主页面会请求live.bundle.js ,其中里面会新建一个Iframe ，你的应用就被注入到了这个 iframe 当中。同时live.bundle.js中含有socket.io的client代码，这样它就能和 webpack-dev-server建立的 http server 进行 websocket 通讯了，并根据返回的信息完成相应的动作。

总之，因为我们的http://localhost:8080/webpack-dev-server/index.html访问的时候加载了live.bundle.js，其具有websocket的client代码，所以当websocket-dev-server服务端代码发生变化的时候会通知到这个页面，这个页面只是需要重新刷新iframe中的页面就可以了。该模式有如下作用:
<pre>
No configuration change needed.（不需要修改配置文件）
Nice information bar on top of your app.(在app上面有information bar)
URL changes in the app are not reflected in the browser’s URL bar.(在app里面的URL改变不会反应到浏览器的地址栏中)
</pre>
##### 3.2 inline mode
webpack-dev-server的客户端入口被添加到文件中，用于自动刷新页面。其中在cli中输入的是:
```js
  webpack-dev-server --inline --content-base ./build
```
此时在页面中输出的内容中看不到插入任何的js代码:

![](https://github.com/liangklfangl/webpack-dev-server/blob/master/inline.png)

但是在控制台中可以清楚的知道页面的重新编译等信息:

![](https://github.com/liangklfangl/webpack-dev-server/blob/master/reload.png)

该模式有如下作用:
<pre>
Config option or command line flag needed.(webpack配置或者命令行配置)
Status information in the console and (briefly) in the browser’s console log.(状态信息在浏览器的console.log中)
URL changes in the app are reflected in the browser’s URL bar(URL的改变会反应到浏览器的地址栏中).
</pre>
每一个模式都是支持Hot Module Replacement的，在HMR模式下，每一个文件都会被通知内容已经改变而不是重新加载整个页面。因此，在HMR执行的时候可以加载更新的模块，从而把他们注册到运行的应用里面。不管是inline模式还是iframe模式，我们都是通过socketjs来连接客户端代码和webpack-dev-server的服务端代码，其原理都是一样的。

### 3.本章小结
该本章节中，我们讲解了如何在你的项目中使用webpack-dev-server来启动一个Express服务器，同时也分析了webpack-dev-server本身具有的inline模式和iframe模式。并深入了讲解了iframe模式的原理，希望你能对webapck-dev-server有一个更加深入的了解。文中提到的示例代码你可以[点击这里下载](https://github.com/liangklfang/webpack-dev-server-demo)。


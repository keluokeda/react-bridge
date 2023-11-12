项目中用到了[webview_flutter](https://pub.dev/packages/webview_flutter)这个插件，它允许我们在Flutter里面嵌套WebView，同时我们也可以利用
**addJavaScriptChannel**方法来让H5和我们交互。
但现在有一个问题就是，**addJavaScriptChannel**可以让h5单向给flutter发消息，但如何在收到消息后回调h5呢？

我们可以利用**runJavaScriptReturningResult**方法来调用h5里面js的方法，
例如
```
controller.runJavaScriptReturningResult(
            "window.JSBridgeCallback($id, '${json.encode(result)}')");
```
这段代码的的意思就是执行js中的
```
window.JSBridgeCallback = JSBridgeCallback;

const JSBridgeMap = {};
let callID = 0;

function JSBridgeCallback(id, params) {
    console.log("JSBridgeCallback", id, params);
    JSBridgeMap[id](params);
    JSBridgeMap[id] = null;
    delete JSBridgeMap[id];
}
```
方法。

接下来是具体步骤


1. 在flutter里面，调用**addJavaScriptChannel**方法来在js的**window**对象上挂载一个对象

```
addJavaScriptChannel("Flutter", onMessageReceived: (message) {
        print("Flutter received: ${message.message}");

        final jsonObject = json.decode(message.message);
        final id = jsonObject['callID'];

        const result = {"version": 10};

        controller.runJavaScriptReturningResult(
            "window.JSBridgeCallback($id, '${json.encode(result)}')");
      })
```
我们需要h5告诉我们回调id、方法名称和参数


2. 在H5项目中新建一个js文件，添加如下内容
```
export function callFlutter(name, params, callback) {
    if (window.Flutter) {
        const id = callID++;
        const paramsObj = {
            method: name,
            callID: id,
            data: params || null
        }
        JSBridgeMap[id] = callback || ((result) => {
        });
        // JSBridgeHandle.call(method, JSON.stringify(paramsObj));
        window.Flutter.postMessage(JSON.stringify(paramsObj));
    }

}


window.JSBridgeCallback = JSBridgeCallback;

const JSBridgeMap = {};
let callID = 0;

function JSBridgeCallback(id, params) {
    console.log("JSBridgeCallback", id, params);
    JSBridgeMap[id](params);
    JSBridgeMap[id] = null;
    delete JSBridgeMap[id];
}
```
我们声明一个**JSBridgeCallback**方法，并把它挂载到window上，叫什么名字看你的喜好，不一定非要这个名字。
暴露**callFlutter**给外面调用，**callFlutter**代码的大致意思是
把回调函数保存到JSBridgeMap里面，然后把方法id、参数和方法名称打包成一个json通过**window.Flutter.postMessage**传给flutter。

3. 在H5的一个按钮上调用callFlutter方法，给flutter传递参数并拿到flutter的回调，我这里用的框架是React
```
function App() {

    const [text, setText] = React.useState('Hello, Flutter!');


    return (
        <div className="App">
            <h1>{text}</h1>
            <button onClick={() => {
                callFlutter('getAppVersion', {method: 'showToast', args: ['Hello, Flutter!']}, (result: any) => {

                    // console.log("收到了来自Flutter的回调: " + result);
                    setText(result);
                });
            }}>点我
            </button>
        </div>
    );
}
```

#### 测试
我们在flutter里面的h5页面点击按钮，拿到了flutter传来的结果并显示
![Screenshot_20231112-211205.png](https://upload-images.jianshu.io/upload_images/3690197-763f0c36fcb4ccbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[flutter项目地址](https://github.com/keluokeda/flutter-bridge)
[React项目地址](https://github.com/keluokeda/react-bridge)

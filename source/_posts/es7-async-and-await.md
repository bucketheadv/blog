---
title: es7中的async/await
date: 2016-10-15
tags: ["node", "es7"]
---

Node.js es7标准中加入了async/await用于处理异步函数。 `async`标记的函数中方能调用`await`, `await`用于等待异步函数的返回值, 而`await`目标必须是一个`promise`对象, 因此实际上`await`就是对`promise`对象取它的`resolve`值。

<!--more-->

### 使用babel-node运行es7代码

```bash
npm install -g babel-cli
npm install babel-preset-es2015 babel-preset-stage-2 --save-dev
```

然后添加`.babelrc`文件，添加以下内容:

```json
{
  "presets": [
    "es2015",
    "stage-2"
  ],  
  "plugins": []
}
```

此时就可以使用`babel-node app.js`运行es7代码了。

### 另一种方式

```
npm install babel-core babel-polyfill --save
```

在程序入口文件 `index.js`中:

```js
require('babel-core/register')
require('babel-polyfill')
require('./app')
```

这样一来也可以在`app.js`或其他文件中编写es7代码了。

### 传统的Promise生成

#### 【示例代码1】
```js
function readWithPromise(filePath) {
  return new Promise((resolve, reject) => {
    fs.readFile(filePath, (err, data) => {
      if (err) {
        reject(err);
      } else {
        resolve(data);
      }
    });
  });
}
```

这段代码就是一返回一个promise对象,它的作用是读取一个文件的内容，因此我们要调用时只需要如下操作:

#### 【示例代码2】
```js
async function run() {
  let data;
  try {
    data = await readWithPromise("./package.json");
    //data = await readWithGenerator("./package.json");
    console.log(data.toString());
  } catch(e) {
    console.error(e);
  }
}
run();
```

### 以Generator方式

除此之外，还有一种生成`promise`对象的方法, 那便是`generator`, 使用这种方法需要引用`co`这个包, 代码如下:

#### 【示例代码3】

```js
import co from 'co';
function readWithGenerator(filePath) {
  return co(function*(){
    let content = yield ((fp) => {
      return (callback) => {
        fs.readFile(fp, "utf8", callback);
      }
    })(filePath);
    return content;
  });
}
```

`【示例代码1】`与`【示例代码3】`作用完全一样, 最后它们都返回一个`promise`对象，因此调用方式也一样(见`【示例代码2】`).

### 小结

`await` 仅仅只是`es7`加入的默认对`promise`的一种处理机制而已，它的作用其实类似于`generator`里面的`yield`。`yield`后面加的是一个`generator`函数，而`await`后面接的是一个`promise`对象，但它们都会在此等待结果返回后才会执行之后的代码。

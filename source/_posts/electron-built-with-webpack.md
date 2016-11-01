---
title: '使用webpack构建electron项目'
tags: ['webpack', 'electron', 'es6']
date: 2016-10-23
---

Webpack是一个很强大的构建工具，不过也有不少坑。

<!--more-->

## 新建项目

```bash
mkdir todo && cd todo
npm init
```

## 安装必要的package

```bash
### 安装webpack相关包
npm install --save-dev webpack html-webpack-plugin
### 安装babel相关包
npm install --save-dev babel-core babel-loader babel-plugin-transform-es2015-spread babel-plugin-transform-object-rest-spread babel-preset-es2015 babel-preset-react
### 安装electron相关包
npm install --save electron
npm install --save-dev electron-packager
### 安装react相关包
npm install --save react react-dom react-tap-event-plugin
### UI相关包
npm install --save material-ui
```

## 项目结构图

![img-1](/1.png)

## webpack配置

```js
var webpack = require('webpack')
var path = require('path')
var fs = require('fs')
var HtmlWebpackPlugin = require('html-webpack-plugin');

var config = {
  // key是生成文件的路径, value是源文件路径
  entry: {
    'js/bundle': './src/components/index.jsx',
    'main': './src/main.jsx'
  },

  // 解决__dirname和__filename路径混乱的问题
  node: {
    __filename: false,
    __dirname: false
  },

  // webpack重实现了require方法，导致大量原生包无法调用，据说也可以添加这句："var fs = global.require('fs')"
  target: 'atom',

  output: {
    path: path.join(__dirname, 'dist'),
    filename: "[name].js"
  },

  module: {
    loaders: [
      {
        //test: path.join(__dirname, 'es6'),
        test: /\.jsx$/,
        exclude: /node_modules/,
        loader: 'babel-loader',
        query: {
          presets: ['es2015', 'react'],
          plugins: ['transform-object-rest-spread']
        }
      },
      /*
      {
        test: /\.(gif|jpg|png|woff|svg|eot|ttf)\??.*$/,
        loader: 'url-loader?limit=50000&name=[path][name].[ext]'
      }
      */
    ]
  },

  resolve: [
    {
      root: path.resolve(__dirname, "src"),
      extensions: ['', '.js', '.jsx', '.css', '.scss']
    }
  ],

  plugins: [
    new HtmlWebpackPlugin({
      filename: 'html/index.html',
      inject: false, //不添加entry列表里的文件到index.html
      template: path.join(__dirname, "/src/html/index.html") //new 一个这个插件的实例，并传入相关的参数
    })
  ]
}

module.exports = config

```

webpack配置中有几条是比较坑的，

- 首先是`target: 'atom'`, 若不配置这个选项，那么electron在起动时会报错，错误内容是`require('fs')时，fs包不存在`。原因在于，webpack重新实现了require方法，因此很多原生包无法require，添加此配置即解决。

- 其次是 `node: { __dirname: false, __filename: false }` , 若不添加此配置，那么打包后的代码会修改`__dirname`和`__filename`的值导致无法获取正确的文件夹和文件路径。

- 最后是HtmlWebpackPlugin的 `inject: false`, 这个插件是用于打包`html`文件，默认会将所有生成的所有`javascript`文件插入到此`html`中，添加`inject: false`后将不再生成引入`javascript`文件的标签。

webpack打包后的文件放在dist文件夹下，因此程序入口调用的文件为`dist/main.js`。

## package.json配置

主要是main和scripts的配置，如下：

```json
{
  "main": "dist/main.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "electron .",
    "start": "npm run build && NODE_ENV=development electron .",
    "build": "node_modules/webpack/bin/webpack.js --progress --colors",
    "pack": "node_modules/electron-packager/cli.js ./ restron --platform=darwin --version=1.4.4 --out=./package --overwrite --icon=~/Node/electron/todo/public/img/tray.icon"
  }
}
```


入口文件，用于调用webpack打包后的文件。

### main.jsx

```js
const electron = require('electron')
const {app, BrowserWindow, ipcMain} = electron
const Tray = electron.Tray
const path = require('path');

let mainWindow = null
let appIcon;

let env = process.env.NODE_ENV || 'production'

app.on('ready', function() {
  let iconPath = path.join(__dirname, "/../public/img/tray.icns")
  mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    icon: iconPath
  })


  //appIcon = new Tray(iconPath);
  //appIcon.setToolTip('This is my application');

  mainWindow.loadURL('file://' + __dirname + '/html/index.html')

  if (env == 'development') {
    mainWindow.webContents.openDevTools()
  }

  mainWindow.on('closed', () => {
    mainWindow = null
  })
})

app.on('window-all-closed', () => {
  app.quit()
})

```

程序窗口的建立，建立完成后载入`html/index.html`, 此html文件中又加载了`components/index.js`渲染组件。

### components/index.jsx

```js
import React from 'react'
import ReactDOM from 'react-dom'
import injectTapEventPlugin from 'react-tap-event-plugin'

const events = window.require('events')
const path = window.require('path')
const fs = window.require('fs')

const electron = window.require('electron')
const {ipcRenderer, shell} = electron
const {dialog} = electron.remote

import getMuiTheme from 'material-ui/styles/getMuiTheme'
import MuiThemeProvider from 'material-ui/styles/MuiThemeProvider'
import TextField from 'material-ui/TextField'
import RaisedButton from 'material-ui/RaisedButton'

let muiTheme = getMuiTheme({
  fontFamily: 'Microsoft YaHei'
})

class MainWindow extends React.Component {
  constructor(props) {
    super(props)
    injectTapEventPlugin()

    this.state = {
      username: '',
      password: ''
    }
  }

  render() {
    return (
      <MuiThemeProvider muiTheme={muiTheme}>
        <div style={styles.root}>
          <img style={styles.icon} src="../../public/img/app-logo.jpg" />

          <TextField
            hintText="请输入用户名"
            value={this.state.username}
            onChange={(event) => {this.setState({username: event.target.value})}}
          />

          <TextField
            hintText="请输入密码"
            type="password"
            value={this.state.password}
            onChange={(event) => {this.setState({password: event.target.value})}}
          />

          <div style={styles.buttons_container}>
            <RaisedButton
              label="登录"
              primary={true}
              onClick={this._handleLogin.bind(this)}
            />
            <RaisedButton
              label="注册"
              primary={false}
              style={{marginLeft: 60}}
              onClick={this._handleRegistry.bind(this)}
            />
          </div>
        </div>
      </MuiThemeProvider>
    )
  }

  _handleRegistry() {}
  _handleLogin() {
    let options = {
      type: 'info',
      buttons: ['确定'],
      title: '登录',
      message: this.state.username,
      defaultId: 0,
      cancelId: 0
    }

    dialog.showMessageBox(options, (res) => {
      if (res == 0) {
        console.log('OK pressed!');
      }
    })
  }
}

const styles = {
  root: {
    position: 'absolute',
    left: 0,
    top: 0,
    right: 0,
    bottom: 0,
    display: 'flex',
    flex: 1,
    flexDirection: 'column',
    alignItems: 'center',
    justifyContent: 'center'
  },

  icon: {
    width: 100,
    height: 100,
    marginBottom: 40
  },

  buttons_container: {
    paddingTop: 30,
    width: '100%',
    display: 'flex',
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center'
  }
}

ReactDOM.render(
  <MainWindow />,
  document.getElementById('app')
)
```

主要的组件，此组件实现了一个登录页面的表单。

### html/index.html

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Restron</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="text/javascript" src="../js/bundle.js"></script>
  </body>
</html>
```

html代码中调用`../js/bundle.js`来渲染当前页面，它通过调用`ReactDOM.render`渲染已经定义的组件。

## 运行

代码已经编写完成，执行`npm start`运行webpack打包并运行程序，`npm run build`是用webpack打包，`npm run pack`打包成app文件。

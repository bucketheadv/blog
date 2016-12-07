---
title: es7修饰器
tags: ['es7', 'decorator', 'babel']
date: 2016-11-21
---

es7的新特性之一就是加入了修饰器(decorator)。

<!--more-->

## 安装es7转码包

```bash
npm install --save-dev babel-plugin-transform-decorators-legacy babel-plugin-transform-class-properties
```

## 配置.babelrc

```json
{
  "plugins": ["transform-decorators-legacy", "transform-class-properties"]
}
```

## 装饰器实例

```js
function deprecated(target) {
  console.log("method is deprecated");
}

class A {
  @deprecated
  m() {}
}

```

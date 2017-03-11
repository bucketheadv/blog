---
title: 使用pm2来启动并监控你的守护进程
date: 2017-03-11
tags: ["Node.js", "pm2"]
---

pm2是基于Node.js开发的集代码发布、部署与监控为一体的工具。

<!--more-->

## 安装pm2

```sh
npm i -g pm2
```

## 支持的发布程序类型(interpreter)

- .sh (bash)
- .pl (perl)
- .py (python)
- .rb (ruby)
- .coffee (coffeescript)
- .php (php)
- .js (node)

pm2 支持以上所有类型的代码启动，当运行`pm2 start app.js`时，由于文件类型为`.js`，pm2将自动设置`interpreter`为`node`，其余同理。若要发布`golang`程序，则只需要编写一个`bash`脚本来启动`golang`二进制程序，然后`pm2`启动这个`bash`脚本即可。

## pm2的优点

- 无缝重启  当重启失败时，将不会杀死老进程，保证服务继承可用(必须是cluster模式启动，然后以reload命令重启才可以)
- 提供多重环境部署  通过`ecosystem.config.js`可以实现一条命令即可将代码发布到相应的环境中
- 提供回滚机制  pm2每发布一个版本都会进行备份，可以通过`revert`命令回滚到上一个版本
- 命令行强大  pm2提供了大量简短的命令可以提升工作效率
- 内部负载均衡  pm2以`cluster`模式启动多进程时，内部提供了负载均衡的机制
- 内存溢出时自动重启  通过配置进程占用内存上限，当内存溢出时自动重启
- 支持docker部署  提供了pm2-docker命令
- 提供apm在线监控  可以在线使用`keymetrics`服务进行监控
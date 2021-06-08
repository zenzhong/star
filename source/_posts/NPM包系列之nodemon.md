---
title: NPM包系列之nodemon
date: 2021-06-08 08:45:24
tags: NPM包
---

本文是NPM包系列的第一篇，关注工具包nodemon。

## 简介

开发node项目时，当更改文件之后，只能重新运行才能看到新的代码运行效果，非常不方便。因此，需要一个能监听文件变化自动重启的工具，那就是`nodemon`。

`nodemon`不需要对项目做任何更改，只需要在原来启动项目的命令中使用`node`的地方替换成`nodemon`即可。

## 安装

```bash
# 全局
npm i -g nodemon
# 项目开发依赖
npm i -D nodemon
```

## 使用

直接使用`nodemon -h`可以查看具体使用方法：

```
Usage: nodemon [options] [script.js] [args]

  Options:

  --config file ............ 指定 nodemon.json 配置文件路径
  -e, --ext ................ 指定监听的文件后缀： 如 js,pug,hbs
  -x, --exec app ........... 指定执行环境： 如 -x "python -v"
  -w, --watch path ......... 指定监听的文件或文件夹
  -i, --ignore ............. 忽略特定的文件或文件夹
  -V, --verbose ............ 输出造成项目重启的原因日志
  -- <your args> ........... 为nodemon执行的script传入参数

  注意: 如果未传script脚本参数，nodemon会从 package.json 读取 main 字段。
  如果没有nodemon.json配置文件，nodemon会默认监听 .js, .mjs, .coffee,
  .litcoffee, .json 后缀的文件。

  获取nodemon.json的高级用法: nodemon --help config
  样例: https://github.com/remy/nodemon/wiki/Sample-nodemon.json

  命令示例:

  $ nodemon server.js
  $ nodemon -w ../foo server.js apparg1 apparg2
  $ nodemon --exec python app.py
  $ nodemon --exec "make build" -e "styl hbs"
  $ nodemon app.js -- --config # 向 app.js 脚本传入 config 参数

  获取所有options: nodemon --help options
```

## 配置文件

配置样例：[https://github.com/remy/nodemon/wiki/Sample-nodemon.json](https://github.com/remy/nodemon/wiki/Sample-nodemon.json)

```json
{
  // 手动重启命令配置：rs
  "restartable": "rs",
  // 忽略文件
  "ignore": [
    ".git",
    "node_modules/**/node_modules"
  ],
  // 是否输出详细日志：输出造成项目重启的原因日志
  "verbose": true,
  // 配置不同文件的执行命令环境，主要用于nodemon本身无法执行的场景，如python、ts等
  "execMap": {
    "js": "node --harmony"
  },
  // 监听文件
  "watch": [
    "test/fixtures/",
    "test/samples/"
  ],
  // 环境变量
  "env": {
    "NODE_ENV": "development"
  },
  // 监听文件后缀
  "ext": "js json",
  // 事件触发
  // start - 子进程启动
  // crash - 子进程崩溃，nodemon不会触发exit
  // exit - 子进程完全退出，非crash
  // restart([ 触发重启的文件数组 ]) - 子进程重启
  // config:update - nodemon的配置更新了
  "events": {
    "restart": "osascript - e 'display notification \"app restarted\"' with title \"nodemon\"'"
  }
}
```

了解以上信息基本上够用了，完整的详情可查看[官网](https://www.npmjs.com/package/nodemon)。
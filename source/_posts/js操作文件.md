---
title: js操作文件
code_block_shrink: false
date: 2025-07-24 15:01:13
tags:
categories:
sticky:
---
一个 Node.js 环境配置脚本，实现一些简单的文件操作，环境切换，日志颜色切换
<!-- more -->
# js操作文件

## 1. 实现的简单功能

1. 启动脚本并传入环境参数
2. 读取指定环境的配置文件，将配置写入通用配置文件 .env.json
3. 获取文件路径
4. 执行文件系统管理操作
5. 输出彩色日志信息

## 2. 功能实现代码展示

### 2.1  启动脚本并传入环境参数

在 命令行 执行 `node ./magic-env.js sit `,sit 为环境参数

```javascript
// 打印日志
console.log('111111');
// 获取环境变量
const argv = require('process').argv;
console.log(argv, 'argv');
let env = argv[argv.length - 1];
console.log(env, 'env');
```

### 2.2 环境变量处理
- 通过 `process.argv` 获取命令行参数中的环境标识
- 读取对应环境的配置文件 `.env-${env}.json`
- 将读取的配置写入到 `.env.json` 文件中，实现环境切换

```javascript
// 文件替换，读取修改
const fs = require('fs');
const path = require('path');
const data = fs.readFileSync(
  path.resolve(__dirname, `./.env-${env}.json`),
  'utf-8'
);
fs.writeFileSync(path.resolve(__dirname, './.env.json'), data, 'utf-8');
console.log('环境变量写入成功', data);
```

### 2.3 文件路径操作
- 使用 `path.join` 和 `path.resolve` 构建文件路径
- 示例中构建了 `src/common/util/crypto.js` 和 `.env.json` 的路径

```javascript
// 获取文件路径  __dirname为当前文件所在目录
const ADir = path.join(
  __dirname,
  'src',
  'common',
  'util',
  'crypto.js'
);
const BDir = path.join(
  __dirname,
  '.env.json'
);
console.log(__dirname, ADir, BDir);
```

### 2.4 文件夹递归删除
- 实现了 `deleteFolderRecursive` 函数用于递归删除文件夹
- 遍历文件夹内所有文件和子文件夹，逐一删除后删除根文件夹
- 当前配置删除 `aaa` 文件夹

```javascript
// 删除文件夹的方法
const deleteFolderRecursive = function (path) {
  var files = [];
  if (fs.existsSync(path)) {
    files = fs.readdirSync(path);
    files.forEach(function (file, index) {
      var curPath = path + "/" + file;
      if (fs.statSync(curPath).isDirectory()) {
        deleteFolderRecursive(curPath);
      } else {
        fs.unlinkSync(curPath);
      }
    })
    fs.rmdirSync(path);
  }
}
const deleteDir = path.join(
  __dirname,
  'aaa'
);
deleteFolderRecursive(deleteDir);
```

### 2.5 控制台彩色输出
- 定义了 `colors` 对象，包含各种 ANSI 颜色代码
- 可以实现控制台文本的彩色显示，提升日志可读性
- 示例中使用红色显示文本

```javascript
// 日志打印颜色
const colors = {
  reset: "\x1b[0m",
  bold: "\x1b[1m",
  underline: "\x1b[4m",
  black: "\x1b[30m",
  red: "\x1b[31m",
  green: "\x1b[32m",
  yellow: "\x1b[33m",
  blue: "\x1b[34m",
  magenta: "\x1b[35m",
  cyan: "\x1b[36m",
  white: "\x1b[37m",
  bgBlack: "\x1b[40m",
  bgRed: "\x1b[41m",
  bgGreen: "\x1b[42m",
  bgYellow: "\x1b[43m",
  bgBlue: "\x1b[44m",
  bgMagenta: "\x1b[45m",
  bgCyan: "\x1b[46m",
  bgWhite: "\x1b[47m"
};
console.log(colors.red,`颜色变红`, '=====>', env + colors.reset)
```

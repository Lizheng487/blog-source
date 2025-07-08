---
title: URL路径解析
date: 2025-07-08 13:52:38
tags: JavaScript
categories: JavaScript
code_block_shrink: false
sticky:
---

解析 URL 参数为对象 parseParam

<!-- more -->

## 实现原理

- 使用`new URL()`或者`window.location.hash`获取到 URL 对象。
- `new URLSearchParams()` 创建一个参数对象。
- 最后使用 `for...of` 循环遍历参数对象，将参数名和参数值组成对象并返回。

## 代码实现

### 核心逻辑

```javascript
const parseParam = (url) => {
  const urlObj = new URL(url);
  const params = new URLSearchParams(urlObj.search);
  const paramObj = {};
  for (const [key, value] of params) {
    paramObj[key] = value;
  }
  return paramObj;
};
parseParam("https://www.example.com?name=John&age=30");
// 输出：{ name: 'John', age: '30' }
```

对代码进行优化，优化只有一个参数的情况、添加判断条件、数字类型判断。

### 优化完整代码

```javascript
/**
 * @param {*} url
 * @returns
 */
function parseParam(url) {
  const urlObj = new URL(url);
  const queryParams = new URLSearchParams(urlObj.search);

  const paramsObj = {};

  for (let [key, value] of queryParams.entries()) {
    if (value === "") {
      paramsObj[key] = true;
    } else {
      let val = decodeURIComponent(value);
      val = /^\d+$/.test(val) ? parseFloat(val) : val;
      if (paramsObj.hasOwnProperty(key)) {
        paramsObj[key] = [].concat(paramsObj[key], val);
      } else {
        paramsObj[key] = val;
      }
    }
  }

  return paramsObj;
}
```

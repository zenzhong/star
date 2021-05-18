---
title: dayjs源码解析
date: 2021-05-18 09:52:14
tags: 源码解析
---

本文内容基于`day.js`的`v1.10.4`版本，将主要从基础理念、工程架构、代码结构、插件体系和国际化体系来解析dayjs源码。

## 基础理念

`dayjs`是国内饿了么团队`iamkun`大佬开发，同时他也是`Element UI`的核心开发者。`dayjs`首次发布在2018年4月，开发初衷就是为了对标`momentjs`从而取代它，因此API的设计与`momentjs`完全一致。正如作者所说：

> Day.js is a minimalist JavaScript library that parses, validates, manipulates, and displays dates and times for modern browsers with a largely Moment.js-compatible API. If you use Moment.js, you already know how to use Day.js.

那`dayjs`究竟解决了哪些`momentjs`的痛点？又是通过什么方式解决的呢？看完全文应该就会有答案。

## 工程架构

### 兼容性

通过`package.json`可以看到项目引入了`babel`用于兼容性转化，使用`karma`和`SauceLabs`进行浏览器兼容性测试，确保浏览器兼容性。另外使用`cross-env`确保`npm scripts`运行命令的兼容性。

### 大小

`dayjs`为了确保最终打包大小在宣传的`2kb`，在项目引入了`size-limit`和`gzip-size-cli`，用于死守`2.99kb`大小的底线。当然，确保大小更重要的还是代码层面的设计，将`国际化体系`和`插件体系`独立在核心之外，这个在后面会有介绍。

### Typescript支持

`dayjs`使用`javascript`进行开发，为了支持`Typescript`，单独书写了各种类型和接口定义，放置在`types`文件夹下。

### 单元测试

`dayjs`主要使用`jest`进行单元测试，需要关注的是开发依赖里面包含`moment`和`moment-timezone`，安装这两个包主要是为了进行两者API一致性的单元测试。另外`mockdate`这个包是用于修改当前时间进行单元测试使用。

### 打包

`dayjs`使用`rollup`进行打包压缩，并使用`ncp`进行代码拷贝输出结果。

### 规范

代码规范使用`eslint`和`prettier`来保障，`eslint`主要采用`airbnb`规范以优化`js`代码格式规范，`prettier`用于优化`markdown`文档格式规范。在提交代码时，使用`pre-commit`进行规范检测，确保提交的代码符合规范。

### 发布

版本发布使用[`Travis CI`](https://travis-ci.com/)进行，在提交代码时将安装`codecov`，执行规范检测，单元测试和代码代码覆盖率检测。发布时需要将代码合并到`master`分支，安装`@semantic-release/changelog` 、`@semantic-release/git`、`semantic-release`用于生成`CHANGELOG.md`代码变更记录，同时会执行单元测试，打包发布等流程。

## 代码结构

`dayjs`的特点就是小而全，它的代码结构也非常简单，主要包含以下几个部分：

```
src
  ├── constant.js // 常量定义
  ├── index.js    // 入口文件
  ├── locale      // 国际化配置
  ├── plugin      // 插件
  └── utils.js    // 工具函数
```

入口文件`index.js`前三行代码引入了`constant.js`、`locale/en.js`、`utils.js`，因此我们先看看这几个文件：

### 常量定义：constant.js

这个文件主要包含一些常量定义：

```js
// 单位转换的命名可以学习：
// 秒数转换
export const SECONDS_A_MINUTE = 60
export const SECONDS_A_HOUR = SECONDS_A_MINUTE * 60
export const SECONDS_A_DAY = SECONDS_A_HOUR * 24
export const SECONDS_A_WEEK = SECONDS_A_DAY * 7

// 毫秒数转换
export const MILLISECONDS_A_SECOND = 1e3
export const MILLISECONDS_A_MINUTE = SECONDS_A_MINUTE * MILLISECONDS_A_SECOND
export const MILLISECONDS_A_HOUR = SECONDS_A_HOUR * MILLISECONDS_A_SECOND
export const MILLISECONDS_A_DAY = SECONDS_A_DAY * MILLISECONDS_A_SECOND
export const MILLISECONDS_A_WEEK = SECONDS_A_WEEK * MILLISECONDS_A_SECOND

// 单位名称
export const MS = 'millisecond'
export const S = 'second'
export const MIN = 'minute'
export const H = 'hour'
export const D = 'day'
export const W = 'week'
export const M = 'month'
export const Q = 'quarter'
export const Y = 'year'
export const DATE = 'date'

// ISO 8601默认时间字符串：2021-05-18T13:56:28Z
export const FORMAT_DEFAULT = 'YYYY-MM-DDTHH:mm:ssZ'

// 无效时间返回值，new Date('Invalid Date')的返回值
export const INVALID_DATE_STRING = 'Invalid Date'

// 解析ISO 8601时间字符串，根据捕获的索引可以得到对应的值
export const REGEX_PARSE = /^(\d{4})[-/]?(\d{1,2})?[-/]?(\d{0,2})[^0-9]*(\d{1,2})?:?(\d{1,2})?:?(\d{1,2})?[.:]?(\d+)?$/
// 解析时间格式字符串，返回相应的时间格式
export const REGEX_FORMAT = /\[([^\]]+)]|Y{1,4}|M{1,4}|D{1,2}|d{1,4}|H{1,2}|h{1,2}|a|A|m{1,2}|s{1,2}|Z{1,2}|SSS/g
```

### 国际化配置：locale

#### en.js

由于默认使用`en`作为`locale`，原生API也是使用`en`输出，因此`en.js`只配置了部分参数：

```js
// English [en]
// We don't need weekdaysShort, weekdaysMin, monthsShort in en.js locale
export default {
  name: 'en',
  weekdays: 'Sunday_Monday_Tuesday_Wednesday_Thursday_Friday_Saturday'.split('_'),
  months: 'January_February_March_April_May_June_July_August_September_October_November_December'.split('_')
}
```

#### zh-cn.js

```js
// Chinese (China) [zh-cn]
import dayjs from 'dayjs'

const locale = {
  name: 'zh-cn',
  weekdays: '星期日_星期一_星期二_星期三_星期四_星期五_星期六'.split('_'),
  weekdaysShort: '周日_周一_周二_周三_周四_周五_周六'.split('_'),
  weekdaysMin: '日_一_二_三_四_五_六'.split('_'),
  months: '一月_二月_三月_四月_五月_六月_七月_八月_九月_十月_十一月_十二月'.split('_'),
  monthsShort: '1月_2月_3月_4月_5月_6月_7月_8月_9月_10月_11月_12月'.split('_'),
  ordinal: (number, period) => {
    switch (period) {
      case 'W':
        return `${number}周`
      default:
        return `${number}日`
    }
  },
  weekStart: 1,
  yearStart: 4,
  formats: {
    LT: 'HH:mm',
    LTS: 'HH:mm:ss',
    L: 'YYYY/MM/DD',
    LL: 'YYYY年M月D日',
    LLL: 'YYYY年M月D日Ah点mm分',
    LLLL: 'YYYY年M月D日ddddAh点mm分',
    l: 'YYYY/M/D',
    ll: 'YYYY年M月D日',
    lll: 'YYYY年M月D日 HH:mm',
    llll: 'YYYY年M月D日dddd HH:mm'
  },
  relativeTime: {
    future: '%s内',
    past: '%s前',
    s: '几秒',
    m: '1 分钟',
    mm: '%d 分钟',
    h: '1 小时',
    hh: '%d 小时',
    d: '1 天',
    dd: '%d 天',
    M: '1 个月',
    MM: '%d 个月',
    y: '1 年',
    yy: '%d 年'
  },
  meridiem: (hour, minute) => {
    const hm = (hour * 100) + minute
    if (hm < 600) {
      return '凌晨'
    } else if (hm < 900) {
      return '早上'
    } else if (hm < 1100) {
      return '上午'
    } else if (hm < 1300) {
      return '中午'
    } else if (hm < 1800) {
      return '下午'
    }
    return '晚上'
  }
}

dayjs.locale(locale, null, true)

export default locale
```


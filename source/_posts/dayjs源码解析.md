---
title: dayjs源码解析
date: 2021-05-18 09:52:14
tags: 源码解析
---

本文内容基于`day.js`的`v1.10.4`版本，将主要从基础理念、工程架构、源码解析、插件体系和国际化体系来解析dayjs源码。

对时间相关的概念和API不熟悉的推荐先阅读上一篇文章[时间](https://star.qingzz.cn/2021/05/14/shi-jian/)。

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

## 源码解析

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

// 解析ISO 8601时间字符串，根据match捕获的索引可以获取年月日时分秒毫秒等信息
export const REGEX_PARSE = /^(\d{4})[-/]?(\d{1,2})?[-/]?(\d{0,2})[^0-9]*(\d{1,2})?:?(\d{1,2})?:?(\d{1,2})?[.:]?(\d+)?$/
// 解析时间格式字符串，返回相应的时间格式
export const REGEX_FORMAT = /\[([^\]]+)]|Y{1,4}|M{1,4}|D{1,2}|d{1,4}|H{1,2}|h{1,2}|a|A|m{1,2}|s{1,2}|Z{1,2}|SSS/g
```

### 国际化配置：locale

#### 使用

`dayjs`会在打包的时候生成`locale.json`文件，存储语言包的`key`和`name`组成的数组。

注册使用语言包:

```
import * as dayjs from 'dayjs';
import 'dayjs/locale/zh-cn'; // 全局注册语言包

dayjs.locale('zh-cn'); // 全局启用
dayjs().locale('zh-cn').format(); // 当前实例启用
```

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
  // 语言名
  name: 'zh-cn',
  // 星期
  weekdays: '星期日_星期一_星期二_星期三_星期四_星期五_星期六'.split('_'),
  // 短的星期
  weekdaysShort: '周日_周一_周二_周三_周四_周五_周六'.split('_'),
  // 最短的星期
  weekdaysMin: '日_一_二_三_四_五_六'.split('_'),
  // 月份
  months: '一月_二月_三月_四月_五月_六月_七月_八月_九月_十月_十一月_十二月'.split('_'),
  // 短的月份
  monthsShort: '1月_2月_3月_4月_5月_6月_7月_8月_9月_10月_11月_12月'.split('_'),
  // 序号生成工厂函数
  ordinal: (number, period) => {
    switch (period) {
      case 'W':
        // W返回周序号
        return `${number}周`
      default:
        // 默认返回日序号
        return `${number}日`
    }
  },
  // 一周起始日，设置为1定义星期一是开始
  weekStart: 1,
  // 一年起始周，设置为4定义一月四号所在周是开始
  yearStart: 4,
  // 时间日期格式
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
  // 相对时间格式
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
  // 时间段
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

// 全局注册语言包
dayjs.locale(locale, null, true)

// 导出语言配置，用于启用
export default locale
```

#### 工具函数：utils.js

```js
// ES Module引入常量，短名减少代码量
import * as C from './constant'

/**
 * @description: 在 string 的开头填充 pad 字符，直到长度为 length，相当于`string.padStart(length, pad)`，空字符串也不做填充
 * @param {String} string 被填充的字符串
 * @param {Number} length 需要扩充到的长度
 * @param {String} pad 填充字符
 * @return {String} 填充后的字符串
 */
const padStart = (string, length, pad) => {
  // 只做了string的类型兼容判断，length，pad都没有，有风险
  // 可参考：https://github.com/lodash/lodash/blob/4.17.15/lodash.js#L14443
  // 但也有一个想法：判断是否够用就行，不用做到极致，否则也是一种浪费？
  const s = String(string)
  if (!s || s.length >= length) return string
  // Array((length + 1) - s.length).join(pad)
  // 可简化为pad.repeat(length - s.length)
  // dayjs打包直接就体现：从2.6KB转成了2.59KB
  return `${Array((length + 1) - s.length).join(pad)}${string}`
}

/**
 * @description: 将实例的UTC偏移量（分钟）转化成的 [+|-]HH:mm的格式
 * @param {Dayjs} instance Dayjs的实例
 * @return {String} UTC偏移量，格式：[+|-]HH:mm
 */
const padZoneStr = (instance) => {
  // 这里逻辑取反，而后面判断<=0为正，是否有必要呢？改为以下代码可读性感觉更好：
  // const minutes = instance.utcOffset()
  // const absMinutes = Math.abs(minutes)
  // const hourOffset = Math.floor(absMinutes / 60)
  // const minuteOffset = absMinutes % 60
  // return `${minutes >= 0 ? '+' : '-'}${padStart(hourOffset, 2, '0')}:${padStart(minuteOffset, 2, '0')}`
  const negMinutes = -instance.utcOffset()
  const minutes = Math.abs(negMinutes)
  const hourOffset = Math.floor(minutes / 60)
  const minuteOffset = minutes % 60
  return `${negMinutes <= 0 ? '+' : '-'}${padStart(hourOffset, 2, '0')}:${padStart(minuteOffset, 2, '0')}`
}

/**
 * @description: 返回两个Dayjs实例的月份差
 * @param {Dayjs} a Dayjs的实例
 * @param {Dayjs} b Dayjs的实例
 * @return {Number} 返回两个实例的月份差
 */
const monthDiff = (a, b) => {
  // 来自moment.js的函数，确保两者返回相同的结果
  // 使用简单的反向逻辑递归调用，大大降低代码逻辑复杂度，减少代码量
  // 但是另外一方面转成了-(b-a)，不知道为什么要做这个反转，同上面的padZoneStr
  // 关于monthDiff算法的讨论：https://stackoverflow.com/questions/2536379/difference-in-months-between-two-dates-in-javascript
  // moment讨论结果：https://github.com/moment/moment/pull/571
  // 以下解析按照不反转逻辑来解析即a-b
  // 算法主要分为两部分，两部分相加即可
  // 1. 不考虑日期，计算相差的月份数，即：总月差 = 年差值 * 12 + 月差值
  // 2. 计算日期差值转成的小数，这部分逻辑就是算法差异，dayjs和momentjs的逻辑如下：
  // 	2.1 获取锚点1 = b日期实例 + 总月差
  //  2.2 获取a日期与锚点1的差值 = a日期 - 锚点1
  //  2.3 判断a日期与锚点1的大小，即如果a日期小于锚点1，说明差距没这么大，需要减去多出来的部分，反之需要多加一部分
  //  2.4 根据2.3的判断生成锚点2 = b日期实例 + 总月差+/-1，如此a日期实例必然落在锚点1和锚点2之间
  //  2.5 获取锚点区间长度 = 锚点2 - 锚点1
  //  2.6 计算a日期占据比例 = 锚点区间占据长度（a日期与锚点1的差值） / 锚点区间长度
  // 3. 计算最终月差 = 第一步计算的总月差 + a日期占据比例
  
  // 正向逻辑写法：
  // if (a.date() < b.date()) return -monthDiff(b, a)
  // const wholeMonthDiff = ((a.year() - b.year()) * 12) + (a.month() - b.month())
  // const anchor1 = b.clone().add(wholeMonthDiff, 'month')
  // const l1 = a - anchor1
  // const anchor2 = b.clone().add(wholeMonthDiff + (l1 < 0 ? -1 : 1), 'month')
  // const l2 = Math.abs(anchor2 - anchor1);
  // return (wholeMonthDiff + l1 / l2) || 0
  
  // 原反向逻辑写法-(b - a)：
  if (a.date() < b.date()) return -monthDiff(b, a)
  const wholeMonthDiff = ((b.year() - a.year()) * 12) + (b.month() - a.month())
  const anchor = a.clone().add(wholeMonthDiff, C.M)
  const c = b - anchor < 0
  const anchor2 = a.clone().add(wholeMonthDiff + (c ? -1 : 1), C.M)
  return +(-(wholeMonthDiff + ((b - anchor) / (c ? (anchor - anchor2) :
    (anchor2 - anchor)))) || 0)
}

/**
 * @description: 向0取整
 * @param {Number} n 要取整的数
 * @return {Number} 返回取整后的数字
 */
const absFloor = n => (
  n < 0
  	// || 0 是为了确保不会出现-0，Math.ceil(-0.1) => -0
  	? Math.ceil(n) || 0
		: Math.floor(n)
)

/**
 * @description: 返回 u 对应的小写单数形式的单位，能自动适配标准格式和缩写格式
 * @param {String} u M(month) y(year) w(week) d(day) D(date) h(hour) m(minute) s(second) ms(millisecond) Q(quarter) 或 其他字符串
 * @return {String} u 对应的单位
 */
const prettyUnit = (u) => {
  const special = {
    M: C.M,
    y: C.Y,
    w: C.W,
    d: C.D,
    D: C.DATE,
    h: C.H,
    m: C.MIN,
    s: C.S,
    ms: C.MS,
    Q: C.Q
  }
  // 返回special定义的单位，或将自定义的单位转为小写并去除结尾s字符的单数形式的单位
  return special[u] || String(u || '').toLowerCase().replace(/s$/, '')
}

/**
 * @description: 判断是否为undefined
 * @param {Any} s
 * @return {Boolean} 返回是否为undefined：true/false
 */
const isUndefined = s => s === undefined

// index.js使用Utils空间，babel无法mangle
// 缩短为了极致优化大小，实际开发不太推荐这样做
export default {
  s: padStart,
  z: padZoneStr,
  m: monthDiff,
  a: absFloor,
  p: prettyUnit,
  u: isUndefined
}
```

### 入口文件：index.js

`index.js`是`dayjs`的核心，为了优化大小，作者在写代码的时候精简了部分参数，包括导入参数、工具函数等等。

为了方便阅读源码，我把有影响阅读的方法参数都重新补全。核心入口文件主要分为以下几部分：

- 引入常量、语言配置、工具函数模块
- 初始化配置语言环境为en，定义全局环境变量
- 定义使用全局变量的工具方法，并统一放到Utils命名空间
- 核心Dayjs类
- 工厂函数对象dayjs
- 为工厂函数对象添加原型方法和静态方法，挂载环境变量
- 返回工厂函数dayjs

```
// 引入常量、语言配置、工具函数模块
import * as CONSTANT from './constant'
import en from './locale/en'
import UTILS from './utils'

// 初始化配置语言环境为en，定义全局环境变量
// 存储当前语言环境
let LOCALE = 'en' // global locale
// 存储已导入的语言包配置
const LoadedLocales = {} // global loaded locale
// 将en语言包配置添加到环境变量对象进行存储
LoadedLocales[LOCALE] = en

/**
 * @description: 判断是否为Dayjs实例
 * @param {Any} d
 * @return {Boolean} 返回是否为Dayjs实例：true/false
 */
const isDayjs = d => d instanceof Dayjs

/**
 * @description 解析语言配置，导入并启用语言包
 * @param {String|Object} preset 语言包名称或语言包配置对象
 * @param {Object} object 语言包配置对象或null
 * @param {Boolean} isLocal 是否为本地语言
 * @return {String} 返回解析到的环境语言名称
 */
const parseLocale = (preset, object, isLocal) => {
  let locale
  // 不传参数则直接返回当前启用的语言包名
  if (!preset) return LOCALE
  if (typeof preset === 'string') {
    // 判断是否preset语言是否已导入到本地
    if (LoadedLocales[preset]) {
      // 已导入则保存解析语言为preset
      locale = preset
    }
    // 判断传入语言配置对象
    if (object) {
      // 传入则直接导入语言配置并保存语言名称为preset
      LoadedLocales[preset] = object
      locale = preset
    }
  } else {
    // 若preset是完整的语言配置对象，则获取语言名称，存储导入并保存解析语言名称
    const { name } = preset
    LoadedLocales[name] = preset
    locale = name
  }
  // 未传入isLocal或isLocal为false时，将启用语言包为环境语言
  if (!isLocal && locale) LOCALE = locale
  // 返回解析到的环境语言名称，未成功解析则返回当前环境语言
  return locale || (!isLocal && LOCALE)
}

/**
 * @description 工厂函数对象dayjs
 * @param {Dayjs|Date} date Dayjs对象实例或Date对象实例
 * @param {Object} c 配置对象
 * @return {Dayjs} 返回Dayjs实例对象
 */
const dayjs = function (date, c) {
  // 若传入Dayjs实例对象，则直接返回对象的拷贝
  if (isDayjs(date)) {
    return date.clone()
  }

  // 获取初始化构建参数，生成Dayjs实例对象
  const config = typeof c === 'object' ? c : {}
  config.date = date
  config.args = arguments
  return new Dayjs(config)
}

/**
 * @description 获取dayjs实例的环境配置结合Date实例对象生成新的Dayjs实例
 * @param {Date} date 日期实例对象
 * @param {Dayjs} instance Dayjs实例对象
 * @return {Dayjs} 返回新的Dayjs实例
 */
const wrapper = (date, instance) =>
  dayjs(date, {
    // 语言环境配置
    locale: instance.$LOCALE,
    // UTC配置
    utc: instance.$utc,
    // 时区配置
    timezone: instance.$timezone,
    // 偏移配置，未启用，忽略
    $offset: instance.$offset // todo: refactor; do not use this.$offset in you code
  })

// 定义使用全局变量的工具方法，并统一放到Utils命名空间
const Utils = UTILS // for plugin use
Utils.parseLocale = parseLocale
Utils.isDayjs = isDayjs
Utils.wrapper = wrapper

/**
 * @description 通过参数生成Date实例对象
 * @param {Object} config 时间参数，包含date：时间参数,utc：Boolean,是否启用UTC
 * @return {Date} 返回Date实例对象
 */
const parseDate = (config) => {
  const { date, utc } = config
  // date不能传入null，将返回Invalid Date
  if (date === null) return new Date(NaN) // null is invalid
  // 未传入date时，则使用当前时间
  if (Utils.isUndefined(date)) return new Date() // today
  // 确保数据不可变性，传入为Date实例时，重新通过new Date生成一个新的实例
  if (date instanceof Date) return new Date(date)
  // date是字符串且未使用z字符结尾
  if (typeof date === 'string' && !/Z$/i.test(date)) {
    // 使用正则捕获日期对应的每个值，如2021-05-19 17:16:15.123，d可得到：
    // [
    //   '2021-05-19 17:16:15.123', // 原始值,d[0]
    //   '2021', // 年,d[1]
    //   '05', // 月,d[2]
    //   '19', // 日,d[3]
    //   '17', // 时,d[4]
    //   '16', // 分,d[5]
    //   '15', // 秒,d[6]
    //   '123', // 毫秒,d[7]
    //   index: 0,
    //   input: '2021-05-19 17:16:15.123',
    //   groups: undefined
    // ]
    const d = date.match(CONSTANT.REGEX_PARSE)
    if (d) {
      // 需关注new Date时传入的月份应为monthIndex即月份的索引=月份值-1
      // 月份可不传，则匹配值为undefined,undefined - 1返回NaN，需要转为1月份即月份索引值为0
      const m = d[2] - 1 || 0
      // 毫秒可能未传入则匹配值为undefined，或者传入过长，因此取三位
      const ms = (d[7] || '0').substring(0, 3)
      // 由于至少匹配到年，因此除了年份值之外都需要做兼容处理：
      // 即往1月1日0时0分0秒0毫秒做兼容
      // 使用兼容性最强的初始化Date对象方式，即传入所有参数或传入时间戳，而不是使用字符串
      // utc用于判断是否是UTC时间
      if (utc) {
        // Date.UTC返回UTC模式的时间戳
        return new Date(Date.UTC(d[1], m, d[3]
          || 1, d[4] || 0, d[5] || 0, d[6] || 0, ms))
      }
      return new Date(d[1], m, d[3]
          || 1, d[4] || 0, d[5] || 0, d[6] || 0, ms)
    }
  }

  // 其他类型的数据将尝试使用Date解析，无法确保能解析成功：https://star.qingzz.cn/2021/05/14/shi-jian/#toc-heading-16
  return new Date(date) // everything else
}

// 核心Dayjs类
class Dayjs {
  /**
   * @description 构造函数
   * @param {Object} config locale,date,utc,timezone等参数
   */
  constructor(config) {
    // 初始化语言环境
    this.$LOCALE = parseLocale(config.locale, null, true)
    // 初始化时间参数
    this.parse(config) // for plugin
  }

  /**
   * @description 初始化时间相关参数
   * @param {Object} config 构造函数的参数对象
   */
  parse(config) {
    // 初始化时间对象，使用原生Date对象大大降低代码量和复杂度
    this.$date = parseDate(config)
    // 初始化时区配置
    this.$timezone = config.timezone || {}
    // 初始化计算各项时间参数
    this.init()
  }

  /**
   * @description 初始化年、月、日、星期、时、分、秒、毫秒属性
   */
  init() {
    const { $date } = this
    this.$year = $date.getFullYear()
    this.$Month = $date.getMonth()
    this.$Date = $date.getDate()
    this.$WeekDay = $date.getDay()
    this.$Hour = $date.getHours()
    this.$minute = $date.getMinutes()
    this.$second = $date.getSeconds()
    this.$millisecond = $date.getMilliseconds()
  }

  /**
   * @description 将工具函数通过方法懒挂载到实例上，而不是直接赋值到某个属性，简化实例对象，更轻量且能复用同一份方法，值得学习
   */
  $utils() {
    return Utils
  }

  /**
   * @description 判断是否合法时间
   * @return {Boolean} 返回是否时间是否合法，true/false
   */
  isValid() {
    // Date非法时，转为字符串为Invalid Date，可根据此进行判断
    // 可以直接用!==，不但减少了一次转换，而且实际上!==的性能比===性能好
    // 大家可以使用下面代码测试一下：
    // // ===
    // console.time();
    // console.log("1 === 1", 1 === 1); // 1 === 1 true
    // console.timeEnd(); // default: 5.81ms

    // console.time();
    // console.log("1 === '1'", 1 === '1'); // 1 === '1' false
    // console.timeEnd(); // default: 0.109ms

    // // !==
    // console.time();
    // console.log("1 !== 1", 1 !== 1); // 1 !== 1 false
    // console.timeEnd(); // default: 0.039ms

    // console.time();
    // console.log("1 !== '1'", 1 !== '1'); // 1 !== '1' true
    // console.timeEnd(); // default: 0.063ms
    
    // 应改为：return CONSTANT.INVALID_DATE_STRING !== this.$date.toString()
    return !(this.$date.toString() === CONSTANT.INVALID_DATE_STRING)
  }

  /**
   * @description 判断当前实例是否与另外一个实例在某个单位层级内相等
   * @param {Dayjs} that Dayjs比较实例
   * @param {String} units 单位层级字符串
   * @return {Boolean} 返回是否相等，true/false
   */
  isSame(that, units) {
    const other = dayjs(that)
    // 夹逼定理：
    return this.startOf(units) <= other && other <= this.endOf(units)
  }

  /**
   * 
   * @param {*} that 
   * @param {*} units 
   * @returns 
   */
  isAfter(that, units) {
    return dayjs(that) < this.startOf(units)
  }

  /**
   * 
   * @param {*} that 
   * @param {*} units 
   * @returns 
   */
  isBefore(that, units) {
    return this.endOf(units) < dayjs(that)
  }

  /**
   * 
   * @param {*} input 
   * @param {*} get 
   * @param {*} set 
   * @returns 
   */
  $generate(input, get, set) {
    if (Utils.isUndefined(input)) return this[get]
    return this.set(set, input)
  }

  /**
   * 
   * @returns 
   */
  unix() {
    return Math.floor(this.valueOf() / 1000)
  }

  /**
   * @description 返回时间值
   * @return {Number} 非法时间将返回NaN，合法时间则返回时间戳
   */
  valueOf() {
    // timezone(hour) * 60 * 60 * 1000 => ms
    return this.$date.getTime()
  }

  /**
   * @description 获取某个单位层级范围空间的起始值或结束值
   * @param {String} units 单位字符串
   * @param {Boolean} startOf 可选，true或不传使用开始值，false使用结束值
   * @return {Dayjs} 返回某个单位层级范围空间的起始值或结束值实例
   */
  startOf(units, startOf) { // startOf -> endOf
    const isStartOf = !Utils.isUndefined(startOf) ? startOf : true
    const unit = Utils.prettyUnit(units)
    /**
     * @description 
     * @param {Number} d 
     * @param {Number} m 
     * @return 
     */
    const instanceFactory = (d, m) => {
      const ins = Utils.wrapper(this.$utc ?
        Date.UTC(this.$year, m, d) : new Date(this.$year, m, d), this)
      return isStartOf ? ins : ins.endOf(CONSTANT.D)
    }
    /**
     * 
     * @param {*} method 
     * @param {*} slice 
     * @return 
     */
    const instanceFactorySet = (method, slice) => {
      // [时,分,秒,毫秒]起始区间
      const argumentStart = [0, 0, 0, 0]
      const argumentEnd = [23, 59, 59, 999]
      return Utils.wrapper(this.toDate()[method].apply(
        this.toDate('s'),
        (isStartOf ? argumentStart : argumentEnd).slice(slice)
      ), this)
    }
    const { $WeekDay, $Month, $Date } = this
    const utcPad = `set${this.$utc ? 'UTC' : ''}`
    switch (unit) {
      case CONSTANT.Y:
        return isStartOf ? instanceFactory(1, 0) :
          instanceFactory(31, 11)
      case CONSTANT.M:
        return isStartOf ? instanceFactory(1, $Month) :
          instanceFactory(0, $Month + 1)
      case CONSTANT.W: {
        const weekStart = this.$locale().weekStart || 0
        const gap = ($WeekDay < weekStart ? $WeekDay + 7 : $WeekDay) - weekStart
        return instanceFactory(isStartOf ? $Date - gap : $Date + (6 - gap), $Month)
      }
      case CONSTANT.D:
      case CONSTANT.DATE:
        return instanceFactorySet(`${utcPad}Hours`, 0)
      case CONSTANT.H:
        return instanceFactorySet(`${utcPad}Minutes`, 1)
      case CONSTANT.MIN:
        return instanceFactorySet(`${utcPad}Seconds`, 2)
      case CONSTANT.S:
        return instanceFactorySet(`${utcPad}Milliseconds`, 3)
      default:
        return this.clone()
    }
  }

  /**
   * @description 获取某个单位层级范围空间的结束值
   * @param {String} arg 单位字符串
   * @return {Dayjs} 返回某个单位层级范围空间的结束值实例
   */
  endOf(arg) {
    return this.startOf(arg, false)
  }

  /**
   * @description 私有方法，
   * @param {*} units 
   * @param {*} int 
   * @returns 
   */
  $set(units, int) { // private set
    const unit = Utils.prettyUnit(units)
    const utcPad = `set${this.$utc ? 'UTC' : ''}`
    const name = {
      [CONSTANT.D]: `${utcPad}Date`,
      [CONSTANT.DATE]: `${utcPad}Date`,
      [CONSTANT.M]: `${utcPad}Month`,
      [CONSTANT.Y]: `${utcPad}FullYear`,
      [CONSTANT.H]: `${utcPad}Hours`,
      [CONSTANT.MIN]: `${utcPad}Minutes`,
      [CONSTANT.S]: `${utcPad}Seconds`,
      [CONSTANT.MS]: `${utcPad}Milliseconds`
    }[unit]
    const arg = unit === CONSTANT.D ? this.$Date + (int - this.$WeekDay) : int

    if (unit === CONSTANT.M || unit === CONSTANT.Y) {
      // clone is for badMutable plugin
      const date = this.clone().set(CONSTANT.DATE, 1)
      date.$date[name](arg)
      date.init()
      this.$date = date.set(CONSTANT.DATE, Math.min(this.$Date, date.daysInMonth())).$date
    } else if (name) this.$date[name](arg)

    this.init()
    return this
  }

  set(string, int) {
    return this.clone().$set(string, int)
  }

  get(unit) {
    return this[Utils.prettyUnit(unit)]()
  }

  add(number, units) {
    number = Number(number) // eslint-disable-line no-param-reassign
    const unit = Utils.prettyUnit(units)
    const instanceFactorySet = (n) => {
      const d = dayjs(this)
      return Utils.wrapper(d.date(d.date() + Math.round(n * number)), this)
    }
    if (unit === CONSTANT.M) {
      return this.set(CONSTANT.M, this.$Month + number)
    }
    if (unit === CONSTANT.Y) {
      return this.set(CONSTANT.Y, this.$year + number)
    }
    if (unit === CONSTANT.D) {
      return instanceFactorySet(1)
    }
    if (unit === CONSTANT.W) {
      return instanceFactorySet(7)
    }
    const step = {
      [CONSTANT.MIN]: CONSTANT.MILLISECONDS_A_MINUTE,
      [CONSTANT.H]: CONSTANT.MILLISECONDS_A_HOUR,
      [CONSTANT.S]: CONSTANT.MILLISECONDS_A_SECOND
    }[unit] || 1 // ms

    const nextTimeStamp = this.$date.getTime() + (number * step)
    return Utils.wrapper(nextTimeStamp, this)
  }

  subtract(number, string) {
    return this.add(number * -1, string)
  }

  format(formatStr) {
    if (!this.isValid()) return CONSTANT.INVALID_DATE_STRING

    const str = formatStr || CONSTANT.FORMAT_DEFAULT
    const zoneStr = Utils.padZoneStr(this)
    const locale = this.$locale()
    const { $Hour, $minute, $Month } = this
    const {
      weekdays, months, meridiem
    } = locale
    const getShort = (arr, index, full, length) => (
      (arr && (arr[index] || arr(this, str))) || full[index].substr(0, length)
    )
    const get$Hour = num => (
      Utils.padStart($Hour % 12 || 12, num, '0')
    )

    const meridiemFunc = meridiem || ((hour, minute, isLowercase) => {
      const m = (hour < 12 ? 'AM' : 'PM')
      return isLowercase ? m.toLowerCase() : m
    })

    const matches = {
      YY: String(this.$year).slice(-2),
      YYYY: this.$year,
      M: $Month + 1,
      MM: Utils.padStart($Month + 1, 2, '0'),
      MMM: getShort(locale.monthsShort, $Month, months, 3),
      MMMM: getShort(months, $Month),
      D: this.$Date,
      DD: Utils.padStart(this.$Date, 2, '0'),
      d: String(this.$WeekDay),
      dd: getShort(locale.weekdaysMin, this.$WeekDay, weekdays, 2),
      ddd: getShort(locale.weekdaysShort, this.$WeekDay, weekdays, 3),
      dddd: weekdays[this.$WeekDay],
      H: String($Hour),
      HH: Utils.padStart($Hour, 2, '0'),
      h: get$Hour(1),
      hh: get$Hour(2),
      a: meridiemFunc($Hour, $minute, true),
      A: meridiemFunc($Hour, $minute, false),
      m: String($minute),
      mm: Utils.padStart($minute, 2, '0'),
      s: String(this.$second),
      ss: Utils.padStart(this.$second, 2, '0'),
      SSS: Utils.padStart(this.$millisecond, 3, '0'),
      Z: zoneStr // 'ZZ' logic below
    }

    return str.replace(CONSTANT.REGEX_FORMAT, (match, $1) => $1 || matches[match] || zoneStr.replace(':', '')) // 'ZZ'
  }

  utcOffset() {
    // Because a bug at FF24, we're rounding the timezone offset around 15 minutes
    // https://github.com/moment/moment/pull/1871
    return -Math.round(this.$date.getTimezoneOffset() / 15) * 15
  }

  diff(input, units, float) {
    const unit = Utils.prettyUnit(units)
    const that = dayjs(input)
    const zoneDelta = (that.utcOffset() - this.utcOffset()) * CONSTANT.MILLISECONDS_A_MINUTE
    const diff = this - that
    let result = Utils.monthDiff(this, that)

    result = {
      [CONSTANT.Y]: result / 12,
      [CONSTANT.M]: result,
      [CONSTANT.Q]: result / 3,
      [CONSTANT.W]: (diff - zoneDelta) / CONSTANT.MILLISECONDS_A_WEEK,
      [CONSTANT.D]: (diff - zoneDelta) / CONSTANT.MILLISECONDS_A_DAY,
      [CONSTANT.H]: diff / CONSTANT.MILLISECONDS_A_HOUR,
      [CONSTANT.MIN]: diff / CONSTANT.MILLISECONDS_A_MINUTE,
      [CONSTANT.S]: diff / CONSTANT.MILLISECONDS_A_SECOND
    }[unit] || diff // milliseconds

    return float ? result : Utils.absFloor(result)
  }

  /**
   * @description 
   * @return 
   */
  daysInMonth() {
    return this.endOf(CONSTANT.M).$Date
  }

  /**
   * @description 私有方法，用于获取当前语言环境配置对象
   * @return {Object} 返回当前语言环境配置对象
   */
  $locale() { // get locale object
    return LoadedLocales[this.$LOCALE]
  }

  /**
   * @description 获取语言环境或启用语言包
   * @param {Any} preset 语言环境字符串或语言配置对象
   * @param {Object} object 一般本地语言包会传入null来启用，也可以通过object直接将语言配置传入
   * @return {String|Dayjs} 不传参数则返回语言环境，传入参数则启用语言包并返回新Dayjs拷贝
   */
  locale(preset, object) {
    if (!preset) return this.$LOCALE
    // 保持不可变性，重新生成拷贝进行环境配置
    const that = this.clone()
    const nextLocaleName = parseLocale(preset, object, true)
    if (nextLocaleName) that.$LOCALE = nextLocaleName
    return that
  }

  /**
   * @description 生成拷贝
   * @return {Dayjs} 复制当前环境参数和Date对象生成一个新的实例
   */
  clone() {
    return Utils.wrapper(this.$date, this)
  }

  /**
   * @description 转化为Date对象
   * @return {Date} 返回Date对象
   */
  toDate() {
    // 体现不可变性，不返回this.$date，而是重新生成Date对象
    return new Date(this.valueOf())
  }

  /**
   * @description 返回JSON格式字符串
   * @return {String|Null} 非法返回null,合法返回类似"2021-05-19T09:47:35.970Z"格式的字符串
   */
  toJSON() {
    return this.isValid() ? this.toISOString() : null
  }

  /**
   * @description 转为ISO 8601格式字符串
   * @return {String} 非法报错，合法返回类似"2021-05-19T09:47:35.970Z"格式的字符串
   */
  toISOString() {
    // ie 8 return
    // new Dayjs(this.valueOf() + this.$date.getTimezoneOffset() * 60000)
    // .format('YYYY-MM-DDTHH:mm:ss.SSS[Z]')
    return this.$date.toISOString()
  }

  /**
   * @description 转为字符串
   * @return {String} 非法返回Invalid Date，合法返回类似"Wed, 19 May 2021 09:45:51 GMT"格式的字符串
   */
  toString() {
    return this.$date.toUTCString()
  }
}

// 为工厂函数对象添加原型方法和静态方法，挂载环境变量
const proto = Dayjs.prototype
dayjs.prototype = proto;
[
  ['$millisecond', CONSTANT.MS],
  ['$second', CONSTANT.S],
  ['$minute', CONSTANT.MIN],
  ['$Hour', CONSTANT.H],
  ['$WeekDay', CONSTANT.D],
  ['$Month', CONSTANT.M],
  ['$year', CONSTANT.Y],
  ['$Date', CONSTANT.DATE]
].forEach((g) => {
  proto[g[1]] = function (input) {
    return this.$generate(input, g[0], g[1])
  }
})

dayjs.extend = (plugin, option) => {
  if (!plugin.$install) { // install plugin only once
    plugin(option, Dayjs, dayjs)
    plugin.$install = true
  }
  return dayjs
}

dayjs.locale = parseLocale

dayjs.isDayjs = isDayjs

dayjs.unix = timestamp => (
  dayjs(timestamp * 1e3)
)

dayjs.en = LoadedLocales[LOCALE]
dayjs.LoadedLocales = LoadedLocales
dayjs.plugins = {}

// 返回工厂函数dayjs
export default dayjs
```








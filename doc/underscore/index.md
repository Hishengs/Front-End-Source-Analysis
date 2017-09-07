## 简介
源码版本：1.8.3

[underscore](http://underscorejs.org/) 源码结构主要由 `Collection`, `Array`, `Function`, `Object`, `Util` 五大主要部分组成。


## 初始化
以下是 underscore 初始化部分代码解析
```js
// Baseline setup
// --------------

// Establish the root object, `window` in the browser, or `exports` on the server.
// 依环境而定，浏览器 this === window，
// 在 Node 环境，直接使用时，this === global，作为模块被引用时，this === exports === module.exports
var root = this;

// Save the previous value of the `_` variable.
// 保存原有的 underscore 实例
var previousUnderscore = root._;

// Save bytes in the minified (but not gzipped) version:
// 为压缩考虑，减少字符
var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

// Create quick reference variables for speed access to core prototypes.
// 减少字节，同时也是为了加快访问速度
var
  push             = ArrayProto.push,
  slice            = ArrayProto.slice,
  toString         = ObjProto.toString,
  hasOwnProperty   = ObjProto.hasOwnProperty;

// All **ECMAScript 5** native function implementations that we hope to use
// are declared here.
// 保留一些需要用到的原生方法引用
var
  nativeIsArray      = Array.isArray,
  nativeKeys         = Object.keys,
  nativeBind         = FuncProto.bind,
  nativeCreate       = Object.create;

// Naked function reference for surrogate-prototype-swapping.
// 用于原型继承(当 Object.create 不可用时，见下面的 baseCreate 方法)
var Ctor = function(){};

// Create a safe reference to the Underscore object for use below.
// 作为构造函数实例化对象时，此时 this 代表即将实例化的对象
var _ = function(obj) {
  if (obj instanceof _) return obj; // 已存在实例化的对象，则直接返回
  if (!(this instanceof _)) return new _(obj); // 非正常实例化，例如 var _ = _()，此时 this = window，不满足条件
  this._wrapped = obj; // mark一下，这个什么用途还不清楚
};

// Export the Underscore object for **Node.js**, with
// backwards-compatibility for the old `require()` API. If we're in
// the browser, add `_` as a global object.
if (typeof exports !== 'undefined') { // Node 环境
  if (typeof module !== 'undefined' && module.exports) {
    exports = module.exports = _;
  }
  exports._ = _;
} else { // 浏览器环境
  root._ = _;
}

// Current version.
_.VERSION = '1.8.3';
```

## Common
接着是定义了一些通用的内部方法供 `Collection`, `Array`, `Object` 等其他模块调用。
```js
// note1: 以下定义的是一些通用的供内部调用的方法
// Internal function that returns an efficient (for current engines) version
// of the passed-in callback, to be repeatedly applied in other Underscore
// functions.
// 返回优化的回调函数
// 应该理解为返回指定 执行函数、上下文、参数个数 的函数
// func 回调函数
// context 调用的对象(上下文)
// argCount 参数个数
var optimizeCb = function(func, context, argCount) {
  // 这里为什么使用 void 0 而不是 undefined? ( void 0 === undefined )
  // 因为低版本的IE，undefined 不是保留字也不是关键字，可被作为变量名使用
  // 这里使用 void 0 能保证一定返回我们要的 undefined
  if (context === void 0) return func; // 未提供 context 则不作修改直接返回
  switch (argCount == null ? 3 : argCount) { // argCount 未提供时默认为3( tip: undefined == null )
    case 1: return function(value) {
      return func.call(context, value); // 相当于 func(value)，此时 this === context
    };
    case 2: return function(value, other) {
      return func.call(context, value, other);
    };
    case 3: return function(value, index, collection) {
      return func.call(context, value, index, collection);
    };
    case 4: return function(accumulator, value, index, collection) {
      return func.call(context, accumulator, value, index, collection);
    };
  }
  return function() {
    return func.apply(context, arguments); // 提供多少个参数就是多少
  };
};

// A mostly-internal function to generate callbacks that can be applied
// to each element in a collection, returning the desired result — either
// identity, an arbitrary callback, a property matcher, or a property accessor.
// 根据 value 的值和类型生成具体的回调函数
var cb = function(value, context, argCount) {
  if (value == null) return _.identity; // 返回默认的迭代器
  if (_.isFunction(value)) return optimizeCb(value, context, argCount);
  if (_.isObject(value)) return _.matcher(value); // 返回一个 _.isMatch 实例(检查目标对象是否拥有指定的属性)，此时 value 是属性对
  return _.property(value); // 返回 用于返回目标对象指定属性( value )的值 的函数
};

// 默认的迭代器
_.iteratee = function(value, context) {
  return cb(value, context, Infinity);
};

// An internal function for creating assigner functions.
// keysFunc 获取对象属性名的函数(返回属性名数组)
// undefinedOnly true 时则只在属性未定义才添加，false 时属性即使定义了也把它覆盖
// 该函数用于扩展目标对象(obj)的属性，可通过传入多个键值对对象的方式: assigner(obj,{name:'hisheng'},{sex:0}...)
var createAssigner = function(keysFunc, undefinedOnly) {
  return function(obj) {
    var length = arguments.length;
    if (length < 2 || obj == null) return obj;
    for (var index = 1; index < length; index++) { // index=1 => 除了 obj 之外的其他参数
      var source = arguments[index],
          keys = keysFunc(source), // 传入参数对象的键名数组
          l = keys.length;
      for (var i = 0; i < l; i++) {
        var key = keys[i];
        if (!undefinedOnly || obj[key] === void 0) obj[key] = source[key]; // obj 不存在该key则添加
      }
    }
    return obj;
  };
};

// An internal function for creating a new object that inherits from another.
// 基于原型构造对象，有两种方式
// 若 Object.create 可用，则直接使用其生成对象
// 否则通过 Ctor 来构造(传统的构造函数实例化对象)
var baseCreate = function(prototype) {
  if (!_.isObject(prototype)) return {};
  if (nativeCreate) return nativeCreate(prototype);
  Ctor.prototype = prototype;
  var result = new Ctor;
  Ctor.prototype = null;
  return result;
};

// 返回目标对象指定属性( value )的值
var property = function(key) {
  return function(obj) {
    return obj == null ? void 0 : obj[key];
  };
};

// Helper for collection methods to determine whether a collection
// should be iterated as an array or as an object
// Related: http://people.mozilla.org/~jorendorff/es6-draft.html#sec-tolength
// Avoids a very nasty iOS 8 JIT bug on ARM-64. #2094
var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1; // 数组下标最大值( JavaScript 能表示的最大的整数是 Math.pow(2, 53)-1 )
var getLength = property('length'); // 获取对象 length 属性值的函数
var isArrayLike = function(collection) { // 检测是否是 类数组对象(一般具有 length 属性的对象都可视为 类数组对象)
  var length = getLength(collection);
  return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
};
```

## Collection
从 `// Collection Functions` 注释开始，是集合相关方法的起点。

## Array
从 `// Array Functions` 注释开始，是数组相关方法的起点。

## Function
从 `// Function (ahem) Functions` 注释开始，是函数相关方法的起点。

## Object
从 `// Object Functions` 注释开始，是对象相关方法的起点。

## Util
从 `// Utility Functions` 注释开始，是实用函数的起点。

## 源码解析
[查看全部源码解析](./#/underscore/source)

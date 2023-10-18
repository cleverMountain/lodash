# lodash整体架构

1. 匿名函数自执行
```js
;(function() {

}.call(this))
``` 

2. 暴露lodash
```js
var runInContext = function runInContext(content) {
  // 自身是一个函数
  function lodash() {
    
  }
  // 在lodash上绑定属性
  lodash.after = after
  ...
  return lodash
}
// 得到lodash
var _ = runInContext();
if (typeof define == 'function' && typeof define.amd == 'object' && define.amd) {
  // amd(异步加载模块)，通过defin定义
  root._ = _;
  define(function() {
    return _;
  });
}
else if (freeModule) {
  // cjs,module.exports
  (freeModule.exports = _)._ = _;
  freeExports._ = _;
}
else {
  // 浏览器的es6环境，root为window
  root._ = _;
}
```

3. runInContext方法
- 3.1定义lodash,返回LodashWrapper实例
```js
function LodashWrapper(value, chainAll) {
  this.__wrapped__ = value; // 传入的值
  this.__actions__ = []; // 
  this.__chain__ = !!chainAll; // false 不支持链式调用 
  this.__index__ = 0;
  this.__values__ = undefined;
}
function runInContext(content) {
    function lodash(value) {

      return new LodashWrapper(value);
    }
  // 在lodash上绑定属性
  lodash.after = after
  ...
  return lodash
}
```
- 3.2 绑定LodashWrapper原型
```js
// 空函数
function baseLodash() {}
lodash.prototype = baseLodash.prototype;
lodash.prototype.constructor = lodash;
// 重新定义LodashWrapper原型对象
LodashWrapper.prototype = baseCreate(baseLodash.prototype);
// 在原型对象上在添加constructor，该构造函数又重新指回LodashWrapper
LodashWrapper.prototype.constructor = LodashWrapper;
var baseCreate = (function() {
  function object() {}
  return function(proto) {
    if (!isObject(proto)) {
      return {};
    }
    if (objectCreate) {
      // Object.create
      return objectCreate(proto);
    }
    object.prototype = proto;
    var result = new object;
    object.prototype = undefined;
    return result;
  };
}());
```

4. mixin函数
- 4.1  mixin(lodash, lodash),在lodash原型上添加lodash属性
```js
mixin(lodash, lodash)
function mixin(object, source, options) {
  // 所有方法的key及名字
  var props = keys(source),
      methodNames = baseFunctions(source, props);
  // 是否可链式调用
  var chain = !(isObject(options) && 'chain' in options) || !!options.chain,
      isFunc = isFunction(object);
  arrayEach(methodNames, function(methodName) {
    var func = source[methodName];
    object[methodName] = func;
    if (isFunc) {
      // 在原型上定义属性
      object.prototype[methodName] = function() {
        var chainAll = this.__chain__;
        if (chain || chainAll) {
          var result = object(this.__wrapped__),
              actions = result.__actions__ = copyArray(this.__actions__);
          actions.push({ 'func': func, 'args': arguments, 'thisArg': object });
          result.__chain__ = chainAll;
          return result;
        }
        return func.apply(object, arrayPush([this.value()], arguments));
      };
    }
  });
  return object;
}
```

# 函数部分
1. _.before(n, func)
- 创建一个调用func的函数，通过this绑定和创建函数的参数调用func，调用次数不超过 n 次。 之后再调用这个函数，将返回一次最后调用func的结果。
```js
function before(n, func) {
  var result;
  // 必须是函数
  if (typeof func != 'function') {
    throw new TypeError(FUNC_ERROR_TEXT);
  }
  // 次数
  n = toInteger(n);
  // 闭包，返回一个新函数，利用保存的次数n，每调用一次n就--,
  // 当n > 0时调用传入的方法，并返回值，当n < 0时不调用直接返回undefined
  return function() {
    if (--n > 0) {
      result = func.apply(this, arguments);
    }
    if (n <= 1) {
      func = undefined;
    }
    return result;
  };
}
```

2. _.after(n, func)
```js
function after(n, func) {
  if (typeof func != 'function') {
    throw new TypeError(FUNC_ERROR_TEXT);
  }
  n = toInteger(n);
  return function() {
    if (--n < 1) {
      return func.apply(this, arguments);
    }
  };
}
```

3. _.ary(func, [n=func.length])
- func.length是参数的长度
- 创建一个调用func的函数。调用func时最多接受 n个参数，忽略多出的参数。
```js
function ary(func, n, guard) {
  n = guard ? undefined : n;
  n = (func && n == null) ? func.length : n;
  return createWrap();
}
function createWrap () {
  result createHybrid.apply(undefined, newData)
  return setWrapToString(setter(result, newData), func, bitmask);
}
function createHybrid() {
  function wrapper() {

  }
  return wrapper
}
// 最后返回的是wrapper
function wrapper() {
  // 获取参数长度
  var length = arguments.length,
      args = Array(length), // 空数组
      index = length
  // args = JSON.parse(JSON.strgify(arguments))
  while (index--) {
    args[index] = arguments[index];
  }
  return fn.apply(thisBinding, args);
}

// 自己简单实现
// 创建一个函数，最多接受两个参数
function ary(func, n) {
  return function() {
   let length = arguments.length
       args = []     
   while(n--) {
    args[index] = arguments[index]
   }
   return func.apply(this, args)
  }
}
```
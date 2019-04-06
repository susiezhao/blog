---
title: Proxy VS Object.defineProperty
date: "2019-04-06T19:37:30.284Z"
description: 基于 Proxy 和 Object.defineProperty 两种方式的数据劫持原理分析.
---

## 基于数据劫持的双向数据绑定
常见的基于数据劫持的双向数据绑定的实现方式：ES5 提供的 `Object.defineProperty` 和 ES6 提供的 `Proxy` 。基于数据劫持的双向数据绑定的优势在于：1、无需显式调用；2、可精确得知变化数据。

### Object.defineProperty
`Object.defineProperty()` 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回这个对象。  
用法：`Object.defineProperty(obj, prop, descriptor) `
- `obj`: 要在其上定义属性的对象 
- `prop`: 要定义或修改的属性名称
- `descriptor`: 将被定义或修改的属性描述符

#### 属性描述符
对象里目前存在的属性描述符有两种主要形式：数据描述符和存取描述符。数据描述符是一个具有值的属性，该值可能是可写的也可能是不可写的；存取描述符是由 `getter - setter` 函数对描述的属性。`descriptor` 必须是这两种形式之一，不能同时是两者。

数据描述符和存取描述符公有的属性：
- `configurable`: 当且仅当该属性值为 `true` 时，该属性描述符才能够被修改，同时该属性也能从对应的对象上被删除。默认值为 `false`；在对象上直接定义的属性，该特性默认值为 `true`；
- `enumerable`: 当且仅当该属性值为 `true` 时，该属性才能出现在对象的枚举属性中。默认值为 `false`；在对象上直接定义的属性，该特性默认值为 `true`；

数据描述符独有的属性：
- `writable`: 当且仅当该属性值为 `true` 时，`value` 才能被赋值运算符修改。默认值为 `false`；在对象上直接定义的属性，该特性默认值为 `true`；
- `value`: 该属性对应的值。默认值为 `undefined`；

存取描述符独有的属性：
- `get`: 给属性提供 `getter` 方法，在读取属性时调用的方法。默认值为 `undefined`；
- `set`: 给属性提供 `setter` 方法，在写入属性时调用的函数。默认值为 `undefined`

注意：
- 调用 `Object.defineProperty()` 方法创建一个新属性时，如果不指定，则 `configurable` `writable` `enumerable` 特性的默认值都是 `false`。
- 当试图改变不可配置属性（除了`value` 和 `writable` 属性之外，且 `writable` 属性值只能改为 `false` ）的值时会抛出 `TypeError`。

```javascript
var obj = {
    a: 1,
    get d() {
        return this.a;
    },
    set d(value) {
        this.a = value;
    }
};
Object.defineProperty(obj, 'b', {});
Object.defineProperty(obj, 'c', {
    get: function() {
        console.log('getter:');
        return this.a;
    },
    set: function(value) {
        console.log('setter');
        this.a = value;
    }
});
var descriptor_a = Object.getOwnPropertyDescriptor(obj, 'a'); // { value: 1, configurable: true, enumerable: true, writable: true }
var descriptor_b = Object.getOwnPropertyDescriptor(obj, 'b'); // { value: undefined, configurable: false, enumerable: false, writable: false }
var descriptor_c = Object.getOwnPropertyDescriptor(obj, 'c'); // { configurable: false, enumerable: false, get: f, set: f }
var descriptor_d = Object.getOwnPropertyDescriptor(obj, 'c'); // { configurable: true, enumerable: true, get: f, set: f }
```

#### 数据劫持
`Object.defineProperty()` 主要利用存取属性描述符中的 `get` 和 `set` 属性来实现劫持一个对象的属性，在对象属性发生变化时进行特定的操作。数据劫持的应用例子如下：
```javascript
function Observer(obj, callback) {
  var observe = function(obj, path) {
    let type = Object.prototype.toString.call(obj);
    // Object 类型
    if (type === '[object Object]' || type === '[object Array]') {
      observeObject(obj, path);
      if (type === '[object Array]') {
        observeArrayPreparation(obj, path);
      }
    }
  };

  var observeObject = function(obj, path) {
    // for...in 可以遍历对象实力属性和原型属性
    for (let prop in obj)  {
      let value = obj[prop];
      let _path = path.slice();
      _path.push(prop);
      Object.defineProperty(obj, prop, {
        get: function() {
          return value;
        },
        set: function(newValue) {
          if (value === newValue) {
            return;
          }
          callback(_path, newValue, value);
          value = newValue;
        }
      });
      // 递归
      observe(value, _path);
    } 
  };

  var observeArrayPreparation = function(arr, path) {
    var _props = ['push', 'pop', 'unshift', 'shift', 'splice', 'sort', 'reverse']; // 改变原数组的操作
    var _newProto = Object.create(Array.prototype);
    _props.forEach((prop) => {
      Object.defineProperty(_newProto, prop, {
        value: function() {
          var _path = path.slice();
          _path.push(prop);
          callback(_path);
          Array.prototype[prop].apply(arr, arguments);
        }
      });
    });
    arr.__proto__ = _newProto;
  };

  observe(obj, []);
}

var obj1 = {
  a: 1,
  b: 2,
  c: {
    d: 3,
    e: 4,
  },
};

var obj2 = {
  a: 1,
  b: 2,
  c: [3, 4],
};

new Observer(obj1, (path, newVle, oldVle) => {
  console.log(`path: ${path}, newValue: ${newVle}, oldValue: ${oldVle}`)
});

new Observer(obj2, (path, newVle, oldVle) => {
  console.log(`path: ${path}, newValue: ${newVle}, oldValue: ${oldVle}`)
});
```
`Object.defineProperty()` 用作数据劫持时存在如下缺陷：
- 监听情况只限于对象属性的修改，如果对对象属性的增删，此时不是进行数据劫持；
- 该方法本身无法对数组对象进行监听，通过上述自定义数组对象进行处理后，数组中新增加的元素的修改仍然无法监听；


### Proxy
ES6 提供了 Proxy 构造函数，用来生成一个 Proxy 实例。其在目标对象之前架设一层拦截，外界对该对象的访问，都必须通过这层拦截，因此 Proxy 提供了一种机制，可以对外界的访问进行过滤和改写。Proxy 可以使用 `get` 和 `set` 来拦截对象属性的访问，`apply` 拦截函数的调用等多达13中拦截操作。  
用法：`var proxy = new Proxy(target, handler);`
- `target`: 表示所要拦截的目标对象，可以是 JavaScript 中任何合法的对象；
- `handler`: handler 参数也是一个对象，用来定制拦截行为；

```javascript
var obj = new Proxy({}, {
  get: function (target, key, receiver) {
    console.log(`getting ${key}!`);
    return Reflect.get(target, key, receiver);
  },
  set: function (target, key, value, receiver) {
    console.log(`setting ${key}!`);
    return Reflect.set(target, key, value, receiver);
  }
});

obj.count = 1; // setting count!
++obj.count;
// getting count!
// setting count!
// 2
```

#### 数据劫持

```javascript
/* maps observable propertires to a Set of observer functions, which use the property */
var observers = new WeakMap();

// contains the triggered observer functions, which should run soon
var queuedObservers = new Set();

// points to the currently running observer function, can be undefined
var currentObserver;

/*
*transforms an object into an observable by wrapping it into a proxy,
* it also adds a blank Map for property-observer pairs to be saved later
*/
function observable(obj) {
  observers.set(obj, new Map());
  return new Proxy(obj, { get, set });
}

// the exposed observe function, which defined which property of the object to be observed
function observe(fn) {
  queueObserver(fn);
}

function get(target, key, receiver) {
  const result = Reflect.get(target, key, receiver);
  if (currentObserver) {
    registerObserver(target, key, currentObserver);
    if (typeof result === 'object') {
      const observableResult = observable(result);
      Reflect.set(target, key, observableResult, receiver);
      return observableResult;
    }
  }
  return result;
}

function registerObserver(target, key, observer) {
  let observersForKey = observers.get(target).get(key);
  if (!observersForKey) {
    observersForKey = new Set();
    observers.get(target).set(key, observersForKey);
  }
  observersForKey.add(observer);
}

function set(target, key, value, receiver) {
  const observersForKey = observers.get(target).get(key);
  if (observersForKey) {
    observersForKey.forEach(queueObserver);
  }
  return Reflect.set(target, key, value, receiver);
}

/*
*Queued observers run asynchronously in one batch, which results in superior performance. *During *registration, the observers are synchronously added to the queuedObservers Set. A Set *cannot contain *duplicates, so enqueuing the same observer multiple times won't result in *multiple executions.
*/
function queueObserver(observer) {
  if (queuedObservers.size === 0) {
    Promise.resolve().then(runObservers);
  }
  queuedObservers.add(observer);
}
// execute the observe functions in batch
function runObservers() {
  try {
    queuedObservers.forEach((observer) => {
      currentObserver = observer;
      observer();
    });
  } finally {
    currentObserver = undefined;
    queuedObservers.clear();
  }
}

// Test example
var obj = {
  a: 1,
  b: {
    c: 1,
    d: [1, 2],
  },
  e: [1, 2, 3]
};
var proxy = observable(obj);
function print() {
  console.log(`监听属性发生变化:\n a: ${proxy.a}\n b: b.c:${proxy.b.c}, b.d:${proxy.b.d}\n e: ${proxy.e}`, );
}

observe(print);
```
## 优点 && 缺点
优点：Proxy 用于数据劫持是支持对象的扩展属性的数据劫持，因为 Proxy 是按照单个对象定义数据劫持的，所以对象动态添加的属性可以被数据劫持；而 ES5 提供的 `Object.defineProperty` 在进行数据劫持时是不支持扩展属性的劫持的，其每个属性的 `get` 和 `set` 的拦截操作都是必须预先定义才能实现拦截。例如 `Object.defineProperty` 对于数组的数据劫持必须通过重定义数组的方法才能实现，其本身是无法直接支持数组的劫持的；而 Proxy 本身直接支持数组的劫持；  

缺点: Proxy 的浏览器的支持不全，且不能通过 polyfill 实现；而 `Object.defineProperty` 的浏览器支持性很好，基本上主流的浏览器全部实现支持了。

## 参考
[Data Binding with ES6 Proxies](https://blog.risingstack.com/writing-a-javascript-framework-data-binding-es6-proxy/)  
[Data binding introduction](https://blog.risingstack.com/writing-a-javascript-framework-data-binding-dirty-checking/)   
[nx-js/observer-util](https://github.com/nx-js/observer-util)

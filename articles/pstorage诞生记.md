### github 地址

https://github.com/GreenPomelo/pstorage

### 背景

在[南京邮电大学小程序](https://github.com/GreenPomelo/Undergraduate)中，存在课程表、个人信息之类不经常更新的数据，这些数据我们通常是通过本地缓存的方式进行存储，并在特定的情况下才进行数据更新。由于小程序本身的异步缓存接口采用的是传统回调函数的形式，而且在一开始无需考虑异步和同步缓存的性能差异，我们都使用了同步的方法去处理本地数据存储：

```javascript
// In login page
wx.setStorageSync('User', userData);
// In profile page
const userData = wx.getStorageSync('User')
```



同时，由于团队内部大家的水平有高有低，偶尔会有刚入门的新同学加入，就会出现以下代码：

```javascript
if (wx.getStorageSync('User')) {
  // do something
  ...
  this.name = wx.getStorageSync('User').name;
}
```

很显然，这样的做法会造成缓存被频繁读取，并不是一种最佳实践。

并且，随着业务越来越复杂，我们需要缓存的数据越来越多，在业务代码里，我们写出了越来越多不可控制的存取缓存的代码，并且分散在各个页面，以至于很多时候，不知道究竟哪些数据在哪里被缓存了。另一方面，由于代码整体都采用的同步方式处理数据，当存储类似课表或者成绩这种体积偏大的数据的时候就会存在一定的性能问题。

所以综上所述，为了更加合理的管理缓存数据，并且在一定程度上提升性能，我们设计了一个通用的缓存管理方案— `pstorage `来提升应用的性能以及架构上的合理性。

###pstroage 介绍

在核心 `api` 设计上，`pstorage` 严格遵循 W3C 标准：

```typescript
interface Storage {
  readonly attribute unsigned long length;
  DOMString? key(unsigned long index);
  getter DOMString? getItem(DOMString key);
  setter void setItem(DOMString key, DOMString value);
  deleter void removeItem(DOMString key);
  void clear();
};
```

`storage` 实例的核心方法有 10 个，异步方法是：`getItem` /`setItem`/`removeItem`/`clear`/`getInfo`，对应的同步方法是 `getItemSync`/`setItemSync`/`removeItemSync`/`clearSync`/`getInfoSync`

#### 特点

\- 支持多端缓存器适配

\- 统一的存储数据管理

\- 运行时缓存，减小对 Storage 的读取压力

\- 掌握被存储数据的信息

\- 同时支持同步、异步写法

\- 兼容缓存器其他没有使用到的 api

#### 使用

基本使用方法：

```javascript
const Storage = require('pstorage');
const storage = new Storage({
  target: localStorage,
  keys: ['userInfo']
});
const userInfo = {
  name: 'Jack'
};
try {
  storage.setItemSync('userInfo', userInfo);
} catch (err) {
  console.log(err);
}
// 异步存储读取
storage
  .setItem('userInfo', userInfo)
  .then(() => {
  	storage.getItem('userInfo').then(result => {
      console.log(result);
    });
	});
  .catch(err => {
    console.log(err);
  });
```

注意：在使用 `pstorage` 的时候，需要先通过 `keys` 字段注册需要存储的变量，以此来管理需要使用的变量。

#### 多平台适配

由于未来的产品可能会进行多端投放，所以 `pstroage` 本身已经支持了多平台：

-  Web 容器内已经支持的缓存器有：`localStorage`, `sessionStorage`；

- 小程序容器下已经支持：微信小程序、阿里小程序、头条小程序；

- ReactNative 的容器下支持：`AsyncStorage`

同时，还支持适配器的写法，用来选择性重写缓存器的原生方法，具体使用方法如下：

```javascript
const Storage = require('pstorage');

const getItemAsync = function(getItem) {

  return function(key, callback, fallback) {

    try {

      const value = getItem(key);

      callback(value);

    } catch (err) {

      fallback(err);

    }

  };

};

const setItemAsync = function(setItem) {

  return function(key, data, callback, fallback) {

    setItem(key, data);

    callback();

  };

};

const storage = new Storage({

  target: localStorage,

  keys: ['userInfo'],

  adapters: {

    getItem: getItemAsync(localStorage.getItem),

    setItem: setItemAsync(localStorage.setItem),

    getItemSync: localStorage.getItem,

    setItemSync: localStorage.setItem

  }

});
```

#### 规划

- 初始化应用时候的同步读取缓存换用异步读取
- 增加代码测试
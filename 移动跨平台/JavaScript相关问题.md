# JavaScript相关问题

> **前言**
>
> 随着现在大前端趋势的发展，JavaScript渐渐成为了移动端开发人员需要掌握的一项技能，包括微信小程序和React Native等热门的跨平台技术都使用到了JavaScript，本文记录一下我学习JavaScript过程中遇到的一些问题。

[TOC]

## 1.this的指向问题

### 1.1 普通函数调用

**对于普通函数，函数中的`this`指向调用它的对象。**正因如此，随着调用函数对象的不同，this对象的指向也不同。下面展示几个例子：

**例子1**

```javascript
var person = {
  name: 'Curry',
  sayName: function() {
    console.log(this.name);
  }
};
person.sayName(); // 打印'Curry'
```

这里调用`sayName()`的是person对象，因此函数中的this指向person对象，会打印出person对象的name属性。

**例子2**

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  sayName: function() {
    console.log(this.name);
  }
};

var sayName = person.sayName; // 将函数引用存起来
sayName(); // 相当于window.sayName()，打印'Thompson'
```

这里调用`sayName()`的是window对象，因此函数中的this指向window对象。

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  sayName: function() {
    return function() {
      console.log(this.name);
    };
  }
};

person.sayName()();
```

this对象是在运行时基于函数的执行环境绑定的：在全局函数中，this等于window，而当函数被作为某个对象的方法调用时，this等于那个对象。不过，匿名函数的执行环境具有全局性，因为其this对象通常指向window。

### 1.2 箭头函数

箭头函数中的`this`是在定义函数的时候绑定，而不是在执行函数的时候绑定。

### 1.3 call、apply和bind

* **function.prototype.call()**

使用`call`方法可以改变函数内部的`this`指向。

```javascript
var person = {
  name: 'Curry',
  sayName: function() {
    console.log(this.name);
  }
};

var name = 'Thompson';
var sayName = person.sayName;
sayName.call(person); // 打印'Curry'
```



* **function.prototype.apply()**

`apply`方法的作用与`call`方法类似，也可以改变`this`的指向，唯一的区别就是，它的第二参数为一个数组。

* **function.prototype.bind()**

`bind`方法用于将函数体内的`this`绑定到某个对象，然后返回一个新函数。

### 1.4 常见场景this问题的解决方案

* 微信小程序中的this

对于回调函数中的this对象。对于普通函数，this指向调用时所在的对象（即window对象）。对于箭头函数，this指向定义生效时所在的对象。

* React Native



## 2.原型链


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

var sayName = person.sayName; // 将person.sayName赋给sayName
sayName(); // 相当于window.sayName()，打印'Thompson'
```

这里调用`sayName()`的是window对象，因此函数中的this指向window对象。

**例子3**

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

person.sayName()(); // 打印'Thompson'
```

这里person.sayName()放回的是一个函数，最后调用该函数的还是window，因此函数中的this指向window。

此外，sayName中返回的其实是一个匿名函数，我们需要记住一点，**匿名函数具有全局性，匿名函数中的this指向window（前提是非严格模式下）。**

### 1.2 构造函数调用

```javascript
function Person(name) {
  this.name = name;
  console.log(this); // window
}

var person = Person('Curry');
console.log(person.name); // undefined
console.log(window.name); // 打印Curry
```

这里没有使用new关键字，相当于window调用了Person()函数，因此函数中的this指向window，`this.name=name`的执行结果就是给windwo添加了一个name属性并赋值，故window.name会打印出"Curry"。

```javascript
function Person(name) {
  this.name = name;
  console.log(this); // Person {name:"Curry"}
}

var person = new Person('Curry');
console.log(person.name);
```

使用new关键字创建了一个实例对象，**this就指向创建的实例对象**。

### 1.3 箭头函数

ES6引入了箭头函数，让函数的书写变得简洁，同时也解决了this执行上下文带来的问题。与普通函数不同，箭头函数中的`this`是在定义函数的时候确定的，不是函数调用时确定的，**指向箭头函数所属的对象**，也可以说是继承了父级中的this。

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  sayName: function() {
    return () => console.log(this.name);
  }
};

person.sayName()();
```

这里使用普通函数中的例子3，将返回的function改为箭头函数，这里的this指向person，因此最后会打印出person中的name属性。

再看一个例子

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  // sayName: function() {
  //   setTimeout(function() {
  //     console.log(this.name); // this指向window
  //   });
  // },
  sayName: function() {
    setTimeout(() =>
      console.log(this.name)); // 改为箭头函数，this指向person对象
  }
};
person.sayName(); // 打印Curry
```

这里使用了箭头函数，原本setTimeout中匿名函数的this指向window，现在指向了person对象。

### 1.4 call、apply和bind

call、apply、bind的作用是可以改变this的指向，解决this指向可能导致的undefined问题。

* **function.prototype.call()**

`call()` 方法的第一个参数表示要绑定的this对象，此外还可以添加多个参数，表示要传递给被调用函数的参数。

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  sayName: function() {
    console.log(this.name);
  }
};

person.sayName.call(); // 打印Thompson
var sayName = person.sayName;
sayName.call(person); // 打印'Curry'
```

如果第一个参数不传，默认this指向window，因此`person.sayName.call()`就相当于`person.sayName.call(window)`，函数中的this指向window。而`sayName.call(person)`将this指向了person对象，因此会打印出person中的name。

* **function.prototype.apply()**

`apply`方法的作用与`call`方法类似，同样可以改变`this`的指向，唯一的区别就是，它的接收的第二个参数为一个数组，将要传递给被调用函数的参数放到数组中。

* **function.prototype.bind()**

`bind`方法用于将函数体内的`this`绑定到某个对象，然后返回一个新函数。与call和apply不同的是，bind不会立即调用函数，可以在需要的时候再调用。

```javascript
var name = 'Thompson';
var person = {
  name: 'Curry',
  sayName: function() {
    console.log(this.name);
  }
};

var sayName = person.sayName.bind(person);
sayName(); // 打印'Curry'
```

这里`person.sayName.bind(person)`将函数内的this指向了person对象，因此之后调用`sayName()`会打印出person中name属性。

### 1.5 常见场景this问题的解决方案

* 微信小程序中的this

在微信小程序中，我们使用wx.request请求网络数据，一般会在成功的回调函数中设置数据，如下所示：

```javascript
getData: function() {
  wx.request({
    url: 'url',
    success: function(res) {
      that.setData({
        name: res.name
      });
    }
  });
}
```

但是这样会报错**TypeError: Cannot read property 'setData' of undefined**，原因就是在success回调函数中的this不是指向当前Page对象，因此是找不到`setData`方法的。

解决方案：

1）将this赋值给一个临时变量

```javascript
getData: function() {
  var that = this;
  wx.request({
    url: 'url',
    success: function(res) {
      that.setData({
        name: res.name
      });
    }
  });
}
```

变量名可以随意取，在success回调函数中就可以调用that.setData()来设置数据了。

2）使用箭头函数

```javascript
getData: function() {
  wx.request({
    url: 'url',
    success: (res) => {
      this.setData({
        name: res.name
      });
    }
  });
}
```

由于箭头函数中的this指向声明函数时父一级的this，因此就可以调用this.setData()方法了。

3）使用bind

```javascript
getData: function() {
  wx.request({
    url: 'url',
    success: function(res) {
      this.setData({
        name: res.name
      });
    }.bind(this)
  });
}
```

* React Native绑定事件




## 2.作用域



## 3.原型链



## 4.参数传递问题

首先确定一下概念：

**值传递**（pass by value）是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

**引用传递**（pass by reference）是指在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

JavaScript中的数据类型可以分为以下两大类：

基本数据类型：Undefined、Null、Boolean、Number、String

引用类型：Object、Array、Date、RegExp、Function、......

首先来看基本数据类型的传递，以String 为例：

```java
var str = 'a';
var str1 = str;
str1 = 'b';
console.log(str); // 打印'a'
```

将str赋值给str1，从运行结果可以看出改变str1的值并不会改变str的值。

然后来看引用类型，以Object和Array为例：

```javascript
// Object
var obj = {
  'name': 'a'
};
var obj1 = obj;
obj1.name = 'b';
console.log(obj); // 打印{name:"b"}

// Array
var list = [1, 2, 3];
var list1 = list;
list1[0] = 4;
console.log(list); // 打印[4,2,3]
```

这种情况下可以看到原变量的值被修改了。

从以上两种情况下我们可能会得出结论：基本数据类型是值传递；引用类型是引用传递，这个结论是不正确的，虽然表面上看确实是这样，但其实**JavaScript只有值传递**，因为对于引用类型，在赋值时也是会复制出一个新变量，只不过这个新的变量和原变量指向的同一块内存地址，因此改变新变量的值也会影响到原变量，可以这样理解，对于引用类型，传递的值就是它的地址。
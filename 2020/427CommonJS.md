## commonJS规范

- 模块

  CommonJS中规定每个文件是一个模块。将一个JavaScript文件直接通过script标签插入页面中与封装成commonJS模块最大的不同在于，前者的顶层作用域是全局作用域，在进行变量及函数声明时会污染全局环境；而后者会形成一个属于模块自身的作用域，所有的变量及函数只有自己能访问，对外是不可见的。

- 导出

  导出是一个模块向外暴露自身的唯一方式。在commonJS中，通过module.exports可以导出模块的内容。

  ```js
  module.exports = {
      name: 'calculator',
      add: function(a, b) {
          return a + b;
      }
  }
  ```

  commonJS模块内部会有一个module对象用于存放当前模块的信息，可以理解成在每个模块的最开始定义了以下对象：

  ```js
  var module = {};
  module.exports = {};
  ```

  为了书写方便，commonJS也支持另一种简化的导出方式——直接使用exports。

  ```js
  exports.name = 'calculator';
  exports.add = function(a, b) {
      return a + b;
  }
  ```

  在实现效果上，这段代码和上述的module.exports没有任何不同。其内在机制是将exports指向了module.exports，而module.exports在初始化时是一个空对象。我们可以简单地理解为，commonJS在每个模块的首部默认添加了以下代码：

  ```js
  var module = {
      exports: {}
  };
  var exports = module.exports;
  ```

  因此，为exports.add赋值相当于在module.exports对象上添加了一个属性。

  在使用exports时要注意一个问题，即不要直接给exports赋值，否则会导致其失效，如：

  ```js
  exports = {
      add: 'calaulator'
  };
  ```

  上述代码中，由于对exports进行了赋值操作，使其指向了新的对象，module.exports却仍然是原来的空对象，因此name不会被导出。

  另一个需要注意的是module.exports和exports混用，因为会导致重新赋值。

- 导入

  在commonJS中使用require导入模块，模块名称是一个文件，使用其中的属性或方法需要下标访问：

  ```js
  // calculator.ts
  module.exports = {
      add: function(a, b) {
          return a + b;
      }
  };
  ```

  在需要引入时：

  ```js
  // index.ts
  const calculator = require('./calculator.ts');
  console.log(calculator.add(2, 3));  // 5
  ```

  

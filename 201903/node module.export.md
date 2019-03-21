# Node.js exports 和 module.export
> 其实，Module.exports才是真正的接口，exports只不过是它的一个辅助工具。　
> 最终返回给调用的是Module.exports而不是exports。

- module.exports 初始值为一个空对象 {}
- exports 是指向的 module.exports 的引用 `var exports = module.exports`
- require() 返回的是 module.exports 而不是 exports

## 修改引用属性
exports 是 module.exports 的一个引用
```js
// exports.js
module.exports.name = 'Lee';
exports.age = 18;
```

```js
// main.js
var man = require('./exports');

console.log(man.age);
console.log(man.name);

```
输出
18
Lee

因为指向同一个对象，所以可以给对象添加属性
但是如果给 exports 重新复制，则 exports 不再指向 module.exports，所以属性不会导出

## 修改 exports 引用的指向
```js
// exports.js
module.exports.name = 'Lee';
exports = {
    age: 18
};
```

输出：
undefined
Lee

exports 此时指向了一个新的对象，从而与 module.exports 脱离了联系，因此没有导出

## 修改 module.exports 引用的指向
所以，如果要修改引用指向的对象，要修改 module.exports
```js
// exports.js
module.exports = {
    age: 18
};
exports.name = 'Lee';
```
输出：
18
undefined

此时，因为 module.exports 指向了新对象，而 exports 仍然指向原来的对象 ({})，所以此时 exports.name = 'Lee' 没有效果。
需要改为：

```js
module.exports = {
    age: 18
};
module.exports.name = 'Lee';
```
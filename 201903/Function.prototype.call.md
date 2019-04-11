
```js
var toString = Function.call.bind(Object.prototype.toString);

function isObj(o) {
  return toString(o) === '[object Object]';
}
在这里，调用 toString(o) 相当于调用
object.prototype.toString.call(o)
```

```js
function Product(name, price) {
  this.name = name;
  this.price = price;
  this.func = function(){
    console.log(this.name)
  }
}
let p = new Product('a','b')
p.func()
var t = p.func
Function.prototype.call(t, p)
t.call(p)

let newf = p.func.bind(p)
 newf()
let b = Function.prototype.call.bind(p.func)
b(p)
p.func.call(p)
//var toStr = Function.prototype.call.bind( Product );

function Food(name, price) {
  Product.call(this, name, price);
  this.category = 'food';
}

console.log(new Food('cheese', 5).name);
// expected output: "cheese"
```

输出：
```
> "a"
> "a"
> "a"
> "a"
> "a"
> "cheese"
```
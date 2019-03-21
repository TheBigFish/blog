# es6 export 和 import
>export是es6引出的语法，用于导出模块中的变量，对象，函数，类。对应的导入关键字是import。

- export default在一个模块中只能有一个，当然也可以没有。export在一个模块中可以有多个
export 导出多个实体组成一个对象，对应的在 import 中也要有对象解析语法来解析实体名：

## 命名导出 export

```js
var name = lee；
var person=function(){
}

export {name，person};
export const a = 1；


import {name, person} from './text.js'
```

## 默认导出 export default
默认导出对象，外部引用时不需制定名称直接使用

- 默认导出（函数）：
`export default function() {}`
- 默认导出（类）：
`export default class {}`
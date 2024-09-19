## 前言

阿里三面的时候被问到了这个问题，当时思路虽然正确，可惜表述的不够清晰

后来花了一些时间整理了下思路，那么如何实现给所有的 async 函数添加 try/catch 呢？

## async 如果不加 try/catch 会发生什么事？

```js
// 示例
async function fn() {
  let value = await new Promise((resolve, reject) => {
    reject('failure')
  })
  console.log('do something...')
}
fn()
```

导致浏览器报错：一个未捕获的错误

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8c8226d96c4411697587d64a39bad00~tplv-k3u1fbpfcp-zoom-1.image)

在开发过程中，为了保证系统健壮性，或者是为了捕获异步的错误，需要频繁的在 async 函数中添加 try/catch，避免出现上述示例的情况

可是我很懒，不想一个个加，`懒惰使我们进步`😂

下面，通过手写一个 babel 插件，来给所有的 async 函数添加 try/catch

## babel 插件的最终效果

原始代码：

```js
async function fn() {
  await new Promise((resolve, reject) => reject('报错'))
  await new Promise(resolve => resolve(1))
  console.log('do something...')
}
fn()
```

使用插件转化后的代码：

```js
async function fn() {
  try {
    await new Promise((resolve, reject) => reject('报错'))
    await new Promise(resolve => resolve(1))
    console.log('do something...')
  } catch (e) {
    console.log('\nfilePath: E:\\myapp\\src\\main.js\nfuncName: fn\nError:', e)
  }
}
fn()
```

打印的报错信息：

![error.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa8c6a58a6dc43ac86349c159953e8b5~tplv-k3u1fbpfcp-watermark.image)

通过详细的报错信息，帮助我们快速找到目标文件和具体的报错方法，方便去定位问题

## babel 插件的实现思路

1）借助 AST 抽象语法树，遍历查找代码中的 await 关键字

2）找到 await 节点后，从父路径中查找声明的 async 函数，获取该函数的 body（函数中包含的代码）

3）创建 try/catch 语句，将原来 async 的 body 放入其中

4）最后将 async 的 body 替换成创建的 try/catch 语句

## babel 的核心：AST

先聊聊 AST 这个帅小伙 🤠，不然后面的开发流程走不下去

AST 是代码的树形结构，生成 AST 分为两个阶段：[**词法分析**](https://en.wikipedia.org/wiki/Lexical_analysis)和  [**语法分析**](https://en.wikipedia.org/wiki/Parsing)

**词法分析**

词法分析阶段把字符串形式的代码转换为**令牌（tokens）** ，可以把 tokens 看作是一个扁平的语法片段数组，描述了代码片段在整个代码中的位置和记录当前值的一些信息

比如`let a = 1`，对应的 AST 是这样的

![ast-a.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/253c13fe11a1447e9e2b4e7e6dd76a09~tplv-k3u1fbpfcp-watermark.image)

**语法分析**

语法分析阶段会把 token 转换成 AST 的形式，这个阶段会使用 token 中的信息把它们转换成一个 AST 的表述结构，使用 type 属性记录当前的类型

例如 let 代表着一个变量声明的关键字，所以它的 type 为 `VariableDeclaration`，而 a = 1 会作为 let 的声明描述，它的 type 为 `VariableDeclarator`

AST 在线查看工具：[**AST explorer**](https://astexplorer.net/)

**再举个 🌰，加深对 AST 的理解**

```js
function demo(n) {
  return n * n
}
```

转化成 AST 的结构

```js
{
  "type": "Program", // 整段代码的主体
  "body": [
    {
      "type": "FunctionDeclaration", // function 的类型叫函数声明；
      "id": { // id 为函数声明的 id
        "type": "Identifier", // 标识符 类型
        "name": "demo" // 标识符 具有名字
      },
      "expression": false,
      "generator": false,
      "async": false, // 代表是否 是 async function
      "params": [ // 同级 函数的参数
        {
          "type": "Identifier",// 参数类型也是 Identifier
          "name": "n"
        }
      ],
      "body": { // 函数体内容 整个格式呈现一种树的格式
        "type": "BlockStatement", // 整个函数体内容 为一个块状代码块类型
        "body": [
          {
            "type": "ReturnStatement", // return 类型
            "argument": {
              "type": "BinaryExpression",// BinaryExpression 二进制表达式类型
              "start": 30,
              "end": 35,
              "left": { // 分左 右 中 结构
                "type": "Identifier",
                "name": "n"
              },
              "operator": "*", // 属于操作符
              "right": {
                "type": "Identifier",
                "name": "n"
              }
            }
          }
        ]
      }
    }
  ],
  "sourceType": "module"
}
```

## 常用的 AST 节点类型对照表

| 类型原名称                | 中文名称         | 描述                                                  |
| ------------------------- | ---------------- | ----------------------------------------------------- |
| Program                   | 程序主体         | 整段代码的主体                                        |
| VariableDeclaration       | 变量声明         | 声明一个变量，例如 var let const                      |
| `FunctionDeclaration`     | 函数声明         | 声明一个函数，例如 function                           |
| ExpressionStatement       | 表达式语句       | 通常是调用一个函数，例如 console.log()                |
| BlockStatement            | 块语句           | 包裹在 {} 块内的代码，例如 if (condition){var a = 1;} |
| BreakStatement            | 中断语句         | 通常指 break                                          |
| ContinueStatement         | 持续语句         | 通常指 continue                                       |
| ReturnStatement           | 返回语句         | 通常指 return                                         |
| SwitchStatement           | Switch 语句      | 通常指 Switch Case 语句中的 Switch                    |
| IfStatement               | If 控制流语句    | 控制流语句，通常指 if(condition){}else{}              |
| Identifier                | 标识符           | 标识，例如声明变量时 var identi = 5 中的 identi       |
| CallExpression            | 调用表达式       | 通常指调用一个函数，例如 console.log()                |
| BinaryExpression          | 二进制表达式     | 通常指运算，例如 1+2                                  |
| MemberExpression          | 成员表达式       | 通常指调用对象的成员，例如 console 对象的 log 成员    |
| ArrayExpression           | 数组表达式       | 通常指一个数组，例如 [1, 3, 5]                        |
| `FunctionExpression`      | 函数表达式       | 例如 const func = function () {}                      |
| `ArrowFunctionExpression` | 箭头函数表达式   | 例如 const func = ()=> {}                             |
| `AwaitExpression`         | await 表达式     | 例如 let val = await f()                              |
| `ObjectMethod`            | 对象中定义的方法 | 例如 let obj = { fn () {} }                           |
| NewExpression             | New 表达式       | 通常指使用 New 关键词                                 |
| AssignmentExpression      | 赋值表达式       | 通常指将函数的返回值赋值给变量                        |
| UpdateExpression          | 更新表达式       | 通常指更新成员值，例如 i++                            |
| Literal                   | 字面量           | 字面量                                                |
| BooleanLiteral            | 布尔型字面量     | 布尔值，例如 true false                               |
| NumericLiteral            | 数字型字面量     | 数字，例如 100                                        |
| StringLiteral             | 字符型字面量     | 字符串，例如 vansenb                                  |
| SwitchCase                | Case 语句        | 通常指 Switch 语句中的 Case                           |

## await 节点对应的 AST 结构

1）原始代码

```js
async function fn() {
  await f()
}
```

对应的 AST 结构

![async.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/334884dd289b4a85ab9584465e459135~tplv-k3u1fbpfcp-watermark.image)

2）增加 try catch 后的代码

```js
async function fn() {
  try {
    await f()
  } catch (e) {
    console.log(e)
  }
}
```

对应的 AST 结构

![try.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f21f25a4b5e45ea8c0d6ad0d92640c9~tplv-k3u1fbpfcp-watermark.image)

**通过 AST 结构对比，插件的核心就是将原始函数的 body 放到 try 语句中**

## babel 插件开发

我曾在[《「历时 8 个月」10 万字前端知识体系总结（工程化篇）🔥》](https://juejin.cn/post/7146976516692410376#heading-34)中聊过如何开发一个 babel 插件

这里简单回顾一下

### 插件的基本格式示例

```js
module.exports = function (babel) {
   let t = babel.type
   return {
     visitor: {
       // 设置需要范围的节点类型
       CallExression: (path, state) => {
         do soming ……
       }
     }
   }
 }
```

1）通过 `babel` 拿到 `types` 对象，操作 AST 节点，比如创建、校验、转变等

2）`visitor`：定义了一个访问者，可以设置需要访问的节点类型，当访问到目标节点后，做相应的处理来实现插件的功能

### 寻找 await 节点

回到业务需求，现在需要找到 await 节点，可以通过`AwaitExpression`表达式获取

```js
module.exports = function (babel) {
  let t = babel.type
  return {
    visitor: {
      // 设置AwaitExpression
      AwaitExpression(path) {
        // 获取当前的await节点
        let node = path.node
      },
    },
  }
}
```

### 向上查找 async 函数

通过`findParent`方法，在父节点中搜寻 async 节点

```js
// async节点的属性为true
const asyncPath = path.findParent(p => p.node.async)
```

async 节点的 AST 结构

![asyncTrue.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ece3ff1773e94b839d8131e322dc4c3c~tplv-k3u1fbpfcp-watermark.image)

**这里要注意，async 函数分为 4 种情况：函数声明 、箭头函数 、函数表达式 、函数为对象的方法**

```js
// 1️⃣：函数声明
async function fn() {
  await f()
}

// 2️⃣：函数表达式
const fn = async function () {
  await f()
}

// 3️⃣：箭头函数
const fn = async () => {
  await f()
}

// 4️⃣：async函数定义在对象中
const obj = {
  async fn() {
    await f()
  },
}
```

需要对这几种情况进行分别判断

```js
module.exports = function (babel) {
  let t = babel.type
  return {
    visitor: {
      // 设置AwaitExpression
      AwaitExpression(path) {
        // 获取当前的await节点
        let node = path.node
        // 查找async函数的节点
        const asyncPath = path.findParent(
          p =>
            p.node.async &&
            (p.isFunctionDeclaration() ||
              p.isArrowFunctionExpression() ||
              p.isFunctionExpression() ||
              p.isObjectMethod())
        )
      },
    },
  }
}
```

### 利用 babel-template 生成 try/catch 节点

[babel-template](https://babel.docschina.org/docs/en/babel-template/)可以用以字符串形式的代码来构建 AST 树节点，快速优雅开发插件

```js
// 引入babel-template
const template = require('babel-template')

// 定义try/catch语句模板
let tryTemplate = `
try {
} catch (e) {
console.log(CatchError：e)
}`

// 创建模板
const temp = template(tryTemplate)

// 给模版增加key，添加console.log打印信息
let tempArgumentObj = {
  // 通过types.stringLiteral创建字符串字面量
  CatchError: types.stringLiteral('Error'),
}

// 通过temp创建try语句的AST节点
let tryNode = temp(tempArgumentObj)
```

### async 函数体替换成 try 语句

```js
module.exports = function (babel) {
  let t = babel.type
  return {
    visitor: {
      AwaitExpression(path) {
        let node = path.node
        const asyncPath = path.findParent(
          p =>
            p.node.async &&
            (p.isFunctionDeclaration() ||
              p.isArrowFunctionExpression() ||
              p.isFunctionExpression() ||
              p.isObjectMethod())
        )

        let tryNode = temp(tempArgumentObj)

        // 获取父节点的函数体body
        let info = asyncPath.node.body

        // 将函数体放到try语句的body中
        tryNode.block.body.push(...info.body)

        // 将父节点的body替换成新创建的try语句
        info.body = [tryNode]
      },
    },
  }
}
```

到这里，插件的基本结构已经成型，但还有点问题，如果函数已存在 try/catch，该怎么处理判断呢？

### 若函数已存在 try/catch，则不处理

```js
// 示例代码，不再添加try/catch
async function fn() {
  try {
    await f()
  } catch (e) {
    console.log(e)
  }
}
```

通过`isTryStatement`判断是否已存在 try 语句

```js
module.exports = function (babel) {
  let t = babel.type
  return {
    visitor: {
      AwaitExpression(path) {
        // 判断父路径中是否已存在try语句，若存在直接返回
        if (path.findParent(p => p.isTryStatement())) {
          return false
        }

        let node = path.node
        const asyncPath = path.findParent(
          p =>
            p.node.async &&
            (p.isFunctionDeclaration() ||
              p.isArrowFunctionExpression() ||
              p.isFunctionExpression() ||
              p.isObjectMethod())
        )
        let tryNode = temp(tempArgumentObj)
        let info = asyncPath.node.body
        tryNode.block.body.push(...info.body)
        info.body = [tryNode]
      },
    },
  }
}
```

### 添加报错信息

获取报错时的文件路径 `filePath` 和方法名称 `funcName`，方便快速定位问题

**获取文件路径**

```js
// 获取编译目标文件的路径，如：E:\myapp\src\App.vue
const filePath = this.filename || this.file.opts.filename || 'unknown'
```

**获取报错的方法名称**

```js
// 定义方法名
let asyncName = ''

// 获取async节点的type类型
let type = asyncPath.node.type

switch (type) {
  // 1️⃣函数表达式
  // 情况1：普通函数，如const func = async function () {}
  // 情况2：箭头函数，如const func = async () => {}
  case 'FunctionExpression':
  case 'ArrowFunctionExpression':
    // 使用path.getSibling(index)来获得同级的id路径
    let identifier = asyncPath.getSibling('id')
    // 获取func方法名
    asyncName = identifier && identifier.node ? identifier.node.name : ''
    break

  // 2️⃣函数声明，如async function fn2() {}
  case 'FunctionDeclaration':
    asyncName = (asyncPath.node.id && asyncPath.node.id.name) || ''
    break

  // 3️⃣async函数作为对象的方法，如vue项目中，在methods中定义的方法: methods: { async func() {} }
  case 'ObjectMethod':
    asyncName = asyncPath.node.key.name || ''
    break
}

// 若asyncName不存在，通过argument.callee获取当前执行函数的name
let funcName =
  asyncName || (node.argument.callee && node.argument.callee.name) || ''
```

### 添加用户选项

用户引入插件时，可以设置`exclude`、`include`、 `customLog`选项

`exclude`： 设置需要排除的文件，不对该文件进行处理

`include`： 设置需要处理的文件，只对该文件进行处理

`customLog`： 用户自定义的打印信息

### 最终代码

**入口文件 index.js**

```js
// babel-template 用于将字符串形式的代码来构建AST树节点
const template = require('babel-template')

const {
  tryTemplate,
  catchConsole,
  mergeOptions,
  matchesFile,
} = require('./util')

module.exports = function (babel) {
  // 通过babel 拿到 types 对象，操作 AST 节点，比如创建、校验、转变等
  let types = babel.types

  // visitor：插件核心对象，定义了插件的工作流程，属于访问者模式
  const visitor = {
    AwaitExpression(path) {
      // 通过this.opts 获取用户的配置
      if (this.opts && !typeof this.opts === 'object') {
        return console.error(
          '[babel-plugin-await-add-trycatch]: options need to be an object.'
        )
      }

      // 判断父路径中是否已存在try语句，若存在直接返回
      if (path.findParent(p => p.isTryStatement())) {
        return false
      }

      // 合并插件的选项
      const options = mergeOptions(this.opts)

      // 获取编译目标文件的路径，如：E:\myapp\src\App.vue
      const filePath = this.filename || this.file.opts.filename || 'unknown'

      // 在排除列表的文件不编译
      if (matchesFile(options.exclude, filePath)) {
        return
      }

      // 如果设置了include，只编译include中的文件
      if (options.include.length && !matchesFile(options.include, filePath)) {
        return
      }

      // 获取当前的await节点
      let node = path.node

      // 在父路径节点中查找声明 async 函数的节点
      // async 函数分为4种情况：函数声明 || 箭头函数 || 函数表达式 || 对象的方法
      const asyncPath = path.findParent(
        p =>
          p.node.async &&
          (p.isFunctionDeclaration() ||
            p.isArrowFunctionExpression() ||
            p.isFunctionExpression() ||
            p.isObjectMethod())
      )

      // 获取async的方法名
      let asyncName = ''

      let type = asyncPath.node.type

      switch (type) {
        // 1️⃣函数表达式
        // 情况1：普通函数，如const func = async function () {}
        // 情况2：箭头函数，如const func = async () => {}
        case 'FunctionExpression':
        case 'ArrowFunctionExpression':
          // 使用path.getSibling(index)来获得同级的id路径
          let identifier = asyncPath.getSibling('id')
          // 获取func方法名
          asyncName = identifier && identifier.node ? identifier.node.name : ''
          break

        // 2️⃣函数声明，如async function fn2() {}
        case 'FunctionDeclaration':
          asyncName = (asyncPath.node.id && asyncPath.node.id.name) || ''
          break

        // 3️⃣async函数作为对象的方法，如vue项目中，在methods中定义的方法: methods: { async func() {} }
        case 'ObjectMethod':
          asyncName = asyncPath.node.key.name || ''
          break
      }

      // 若asyncName不存在，通过argument.callee获取当前执行函数的name
      let funcName =
        asyncName || (node.argument.callee && node.argument.callee.name) || ''

      const temp = template(tryTemplate)

      // 给模版增加key，添加console.log打印信息
      let tempArgumentObj = {
        // 通过types.stringLiteral创建字符串字面量
        CatchError: types.stringLiteral(
          catchConsole(filePath, funcName, options.customLog)
        ),
      }

      // 通过temp创建try语句
      let tryNode = temp(tempArgumentObj)

      // 获取async节点(父节点)的函数体
      let info = asyncPath.node.body

      // 将父节点原来的函数体放到try语句中
      tryNode.block.body.push(...info.body)

      // 将父节点的内容替换成新创建的try语句
      info.body = [tryNode]
    },
  }
  return {
    name: 'babel-plugin-await-add-trycatch',
    visitor,
  }
}
```

**util.js**

```js
const merge = require('deepmerge')

// 定义try语句模板
let tryTemplate = `
try {
} catch (e) {
console.log(CatchError,e)
}`

/*
 * catch要打印的信息
 * @param {string} filePath - 当前执行文件的路径
 * @param {string} funcName - 当前执行方法的名称
 * @param {string} customLog - 用户自定义的打印信息
 */
let catchConsole = (filePath, funcName, customLog) => `
filePath: ${filePath}
funcName: ${funcName}
${customLog}:`

// 默认配置
const defaultOptions = {
  customLog: 'Error',
  exclude: ['node_modules'],
  include: [],
}

// 判断执行的file文件 是否在 exclude/include 选项内
function matchesFile(list, filename) {
  return list.find(name => name && filename.includes(name))
}

// 合并选项
function mergeOptions(options) {
  let { exclude, include } = options
  if (exclude) options.exclude = toArray(exclude)
  if (include) options.include = toArray(include)
  // 使用merge进行合并
  return merge.all([defaultOptions, options])
}

function toArray(value) {
  return Array.isArray(value) ? value : [value]
}

module.exports = {
  tryTemplate,
  catchConsole,
  defaultOptions,
  mergeOptions,
  matchesFile,
  toArray,
}
```

[github 仓库](https://github.com/xy-sea/babel-plugin-await-add-trycatch)

## babel 插件的安装使用

npm 网站搜索`babel-plugin-await-add-trycatch`

![npm.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7bd639f844243af89fb5e877c05b49c~tplv-k3u1fbpfcp-watermark.image)

有兴趣的朋友可以下载玩一玩

[babel-plugin-await-add-trycatch](https://www.npmjs.com/package/babel-plugin-await-add-trycatch)

## 总结

通过开发这个插件，了解很多 AST 方面的知识，掌握了一些 babel 的原理

友情提醒：不要在生产环境中使用该插件，该插件的功能还太过简陋。实际开发中，大家可以结合具体的业务需求开发自己的插件，一起动手玩一玩 babel

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd0e972f1d4a4f819156e607255451e7~tplv-k3u1fbpfcp-watermark.image" alt="nice.gif" width="30%" />

**参考资料**
[Babel 插件手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)
[嘿，不要给 async 函数写那么多 try/catch 了](https://juejin.cn/post/6844903886898069511)

## 10w 字笔记推荐

[「历时 8 个月」10 万字前端知识体系总结（基础知识篇）🔥](https://juejin.cn/post/7146973901166215176)

[「历时 8 个月」10 万字前端知识体系总结（算法篇）🔥](https://juejin.cn/post/7146975493278367752)

[「历时 8 个月」10 万字前端知识体系总结（工程化篇）🔥](https://juejin.cn/post/7146976516692410376)

[「历时 8 个月」10 万字前端知识体系总结（前端框架+浏览器原理篇）🔥](https://juejin.cn/post/7146996646394462239)

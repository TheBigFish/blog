# koa-websocket 同时使用 http 和 websocket

## 单文件使用

- 使用 websockify 装饰 Koa app 实例
- 同时创建 http 和 ws 的路由实例
- 使用 qpp.use 注册两个路由实例

程序如下：

```js
const Koa = require('koa')
const websockify = require('koa-websocket')
const router = require('koa-router')

const http = router()
const ws = router()

// the magic happens right here
const app = websockify(new Koa())

http.get('/', (ctx) => {
    ctx.status = 200
    ctx.body = 'Hello!'
})

http.get('/h1', (ctx) => {
    ctx.status = 200
    ctx.body = 'Hello h1!'
})


ws.get('/a1', (ctx) => {
    // `ctx` is the regular koa context created
    // from the `ws` onConnection `socket.upgradeReq` object.
    // the websocket is added to the context on `ctx.websocket`.
    ctx.websocket.send('Hello a1')
    ctx.websocket.on('message', (message) => {
        // do something with the message from client
        ctx.websocket.send(message)
    })
})

ws.get('/a2', (ctx) => {
    ctx.websocket.send('Hello a2')
    ctx.websocket.on('message', (message) => {
        // do something with the message from client
        ctx.websocket.send(message)
    })
})


app.use(http.routes()).use(http.allowedMethods())
app.ws.use(ws.routes()).use(ws.allowedMethods())


app.listen(3000)
```

## 模块化使用 1

routes/ws.js

```js
const koaRouter = require('koa-router');
const router = koaRouter();

function websocket(app) {
  console.log('load websocket...')

  app.ws.use(function (ctx, next) {
    return next(ctx);
  });

  router.get('/a1', async function (ctx) {
    ctx.websocket.send('123');
  })

  app.ws.use(router.routes())
  app.ws.use(router.allowedMethods());
}


module.exports = websocket
```

app.js

```js
const websockify = require('koa-websocket');
const app = websockify(new Koa())
const ws = require('./routes/ws')
...

ws(app)

```

## 模块化使用 2

routes/ws.js

```js
const router = require('koa-router')()

router.prefix('/ws')

router.get('/a1', function (ctx, next) {
    ctx.websocket.send('Hello a1');
})

router.get('/a2', function (ctx, next) {
    ctx.websocket.send('Hello a2');
})

module.exports = router
```

routes/http.js

```js
const router = require('koa-router')()

router.get('/', async (ctx, next) => {
  await ctx.render('index', {
    title: 'Hello Koa 2!'
  })
})

router.get('/string', async (ctx, next) => {
  ctx.body = 'koa2 string'
})

router.get('/json', async (ctx, next) => {
  ctx.body = {
    title: 'koa2 json'
  }
})

module.exports = router
```

```js
const websockify = require('koa-websocket');
const app = websockify(new Koa())

const index = require('./routes/index')
const ws = require('./routes/ws')

...

app.ws.use(ws.routes(), ws.allowedMethods())
app.use(index.routes(), index.allowedMethods())
```

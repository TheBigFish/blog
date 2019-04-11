# log4js 配置

```js
{
    "appenders": {
      "access": {
        "type": "dateFile",
        "filename": "log/access.log",
        "pattern": "-yyyy-MM-dd",
        "category": "http"
      },
      "app": {
        "type": "file",
        "filename": "log/app.log",
        "maxLogSize": 10485760,
        "numBackups": 3
      },
      "errorFile": {
        "type": "file",
        "filename": "log/errors.log"
      },
      "errors": {
        "type": "logLevelFilter",
        "level": "ERROR",
        "appender": "errorFile"
      }
    },
    "categories": {
      "default": { "appenders": [ "app", "errors" ], "level": "DEBUG" },
      "http": { "appenders": [ "access"], "level": "DEBUG" }
    }
  }
```

## 配置解析
appenders 配置日志输出归属，可以是控制台、文件、日期文件等等。每一个条目代表一个输出目的地。
categories 配置日志输出类，每一类可配置多个 appender

以上述配置为例:

appenders 设置了三个日志输出目的，access、 app、 errorFile。 其中 errors 是一个 logLevelFilter 的过滤器，该过滤器设置当日志级别 >= ERROR 的日志会发送给 errorFile.

categories 设置了两个日志输出类。
http 当level级别>=DEBUG触发，发送给 access
default 默认输出类，当无法匹配时输出到默认类。定义了两个输出目的地：app、 errors。
级别 >= DEBUG 都会发送，app 记录到 log/app.log. 溶蚀 errors 是一个日志级别过滤器，当日志级别在 >= ERROR 时候，发送给 errorFile, 记录到 log/errors.log


## 程序使用
www.js 启动程序

```js
var log4js = require('log4js');
log4js.configure('./config/log4js.json');

var log = log4js.getLogger("startup");
log.info("start")
```

categories 中并没有配置 startup, 所以会使用默认的 default输出

`[2019-04-11T10:27:58.823] [INFO] startup - start`


routes/index.js
```js
var express = require('express');
var router = express.Router();
var log = require('log4js').getLogger("index");

/* GET home page. */
router.get('/', function(req, res) {
  log.debug("This is in the index module");
  res.render('index', { title: 'Express' });
});

module.exports = router;

```

categories 中并没有配置 index, 所以会使用默认的 default输出

`[2019-04-11T10:30:08.041] [DEBUG] index - This is in the index module`


app.js

```js
var log = log4js.getLogger("app");
app.use(log4js.connectLogger(log4js.getLogger("http"), { level: 'auto' }));

if (app.get('env') === 'development') {
    app.use(function(err, req, res, next) {
        log.error("Something went wrong:", err);
        res.status(err.status || 500);
        res.render('error', {
            message: err.message,
            error: err
        });
    });
}

// production error handler
// no stacktraces leaked to user
app.use(function(err, req, res, next) {
    log.error("Something went wrong:", err);
    res.status(err.status || 500);
    res.render('error', {
        message: err.message,
        error: {}
    });
});
```


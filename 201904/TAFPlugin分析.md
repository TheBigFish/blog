#TAFPLugin 分析

## adapter
adapter.js 负责建立QWebChannel通道

init 返回一个 promise 
新建一个 websocket 连接。 当连接建立时，将 channel 作为全局变量保存。并且 resolve。
此时外部调用只需要 

`
this.channel = await Adapter.init(url)
`

## Result
封装返回结果类


## TAFDevice.js
设备类的父类。

### connect

connect 方法用来连接 TAF 设备的信号槽。
TAF 设备单个设备只有一个信号：sigDeviceCallBack，所有的通知类信号都是通过该信号来进行传递。
将该信号连接到函数 this.logicObj.devCallBack 处理。
先调用 this.convertArrayToObj(ret)，将 args 数组转换为对象，再调用  devCallBack.

```js
/**
* 建立信号槽连接
*/
connect () {
    if (this.logicObj.obj) {
      this.logicObj.obj.sigDeviceCallBack.connect((ret) => this.logicObj.devCallBack(this.convertArrayToObj(ret)))
    } else {
      throw new Error('设备不存在')
    }
}
```

devCallBack 在执行指令时已经注册。
此时解析出来的对象通过ret传递进来，判断 ret.evtID 为 cmd.DAM_EVENT_IDC_COMPLETE, 指令已经完成，resolve 这个 promise, 至此，读卡结束，外部获得卡片信息。

```js
/**
 * 允许进卡
 * mode 1:仅读磁道数据, 2:读磁道+IC卡数据
 */
  read (options = {}) {
    const {
      mode = this.TRACKS_ANYCARD,
      openLight = true
    } = options
    this.isOpenLight = openLight
    this.openLight()
    log.d('CardReader read is called')
    log.d('parameter mode: ', mode)

    return new Promise((resolve, reject) => {
      let callBack = (ret) => {
        log.d('CardReader read callBack')
        let result = {
          cmd: ret.cmd,
          code: ret.code,
          msg: ret.code === constant.API_SUCCESS ? constant.API_SUCCESS_MSG : constant.API_FAIL_MSG,
          data: ret.data
        }
        this.closeLight()
        switch (ret.evtID) {
          case cmd.DAM_EVENT_IDC_CARDINSERT: // 卡进入事件
            return this.emit(constant.CARD_INSERT, new Result(result))
          case cmd.DAM_EVENT_IDC_COMPLETE: // 指令已完成
            return resolve(new Result(result))
          case cmd.DAM_EVENT_IDC_CARDTAKEN: // 卡片被取走
            return this.emit(constant.CARD_REJECT, new Result(result))
          default:
            log.e('未知指令:', ret.evtID)
        }
      }

      const args = JSON.stringify({
        TRACKS: mode.toString()
      })

      this.logicObj.devCallBack = callBack
      this.exec(cmd.DAM_CMD_IDC_READ_RAW_DATA, args)
    })
  }
```
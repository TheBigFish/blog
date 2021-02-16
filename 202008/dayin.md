# 添加打印机

## 安装驱动

```bash
./printer_driver_setup.sh
```

## 添加打印机

- 控制面板  
  双击打印机
  ![控制面板](control_panel.png)

- 添加  
  点击 添加  
  ![添加](add.png)
markd
- 选择打印机
  - 点击 `输入uri`
  - 输入设备URI：  
    输入 `parallel:/dev/usb/lp0`  
    点击 前进

  ![添加打印机](add_pointer.png)

- 选择驱动程序  
  - 选择 GWI
  - 点击 前进
  ![选择驱动程序](select_driver.png)

- 确认
  - 选择 前进
  ![前进](forward.png)

- 修改打印机名称
  - 打印机名称改为：`打印机-并口`
  - 点击 应用
  ![修改名称](change_name.png)

- 关闭
  - 关闭弹出窗口
  ![关闭](complete.png)

- 完成
  ![完成](finish.png)

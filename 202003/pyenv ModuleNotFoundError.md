<!-- en_title: pyenv ModuleNotFoundError -->

# pyenv 找不到系统包问题分析

pyenv import 时找不到包的问题解决

<!-- more -->

## 使用系统python环境
```shell
bigfish@bigfish:~$ python3
Python 3.6.9 (default, Nov  7 2019, 10:44:02)
[GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> import Crypto
>>> sys.path
['', '/usr/lib/python36.zip', '/usr/lib/python3.6', '/usr/lib/python3.6/lib-dynload', '/usr/local/lib/python3.6/dist-packages', '/usr/lib/python3/dist-packages']
>>> Crypto.__file__
'/usr/lib/python3/dist-packages/Crypto/__init__.py'
```

此时，sys.path 中包含路径 `/usr/lib/python3/dist-packages` 所以能够找到包 Crypto


## 使用 pyenv virtualenv 环境

```bash

(venv3.6.9) bigfish@bigfish:~$ python
Python 3.6.9 (default, Mar 13 2020, 14:50:16)
[GCC 7.5.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import sys
>>> sys.path
['', '/home/bigfish/.pyenv/versions/3.6.9/lib/python36.zip', '/home/bigfish/.pyenv/versions/3.6.9/lib/python3.6', '/home/bigfish/.pyenv/versions/3.6.9/lib/python3.6/lib-dynload', '/home/bigfish/.pyenv/versions/venv3.6.9/lib/python3.6/site-packages']
>>> import Crypto
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ModuleNotFoundError: No module named 'Crypto'
```

可以看到, `/usr/lib/python3/dist-packages` 不在 path 中，所以找不到包 Crypto


### 导出 PYTHONPATH
```bash
(venv3.6.9) bigfish@bigfish:~$ export PYTHONPATH=/usr/lib/python3/dist-packages/
(venv3.6.9) bigfish@bigfish:~$ echo $PYTHONPATH
/usr/lib/python3/dist-packages/
(venv3.6.9) bigfish@bigfish:~$ python
Python 3.6.9 (default, Mar 13 2020, 14:50:16)
[GCC 7.5.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import Crypto
>>> Crypto.__file__
'/usr/lib/python3/dist-packages/Crypto/__init__.py'
```
此时能够正确的找到包 Crypto

## `import gi` 错误

使用 export 导出后，在 *3.7.4 虚拟环境中*， gi 能找到包，但是对应的 `_gi` 找不到
```
(test-env) bigfish@bigfish:~/work/test$ python
Python 3.7.4 (default, Mar  6 2020, 17:10:56)
[GCC 7.5.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import gi
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python3/dist-packages/gi/__init__.py", line 42, in <module>
    from . import _gi
ImportError: cannot import name '_gi' from 'gi' (/usr/lib/python3/dist-packages/gi/__init__.py)
```

查找 `_gi`
```bash
(venv3.6.9) bigfish@bigfish:~/work/trust_app$ sudo find / -name _gi*
[sudo] password for bigfish:
find: ‘/run/user/1000/gvfs’: Permission denied
/usr/lib/python3/dist-packages/gi/_gi_cairo.cpython-36m-x86_64-linux-gnu.so
/usr/lib/python3/dist-packages/gi/_gi.cpython-36m-x86_64-linux-gnu.so
/usr/lib/python2.7/dist-packages/gi/_gi.x86_64-linux-gnu.so
/snap/gnome-3-28-1804/116/usr/lib/python3/dist-packages/gi/_gi.cpython-36m-x86_64-linux-gnu.so
/snap/gnome-3-28-1804/116/usr/lib/python3/dist-packages/gi/_gi_cairo.cpython-36m-x86_64-linux-gnu.so
/home/bigfish/.pyenv/versions/3.7.4/envs/test-env/lib/python3.7/site-packages/PySide2/_git_pyside_version.py
```

可以在 `/usr/lib/python3/dist-packages/gi` 下找到 `_gi.cpython-36m-x86_64-linux-gnu.so` 文件，初步判断 3.7.4 版本
无法载入 36m 的包。

重新建立一个 3.6.9 环境，export  PYTHONPATH 后 `import gi`, 成功

## 补充
python3-gi 是一个跟系统相关的包  
在 ubuntu 18.04 版本中 `https://packages.ubuntu.com/bionic/amd64/python3-gi/filelist`  
可以看到
```
/usr/lib/python3/dist-packages/gi/__init__.py
/usr/lib/python3/dist-packages/gi/_constants.py
/usr/lib/python3/dist-packages/gi/_error.py
/usr/lib/python3/dist-packages/gi/_gi.cpython-36m-x86_64-linux-gnu.so
/usr/lib/python3/dist-packages/gi/_option.py
/usr/lib/python3/dist-packages/gi/_propertyhelper.py
/usr/lib/python3/dist-packages/gi/_signalhelper.py
/usr/lib/python3/dist-packages/gi/docstring.py
/usr/lib/python3/dist-packages/gi/importer.py
```
对应的so 为 `_gi.cpython-36m-x86_64-linux-gnu.so`

而在 ubuntu 19.04 版本中 `https://packages.ubuntu.com/disco/amd64/python3-gi/filelist`
```
/usr/lib/python3/dist-packages/PyGObject-3.32.0.egg-info/PKG-INFO
/usr/lib/python3/dist-packages/PyGObject-3.32.0.egg-info/dependency_links.txt
/usr/lib/python3/dist-packages/PyGObject-3.32.0.egg-info/not-zip-safe
/usr/lib/python3/dist-packages/PyGObject-3.32.0.egg-info/requires.txt
/usr/lib/python3/dist-packages/PyGObject-3.32.0.egg-info/top_level.txt
/usr/lib/python3/dist-packages/gi/__init__.py
/usr/lib/python3/dist-packages/gi/_compat.py
/usr/lib/python3/dist-packages/gi/_constants.py
/usr/lib/python3/dist-packages/gi/_error.py
/usr/lib/python3/dist-packages/gi/_gi.cpython-37m-x86_64-linux-gnu.so
/usr/lib/python3/dist-packages/gi/_gtktemplate.py
/usr/lib/python3/dist-packages/gi/_option.py
```
对应的so 为 `_gi.cpython-37m-x86_64-linux-gnu.so`


所以最后在 3.7.4 的python环境中，无法加载 3.6 对应的 `_gi` 
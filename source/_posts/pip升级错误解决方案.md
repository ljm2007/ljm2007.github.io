---
title: pip升级错误解决方案
date: 2022-11-05 16:26:23
tags: pip
categories: pip
top_img: https://pic1.zhimg.com/v2-98ece3ed5b1b7572100de49b97f71da8_1440w.jpg?source=172ae18b
cover: https://pic1.zhimg.com/v2-98ece3ed5b1b7572100de49b97f71da8_1440w.jpg?source=172ae18b
---

 今天在使用 `pip install pip -U`
 命令升级 pip 时没太注意。执行完之后界面好像报了一个错误，当时没在意。结果在安装 mysql 驱动的时候发现报错了。报错信息如下：

```
Traceback (most recent call last):
  File "e:\program files\python39\lib\runpy.py", line 197, in _run_module_as_main
    return _run_code(code, main_globals, None,
  File "e:\program files\python39\lib\runpy.py", line 87, in _run_code
    exec(code, run_globals)
  File "E:\Program Files\Python39\Scripts\pip.exe\__main__.py", line 4, in <module>
ModuleNotFoundError: No module named 'pip'
```

上面的提示信息很明显。就是因为 pip 升级没成功，导致旧版本删除了，新版本没装上。pip 就这么没了。

#                                                    解决

这个问题也好解决，分别执行如下两条命令即可：

```
python -m ensurepip
python -m pip install --upgrade pip
```

---
title: python tqdm模块分析
date: 2016-07-21 13:33:12
tags:
- python
- tqdm
categories:
- Blogs
---

这两天写我的BSqlier的时候，遇到很多问题，其中有一个就是增加进度条的时候遇到很多很多问题，用的也就是tqdm，那没办法，分析下源码吧...

<!--more-->

# 安装tqdm #

没什么可说的
```
pip install tqdm
```

当然也可以安装最新的开发版
```
pip install -e git+https://github.com/tqdm/tqdm.git@master#egg=tqdm
```

# documentation #

首先是官方文档
[https://pypi.python.org/pypi/tqdm](https://pypi.python.org/pypi/tqdm)

但是官方文档有很多错误的代码和示范...不知道为什么，那么就根据源码来看吧

# 源码分析 #

## 在分析源码之前 ##

在分析源码之前，我们首先应该看看这个模块的使用方式

### 自动控制进度更新 ###

首先最基本的用法
```
from tqdm import tqdm
for i in tqdm(range(9)):
    ...
```
通过一个列表，来生成一个进度条。

这个进度条以9为单位
```
>>> for i in tqdm(range(9)):
...     sleep(0.1)
100%|####################################################################| 9/9 [00:00<00:00,  9.95it/s]
```

当然除了tqdm，还有trange,使用方式完全相同
```
>>> for i in trange(100):
...     sleep(0.1)
100%|################################################################| 100/100 [00:10<00:00,  9.97it/s]
```

当然只要传入list就可以了
```
>>> pbar = tqdm(["a", "b", "c", "d"])
>>> for char in pbar:
...         pbar.set_description("Processing %s" % char)
Processing d: 100%|######################################################| 4/4 [00:06<00:00,  1.53s/it]
```

### 手动控制更新 ###

除了自动的更新方式，还可以手动的控制更新
```
>>> with tqdm(total=100) as pbar:
...     for i in range(10):
...         sleep(0.1)
...         pbar.update(10)
100%|################################################################| 100/100 [00:01<00:00, 99.60it/s]
```

还可以这样
```
>>> pbar = tqdm(total=100)
>>> for i in range(10):
...     sleep(0.1)
...     pbar.update(10)
100%|################################################################| 100/100 [00:24<00:00,  4.97it/s] >>> pbar.close()
```

### shell的tqdm用法 ###

官方给出的例子是这样的
```
$ find . -name '*.py' -exec cat \{} \; |
    tqdm --unit loc --unit_scale --total 857366 >> /dev/null
100%|███████████████████████████████████| 857K/857K [00:04<00:00, 246Kloc/s]
```

## 分析源码 ##

仔细分析源码后发现，其实作者在文档中已经把重要的代码逻辑罗列出来了，但是却没做详尽的说明

```
├─tqdm
│      _version.py
│      _utils.py
│      _tqdm_pandas.py
│      _tqdm_notebook.py
│      _tqdm_gui.py
│      _tqdm.py
│      _main.py
│      __init__.py
│      __main__.py
```
上面是核心代码

通过看示范的代码，我们能发现使用的核心是tqdm和trange这两个函数，从代码层面分析tqdm的功能，那首先是__init__.py

### __init__.py ###

在__init__.py中，首先能看到__all

```
__all__ = ['tqdm', 'tqdm_gui', 'trange', 'tgrange', 'tqdm_pandas',
           'tqdm_notebook', 'tnrange', 'main', 'TqdmKeyError', 'TqdmTypeError',
           '__version__']
```
能看到tqdm的所有功能，首先是tqdm，我们跟踪到_tqdm.py

### _tqdm.py ###

能看到tqdm类的声明，首先是初始化
```
def __init__(self, iterable=None, desc=None, total=None, leave=True,
                 file=sys.stderr, ncols=None, mininterval=0.1,
                 maxinterval=10.0, miniters=None, ascii=None, disable=False,
                 unit='it', unit_scale=False, dynamic_ncols=False,
                 smoothing=0.3, bar_format=None, initial=0, position=None,
                 gui=False, **kwargs):
```
而每个参数的作用，在注释中提到了...

**Parameters**

- iterable  : iterable, optional
    Iterable to decorate with a progressbar.
	可迭代的进度条。
    Leave blank to manually manage the updates.
	留空手动管理更新？？
- desc  : str, optional
    Prefix for the progressbar.
	进度条的描述
- total  : int, optional
    The number of expected iterations. If unspecified,
    len(iterable) is used if possible. As a last resort, only basic
    progress statistics are displayed (no ETA, no progressbar).
    If `gui` is True and this parameter needs subsequent updating,
    specify an initial arbitrary large positive integer,
    e.g. int(9e9).
	预期的迭代数目，默认为None，则尽可能的迭代下去，如果gui设置为True，这里则需要后续的更新，将需要指定为一个初始随意值较大的正整数，例如int(9e9)
- leave  : bool, optional
    If [default: True], keeps all traces of the progressbar
    upon termination of iteration.
	保留进度条存在的痕迹，简单来说就是会把进度条的最终形态保留下来，默认为True
- file  : `io.TextIOWrapper` or `io.StringIO`, optional
    Specifies where to output the progress messages
    [default: sys.stderr]. Uses `file.write(str)` and `file.flush()`
    methods.
	指定消息的输出
- ncols  : int, optional
    The width of the entire output message. If specified,
    dynamically resizes the progressbar to stay within this bound.
    If unspecified, attempts to use environment width. The
    fallback is a meter width of 10 and no limit for the counter and
    statistics. If 0, will not print any meter (only stats).
	整个输出消息的宽度。如果指定，动态调整的进度停留在这个边界。如果未指定，尝试使用环境的宽度。如果为0，将不打印任何东西（只统计）。
- mininterval  : float, optional
    Minimum progress update interval, in seconds [default: 0.1].
	最小进度更新间隔，以秒为单位（默认值：0.1）。
- maxinterval  : float, optional
    Maximum progress update interval, in seconds [default: 10.0].
	最大进度更新间隔，以秒为单位（默认值：10）。
- miniters  : int, optional
    Minimum progress update interval, in iterations.
    If specified, will set `mininterval` to 0.
	最小进度更新周期
- ascii  : bool, optional
    If unspecified or False, use unicode (smooth blocks) to fill
    the meter. The fallback is to use ASCII characters `1-9 #`.
	如果不设置，默认为unicode编码
- disable  : bool, optional
    Whether to disable the entire progressbar wrapper
    [default: False].
	是否禁用整个进度条包装(如果为True，进度条不显示)
- unit  : str, optional
    String that will be used to define the unit of each iteration
    [default: it].
	将被用来定义每个单元的字符串？？？
- unit_scale  : bool, optional
    If set, the number of iterations will be reduced/scaled
    automatically and a metric prefix following the
    International System of Units standard will be added
    (kilo, mega, etc.) [default: False].
	如果设置，迭代的次数会自动按照十、百、千来添加前缀，默认为false
- dynamic_ncols  : bool, optional
    If set, constantly alters `ncols` to the environment (allowing
    for window resizes) [default: False].
	不断改变ncols环境，允许调整窗口大小
- smoothing  : float, optional
    Exponential moving average smoothing factor for speed estimates
    (ignored in GUI mode). Ranges from 0 (average speed) to 1
    (current/instantaneous speed) [default: 0.3].
	
- bar_format  : str, optional
    Specify a custom bar string formatting. May impact performance.
    If unspecified, will use '{l_bar}{bar}{r_bar}', where l_bar is
    '{desc}{percentage:3.0f}%|' and r_bar is
    '| {n_fmt}/{total_fmt} [{elapsed_str}<{remaining_str}, {rate_fmt}]'
    Possible vars: bar, n, n_fmt, total, total_fmt, percentage,
    rate, rate_fmt, elapsed, remaining, l_bar, r_bar, desc.
	自定义栏字符串格式化...默认会使用{l_bar}{bar}{r_bar}的格式，格式同上
- initial  : int, optional
    The initial counter value. Useful when restarting a progress
    bar [default: 0].
	初始计数器值，默认为0
- position  : int, optional
    Specify the line offset to print this bar (starting from 0)
    Automatic if unspecified.
    Useful to manage multiple bars at once (eg, from threads).
	指定偏移，这个功能在多个条中有用
- gui  : bool, optional
    WARNING: internal parameter - do not use.
    Use tqdm_gui(...) instead. If set, will attempt to use
    matplotlib animations for a graphical output [default: False].
	内部参数...
**Returns**
- out  : decorated iterator.
  返回为一个迭代器

其实不用分析更多代码，这里已经把tqdm的核心功能展示出来了，接下来我们看别的函数

### trange ###

在_tqdm文件的最后我们能找到trange的定义
```
def trange(*args, **kwargs):
    """
    A shortcut for tqdm(xrange(*args), **kwargs).
    On Python3+ range is used instead of xrange.
    """
    return tqdm(_range(*args), **kwargs)
```

很容易看到其实就是调用了相应参数的tqdm。

所以
```
for i in trange(10): #same as: for i in tqdm(xrange(10))
```

#### tqdm的write方法 ####

仔细分析文档的发现作者在不经意间还是写了很多重要的东西，就比如tqdm的write方法。

如果测试过，你就会发现如果我们在tqdm的每次迭代中，输出任何语句，都会使得tqdm会重新输出一个新的进度条。

但是其实tqdm模块本身提供了输出信息的方法，也就是write方法

具体使用方法是这样的
```
>>> from tqdm import tqdm, trange

>>> from time import sleep

>>> bar = trange(10)
>>> for i in bar:
...     sleep(0.1)
...     if not (i % 3):
...         tqdm.write("Done task %i" % i)
Done task 0
Done task 3
Done task 6
Done task 9
100%|##################################################################| 10/10 [00:45<00:00,  3.73s/it]
```

事实上，文档中作者提到一个问题，由于环境的不确定，所以直接把print替换成tqdm.write()或许是不可取的，但其实我们只要把sys.stdout重定向到tqdm.write()就可以了，因为write的输出其实是就是sys.stdout输出的。

```
 def write(cls, s, file=sys.stdout, end="\n"):
        """
        Print a message via tqdm (without overlap with bars)
        """
        fp = file

        # Clear all bars
        inst_cleared = []
        for inst in cls._instances:
            # Clear instance if in the target output file
            # or if write output + tqdm output are both either
            # sys.stdout or sys.stderr (because both are mixed in terminal)
            if inst.fp == fp or all(f in (sys.stdout, sys.stderr)
                                    for f in (fp, inst.fp)):
                inst.clear()
                inst_cleared.append(inst)
        # Write the message
        fp.write(s)
        fp.write(end)
        # Force refresh display of bars we cleared
        for inst in inst_cleared:
            inst.refresh()
        # TODO: make list of all instances incl. absolutely positioned ones?
```

我们可以通过实例化一个类来完成，文档中给了代码

```
from time import sleep

import contextlib
import sys

from tqdm import tqdm

class DummyTqdmFile(object):
    """Dummy file-like that will write to tqdm"""
    file = None
    def __init__(self, file):
        self.file = file

    def write(self, x):
        # Avoid print() second call (useless \n)
        if len(x.rstrip()) > 0:
            tqdm.write(x, file=self.file)

@contextlib.contextmanager
def stdout_redirect_to_tqdm():
    save_stdout = sys.stdout
    try:
        sys.stdout = DummyTqdmFile(sys.stdout)
        yield save_stdout
    # Relay exceptions
    except Exception as exc:
        raise exc
    # Always restore sys.stdout if necessary
    finally:
        sys.stdout = save_stdout

def blabla():
    print("Foo blabla")

# Redirect stdout to tqdm.write() (don't forget the `as save_stdout`)
with stdout_redirect_to_tqdm() as save_stdout:
    # tqdm call need to specify sys.stdout, not sys.stderr (default)
    # and dynamic_ncols=True to autodetect console width
    for _ in tqdm(range(3), file=save_stdout, dynamic_ncols=True):
        blabla()
        sleep(.5)

# After the `with`, printing is restored
print('Done!')
```
我们只需要调用with stdout_redirect_to_tqdm() as save_stdout:

就可以自动把print重定向到tqdm.write()

### tqdm_notebook & tnrange ###

通过上面的代码来看，这两个函数也相同，tnrange可以说是tqdm_notebook的短标签。

```
def tnrange(*args, **kwargs):
    """
    A shortcut for tqdm_notebook(xrange(*args), **kwargs).
    On Python3+ range is used instead of xrange.
    """
    return tqdm_notebook(_range(*args), **kwargs)
```
所以
```
>>> for i in tnrange(10): #same as: for i in tqdm_notebook(xrange(10))
```

但是有很多问题，文档里这部分代码是
```
>>> from tqdm import tnrange, tqdm_notebook

>>> for i in tnrange(4):
...     for j in tnrange(10):
...         sleep(0.1)
```
但是我执行会报错
```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "e:\python2.7\lib\site-packages\tqdm\__init__.py", line 28, in tnrange
    return _tnrange(*args, **kwargs)
  File "e:\python2.7\lib\site-packages\tqdm\_tqdm_notebook.py", line 219, in tnrange
    return tqdm_notebook(_range(*args), **kwargs)
  File "e:\python2.7\lib\site-packages\tqdm\_tqdm_notebook.py", line 171, in __init__
    self.sp = self.status_printer(self.fp, self.total, self.desc)
  File "e:\python2.7\lib\site-packages\tqdm\_tqdm_notebook.py", line 89, in status_printer
    pbar = IntProgress(min=0, max=total)
NameError: global name 'IntProgress' is not defined
global name 'IntProgress' is not defined

Exception AttributeError: "'tqdm_notebook' object has no attribute 'sp'" in <bound method tqdm_notebook.__del__ of 0/|/  0%|| 0/4 [00:00<?, ?it/s]> ignored
```

跟踪过去，发现是模块加载的有问题
```
try:  # IPython 4.x / 3.x
        from ipywidgets import IntProgress, HBox, HTML
    except ImportError:
        try:  # IPython 2.x
            from ipywidgets import IntProgressWidget as IntProgress
            from ipywidgets import ContainerWidget as HBox
            from ipywidgets import HTML
        except ImportError:
            pass
```
这里报错pass了，但是后面仍然引用了IntProgress

好吧...我放弃了

### tgrange & tqdm_gui ###

```
>>> from tqdm_gui import tgrange[, tqdm_gui]
>>> for i in tgrange(10): #same as: for i in tqdm_gui(xrange(10))
```

### tqdm_pandas ###

```
>>> import pandas as pd
    >>> import numpy as np
    >>> from tqdm import tqdm, tqdm_pandas
    >>>
    >>> df = pd.DataFrame(np.random.randint(0, 100, (100000, 6)))
    >>> tqdm_pandas(tqdm())  # can use tqdm_gui, optional kwargs, etc
    >>> # Now you can use `progress_apply` instead of `apply`
    >>> df.groupby(0).progress_apply(lambda x: x**2)
```

有时间继续补完...
---
title: python中重定向外部C模组的标准输出
urlname: python-redirect
categories:
  - 编程
tags:
  - python
date: 2020-02-20 20:20:44
---

编程中遇到的一个问题。在python里重定向所有的如`print`等的结果并不困难，只需要给`sys.stdout`赋值一个文件对象即可（实际上在python3.4及以后，`contextlib`里就有一个`redirect_stdout`的函数可以直接运行）。

（翻译自<https://eli.thegreenplace.net/2015/redirecting-all-kinds-of-stdout-in-python/>）

{% codeblock lang:python wrap:false %}
import sys
from contextlib import contextmanager

@contextmanager
def redirect_stdout(stream):
    old_stdout = sys.stdout
    sys.stdout = stream
    try：
        yield
    finally:
        sys.stdout = old_stdout

f = io.StringIO()
with redirect_stdout(f):
    print('foobar')
    print(12)
print('Got stdout: "{0}"'.format(f.getvalue()))
{% endcodeblock %}

但是，如果某个module里有在C里直接`printf`或是`cout`字符串，上面的就不管用了。比如例子：

```python
import ctypes
libc = ctypes.CDLL(None)

f = io.StringIO()
with stdout_redirector(f):
    print('foobar')
    print(12)
    libc.puts(b'this comes from C')
    os.system('echo and this is from echo')
print('Got stdout: "{0}"'.format(f.getvalue()))
```

使用`ctypes`模拟C中的`puts`函数，这和在Python中调用C的函数输出到`stdout`的结果是一样的。另一个测试是使用`os.system`创建一个子进程同样输出至`stdout`。而这的运行结果是：

```
this comes from C
and this is from echo
Got stdout: "foobar
12
"
```

`print`的结果相同，而`puts`和`echo`则直接跳过了重定向被输出到了屏幕上。这是为什么呢？

<!-- more -->

## 文件描述符（File Descriptor）与流

解决方案在下一部分，这一部分则是介绍一些原理。

{% asset_img fd-inode-diagram.png %}

像图中一样，文件是由操作系统维护的列表，其中可能有不同的指针指向相同的硬盘数据（比如两个不同的进程访问同一个文件）。而文件描述符则是另一层抽象，是由进程管理。每一个进程有其文件描述符指向系统公用的表格。

文件描述符允许不同进程指向同一文件，同时也对重定向一者至另一者很有用。比如说我们让文件描述符5是文件描述符4的复制，则对5的写入就会和对4的产生相同效果。Unix上的标准输出通常是另一个文件描述符（一般是1）。C库里的流，就是对文件描述符的抽象。通过如`fprintf`，`fgets`等的函数操作于其上。

而python呢？python使用的对文件描述符的抽象则是文件对象（file object）。但是重点是，python，和其加载的C扩展，**使用的是同一个文件描述符**。由于python对其的抽象是`sys.stdout`，C扩展的抽象是C自己的文件对象`FILE`，所以仅仅覆盖`sys.stdout`无法对C扩展的标准输出进行重定向。我们需要触及到其底层的文件描述符。

## 重定向C扩展的标准输出

```python
from contextlib import contextmanager
import ctypes
import io
import os, sys
import tempfile

libc = ctypes.CDLL(None)
c_stdout = ctypes.c_void_p.in_dll(libc, 'stdout')

@contextmanager
def stdout_redirector(stream):
    # The original fd stdout points to. Usually 1 on POSIX systems.
    original_stdout_fd = sys.stdout.fileno()

    def _redirect_stdout(to_fd):
        """Redirect stdout to the given file descriptor."""
        # Flush the C-level buffer stdout
        libc.fflush(c_stdout)
        # Flush and close sys.stdout - also closes the file descriptor (fd)
        sys.stdout.close()
        # Make original_stdout_fd point to the same file as to_fd
        os.dup2(to_fd, original_stdout_fd)
        # Create a new sys.stdout that points to the redirected fd
        sys.stdout = io.TextIOWrapper(os.fdopen(original_stdout_fd, 'wb'))

    # Save a copy of the original stdout fd in saved_stdout_fd
    saved_stdout_fd = os.dup(original_stdout_fd)
    try:
        # Create a temporary file and redirect stdout to it
        tfile = tempfile.TemporaryFile(mode='w+b')
        _redirect_stdout(tfile.fileno())
        # Yield to caller, then redirect stdout back to the saved fd
        yield
        _redirect_stdout(saved_stdout_fd)
        # Copy contents of temporary file to the given stream
        tfile.flush()
        tfile.seek(0, io.SEEK_SET)
        stream.write(tfile.read())
    finally:
        tfile.close()
        os.close(saved_stdout_fd)
```

这里细节挺多，比如使用了`os.dup`和`os.dup2`操作文件描述符。如果你想具体了解这两个函数的话，请看它们的官方文档，这里不会详细介绍。

```python
f = io.BytesIO()

with stdout_redirector(f):
    print('foobar')
    print(12)
    libc.puts(b'this comes from C')
    os.system('echo and this is from echo')
print('Got stdout: "{0}"'.format(f.getvalue().decode('utf-8')))
```

这段代码给出的则是下面的输出：

```
Got stdout: "and this is from echo
this comes from C
foobar
12
"
```

注释：
1. 输出顺序可能并不如我们预想的一样，这是由缓存引起的。如果不同类型的输出之间顺序必须保存，需要另一些操作去关闭不同流的缓存功能。
2. 上面的代码是python3使用的，转移到python2很简单，因为仅仅是如`sys.stdout`不是如python3一样的被`io.TextIOWrapper`包裹等。
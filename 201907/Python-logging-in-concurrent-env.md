Python 多进程下处理日志的思路
=============

在最近的工作开发中，处理了一个 Python 多进程的问题，现记录一下处理的过程和思路

### 背景


在前一次的项目中，项目运行环境为 Gunicorn + Django， Gunicorn 拉起了10个进程，日志设定为 Python 自带的 `TimeRotateHandler` 模块，但是运维同学吐槽两个问题

- 日志切割失败，表现上看仅有一个进程成功的切割了日志，其他都失败了，导致单个日志文件过大，难以处理
- 日志命名不符合规范，运维同学希望日志总以 .log 结尾，当 `TimeRotateHandler` 是以类似 xxx.log.2019-7-1 这种格式

最后，运维同学跟我商量希望我能解决日志问题满足：

- 日志成功按照 时间 切割，每天切割一次
- 同时，日志按照大小切割，每 5G 切割一次
- 日志均要以 .log 结尾，方便他们 模式 匹配等工作


### 问题分析

首先，Python 的 Logging 模块是不支持多进程的，仅仅是线程安全。Python TimerorateHandler 中切换日志的源码为 
对于有问题的代码做了简单的解释

```python
def doRollover(self):
        """
        do a rollover; in this case, a date/time stamp is appended to the filename
        when the rollover happens.  However, you want the file to be named for the
        start of the interval, not the current time.  If there is a backup count,
        then we have to get a list of matching filenames, sort them and remove
        the one with the oldest suffix.
        """
        if self.stream:
            self.stream.close()
            self.stream = None
        # get the time that this sequence started at and make it a TimeTuple
        currentTime = int(time.time())
        dstNow = time.localtime(currentTime)[-1]
        t = self.rolloverAt - self.interval
        if self.utc:
            timeTuple = time.gmtime(t)
        else:
            timeTuple = time.localtime(t)
            dstThen = timeTuple[-1]
            if dstNow != dstThen:
                if dstNow:
                    addend = 3600
                else:
                    addend = -3600
                timeTuple = time.localtime(t + addend)
        dfn = self.rotation_filename(self.baseFilename + "." +
                                     time.strftime(self.suffix, timeTuple))

        ### !!! 以下 两句代码表明，当切割时，如果目标路径已经存在，那么会删除文件 
        ### 这就会造成进程间互相删除对方建立的切割后的文件
        ### 同时 os.rename 的时候肯定有其他的进程仍然在写入，在 windows 下会直接报 permission 错误
        ### unix 下没测试过，从结果上看应该是 rename 失败
        if os.path.exists(dfn):
            os.remove(dfn)
        self.rotate(self.baseFilename, dfn)
        if self.backupCount > 0:
            for s in self.getFilesToDelete():
                os.remove(s)
        if not self.delay:
            self.stream = self._open()
        newRolloverAt = self.computeRollover(currentTime)
        while newRolloverAt <= currentTime:
            newRolloverAt = newRolloverAt + self.interval
        #If DST changes and midnight or weekly rollover, adjust for this.
        if (self.when == 'MIDNIGHT' or self.when.startswith('W')) and not self.utc:
            dstAtRollover = time.localtime(newRolloverAt)[-1]
            if dstNow != dstAtRollover:
                if not dstNow:  # DST kicks in before next rollover, so we need to deduct an hour
                    addend = -3600
                else:           # DST bows out before next rollover, so we need to add an hour
                    addend = 3600
                newRolloverAt += addend
        self.rolloverAt = newRolloverAt

```

这意味着不能在项目中使用 Python 自带的 Handler，需要按需要重新写一个 Handler。


然后是同时按大小和时间切割的需求，目前 Python logging 中要么仅支持时间滚动要么仅支持时间滚动，解决方案是符合两个 Handler 的代码，达到同时切割的目的。


然后是日志命名，Python 标准库中关于命名的函数代码为

```python


def rotation_filename(self, default_name):
        """
        Modify the filename of a log file when rotating.

        This is provided so that a custom filename can be provided.

        The default implementation calls the 'namer' attribute of the
        handler, if it's callable, passing the default name to
        it. If the attribute isn't callable (the default is None), the name
        is returned unchanged.

        :param default_name: The default name for the log file.
        """
        if not callable(self.namer):
            result = default_name
        else:
            result = self.namer(default_name)
        return result

```

在 TimeRotateHandler 中

```python 

dfn = self.rotation_filename(self.baseFilename + "." +
                            time.strftime(self.suffix, timeTuple))

```

可以看到是直接把时间后缀加到 .log 之后，这部分需要重写一个  `rotation_filename` 函数即可。


### 解决过程


##### 进程安全的解决办法

- 将就 Python 的线程安全组件，让进程各自打各自的日志，互不影响
- 一定要实现进程安全的组件

两种我都实现过，最后因为性能问题采取了第一种。

第一种的实现非常简单，在传入给 Handler 的 filename 上做一个小手脚，加上进程号即可，伪代码如下

```python

import os

def init_logfilename(file_name):
    '''

    step 1 : 获取进程号
    step 2 : 在 $name 和 .log 中间加上进程号即可 
        example: 从 info.log  -> info.100.log 
    '''
    
    pid = os.getpid()
    file_name_split = file_name.rsplit('.', 1)
    file_name_split.insert(str(pid), 1)
    return '.'.join(file_name_split)


```

这样，各自的进程产生的日志文件各不相同，达到了切割的目的。


第二种实现起来稍微复杂一点(大雾，是巨复杂)

github 上现成的代码供参考 [concurrent-log-handler](https://github.com/Preston-Landers/concurrent-log-handler)

作者使用了一个文件锁，各个进程在写入的时候都存在 lock file -> unlock file 的过程。保证某一个特定的时刻仅有一个进程在对日志文件进行控制。

这样的弊端也很明显，如果进程过多，会影响 io 效率，然后拖累业务逻辑。

综上，如果对日志数量没有要求，并且存在大量进程的情况下，第一种是更好的选择。


#### 处理按大小和时间同时切割


查看 Python 的源码，在写入到文件的过程中有以下代码


```python 


def emit(self, record):
    """
    Emit a record.

    Output the record to the file, catering for rollover as described
    in doRollover().
    """
    try:
        if self.shouldRollover(record):
            self.doRollover()
        logging.FileHandler.emit(self, record)
    except Exception:
        self.handleError(record)


```

意思是在每次写入的时候都检查一下是否满足 rollover 的条件，满足的话就做一次 rollover，然后再写入到文件。


所以这里增加一下判断就行了, 伪代码


```python 


def emit(self, record):
    """
    Emit a record.

    Output the record to the file, catering for rollover as described
    in doRollover().
    """
    try:
        if self.time_shouldRollover(record):
            self.time_doRollover()
        elif self.file_shouldRollover(record):
            self.file_doRollover()
        logging.FileHandler.emit(self, record)
    except Exception:
        self.handleError(record)


```

如果时间上满足切割的要求，就按时间切割，如果文件大小上满足切割的要求，就按大小进行切割。各自的 `shouldRollover` 和 `doRollover` 函数借鉴 Python 标准库的实现即可。


#### 日志命名


运维同学希望以 `.log` 结尾，然后综上所述，我们需要区分的维度有

- pid
- 时间
- 大小

所有最后拟了一个标准

info.log  ->  info.100(pid).2019-7-6(date).0(大小).log


同样伪代码如下

```python

import os
import time

def rotate_filename(file_name):

    pid = os.getpid()
    current = int(time.time())
    file_sequence = 0

    prefix, suffix = file_name.rsplit(".", 1)
    return ".".join([prefix, pid, current, suffix])

```

就此，我们有了解决所有的问题的思路，接下来就是编写代码了。


### 代码实现

代码实现不表，写这篇文章的时候，我实现的代码仅能满足工作中的要求，离完美还差一点，先不给链接了。


### 测试

同样不表。


### 总结

一次很棒的实践。同时又加深了多线程进程的理解。

2019.07.06. zhouxin

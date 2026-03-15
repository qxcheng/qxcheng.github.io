---
categories:
- - Python
date: 2024-04-16 14:05:47
description: 'Python基础8-日志处理'
id: '86'
tags:
- python
title: Python基础8-日志处理
---


## 1.基本使用

```Python
import logging  

logging.basicConfig(
    level=logging.DEBUG,  # 信息级别：DEBUG<INFO<WARNING<ERROR
    filename = "log1.txt",
    format = '%(asctime)s - %(name)s - %(levelname)s : %(message)s'
)

logger = logging.getLogger(__name__)
logger.info("Start wirte log")
logger.warning("Something maybe wrong")
logger.debug("Try to fix bug")
logger.info("Finish")
```

## 2.日志封装

```python
import logging
import time
import os 

class Logger(object):
    def __init__(self):
        log_path = './logs/'
        if not os.path.exists(log_path):
            os.mkdir(log_path)

        self.logname = os.path.join(log_path, '%s.log' % time.strftime('%Y_%m_%d'))
        self.logger = logging.getLogger()
        self.logger.setLevel(logging.DEBUG)
        self.formatter = logging.Formatter('[%(asctime)s - %(filename)s] - %(levelname)s: %(message)s')

    # 写到本地日志文件
    def __console(self, level, message):
        fh = logging.FileHandler(self.logname, 'a', encoding='utf-8')  
        fh.setLevel(logging.DEBUG)
        fh.setFormatter(self.formatter)
        self.logger.addHandler(fh)

        if level == 'info':
            self.logger.info(message)
        elif level == 'debug':
            self.logger.debug(message)
        elif level == 'warning':
            self.logger.warning(message)
        elif level == 'error':
            self.logger.error(message)

        self.logger.removeHandler(fh)
        fh.close()

    def info(self, message):
        self.__console('info', message)

    def debug(self, message):
        self.__console('debug', message)

    def warning(self, message):
        self.__console('warning', message)

    def error(self, message):
        self.__console('error', message)

if __name__ == '__main__':
    l = Logger()
    l.info("info")
    l.debug("debug")
    l.warning("warning")
    l.error("error")
```

## 3.多进程日志封装

`pip install concurrent-log-handler`

```Python
import logging
import time
import os
from concurrent_log_handler import ConcurrentRotatingFileHandler
import multiprocessing as mp
from multiprocessing import Pool

class Logger(object):
    def __init__(self):
        log_path = './logs/'
        if not os.path.exists(log_path):
            os.mkdir(log_path)

        self.logname = os.path.join(log_path, '%s.log' % time.strftime('%Y_%m_%d'))
        self.logger = logging.getLogger()
        self.logger.setLevel(logging.DEBUG)
        self.formatter = logging.Formatter('[%(asctime)s - %(filename)s] - %(levelname)s: %(message)s')

    # 写到本地日志文件
    def __console(self, level, message):
        handler = ConcurrentRotatingFileHandler(self.logname, "a", 20 * 1024 * 1024, 10)
        handler.setFormatter(self.formatter)
        self.logger.addHandler(handler)

        if level == 'info':
            self.logger.info(message)
        elif level == 'debug':
            self.logger.debug(message)
        elif level == 'warning':
            self.logger.warning(message)
        elif level == 'error':
            self.logger.error(message)

        self.logger.removeHandler(handler)

    def info(self, message):
        self.__console('info', message)

    def debug(self, message):
        self.__console('debug', message)

    def warning(self, message):
        self.__console('warning', message)

    def error(self, message):
        self.__console('error', message)

def func(message):
    l = Logger()
    l.debug('PID-{}: {}'.format(mp.current_process().pid, message))
    time.sleep(0.1)

if __name__ == '__main__':
    p = Pool(3)
    for i in range(10):
        p.apply_async(func, args=(i,))    

    p.close() 
    p.join()
```
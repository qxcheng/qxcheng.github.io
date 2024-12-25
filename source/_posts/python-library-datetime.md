---
categories:
- - Python
date: 2024-04-17 13:21:53
description: 'Python常用库2-时间日期'
id: '113'
tags:
- python
title: Python常用库2-时间日期
---


## 1.time

```python
import time

t = time.time()         
print(t)  # 1594974068.2558458

lt = time.localtime(t)  
print(lt) # time.struct_time(tm_year=2020, tm_mon=7, tm_mday=17, tm_hour=16, tm_min=22, tm_sec=2, tm_wday=4, tm_yday=199, tm_isdst=0)

at = time.asctime(lt)   
print(at) # Fri Jul 17 16:23:03 2020

# 时间戳转字符串
print(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()))   
# 2020-07-17 16:26:02
print(time.strftime("%a %b %d %H:%M:%S %Y", time.localtime()))
# Fri Jul 17 16:26:02 2020

# 将格式字符串转换为时间戳
a = "Fri Jul 17 16:26:02 2020"
print(time.mktime(time.strptime(a,"%a %b %d %H:%M:%S %Y")))
# 1594974362.0

# %y 两位数的年份表示（00-99）
# %Y 四位数的年份表示（000-9999）
# %m 月份（01-12）
# %d 月内中的一天（0-31）
# %H 24小时制小时数（0-23）
# %I 12小时制小时数（01-12）
# %M 分钟数（00=59）
# %S 秒（00-59）
```

## 2.datetime

```python
from datetime import datetime, timedelta, timezone, date

# 日期创建
now = datetime.now()
print(now)    # 2023-09-15 09:18:50.647649

today = now.date()
print(today)  # 2023-09-15

print(datetime(2020, 7, 17, 16, 35))  # 2020-07-17 16:35:00

# datetime和timestamp
ts1 = now.timestamp()
print(ts1)  # 1694740730.647649
print(datetime.fromtimestamp(ts1))  # 2023-09-15 09:18:50.647649       

# date和timestamp
ts2 = time.mktime(today.timetuple())  # date转timestamp
print(date.fromtimestamp(ts2))        # timestamp转date 2023-09-15

# datetime和string
## str转换为datetime  2020-07-17 16:35:59
print(datetime.strptime('2020-7-17 16:35:59', '%Y-%m-%d %H:%M:%S'))  
## datetime转换为str  2023-09-15 09:18:50
print(now.strftime('%Y-%m-%d %H:%M:%S'))                   

# datetime加减
now += timedelta(days=2, hours=12)
print(now)  # 2023-09-17 21:18:50.647649
now -= timedelta(days=1)
print(now)  # 2023-09-16 21:18:50.647649

# datetime转换为str  Fri, Jul 17 16:38
print(datetime.now().strftime('%Y-%m-%d %H:%M:%S'))   # %a, %b %d %H:%M                  

# 本地时间转换为UTC时间，如果系统时区恰好是UTC+8:00，那么下列代码就是正确的，否则，不能强制设置为UTC+8:00时区
tz_utc_8 = timezone(timedelta(hours=8)) # 创建时区UTC+8:00
now = datetime.now()
dt = now.replace(tzinfo=tz_utc_8)       # 强制设置为UTC+8:00
print(dt)

# 时区转换 拿到UTC时间，并强制设置时区为UTC+0:00:
utc_dt = datetime.utcnow().replace(tzinfo=timezone.utc)
# astimezone()将转换时区为北京时间:
bj_dt = utc_dt.astimezone(timezone(timedelta(hours=8)))
print(bj_dt)
# 不是必须从UTC+0:00时区转换到其他时区，任何带时区的datetime都可以正确转换

# 格式化输出
d = date(2020, 7, 17)
print(format(d, '%A, %B %d, %Y'))
# Friday, July 17, 2020
print('Today is {:%d %b %Y}'.format(d))
# Today is 17 Jul 2020
```

## 3.常用日期

```python
# 今天
today = datetime.now().date()
print(today)      # 2023-09-15

# 昨天
yesterday = today - timedelta(days=1)
print(yesterday)  # 2023-09-14

# 明天
tomorrow = today + timedelta(days=1)
print(tomorrow)   # 2023-09-16

# 本周第一天和最后一天
this_week_start = today - timedelta(days=today.weekday())
this_week_end = today + timedelta(days=6-today.weekday())
print(this_week_start, this_week_end)  # 2023-09-11 2023-09-17

# 上周第一天和最后一天
last_week_start = today - timedelta(days=today.weekday()+7)
last_week_end = today - timedelta(days=today.weekday()+1)
print(last_week_start, last_week_end)  # 2023-09-04 2023-09-10

# 本月第一天和最后一天
this_month_start = date(today.year, today.month, 1)
this_month_end = date(today.year, today.month + 1, 1) - timedelta(days=1)
print(this_month_start, this_month_end)  # 2023-09-01 2023-09-30

# 上月第一天和最后一天
last_month_end = this_month_start - timedelta(days=1)
last_month_start = date(last_month_end.year, last_month_end.month, 1)
print(last_month_start, last_month_end)  # 2023-08-01 2023-08-31

# 本季第一天和最后一天
month = (today.month - 1) - (today.month - 1) % 3 + 1
this_quarter_start = date(today.year, month, 1)
this_quarter_end = date(today.year, month + 3, 1) - timedelta(days=1)
print(this_quarter_start, this_quarter_end)  # 2023-07-01 2023-09-30

# 上季第一天和最后一天
last_quarter_end = this_quarter_start - timedelta(days=1)
last_quarter_start = date(last_quarter_end.year, last_quarter_end.month - 2, 1)
print(last_quarter_start, last_quarter_end)  # 2023-04-01 2023-06-30

#本年第一天和最后一天
this_year_start = date(today.year, 1, 1)
this_year_end = date(today.year + 1, 1, 1) - timedelta(days=1)
print(this_year_start, this_year_end)  # 2023-01-01 2023-12-31

# 去年第一天和最后一天
last_year_end = this_year_start - timedelta(days=1)
last_year_start = date(last_year_end.year, 1, 1)
print(last_year_start, last_year_end)  # 2022-01-01 2022-12-31
```

## 4.calendar

```Python
import calendar
cal = calendar.month(2020, 12)
print(cal)
```
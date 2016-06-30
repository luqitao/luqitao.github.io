---
layout: post
title: Openstack taskflow introduction
tags: openstack, taskflow
---

Openstack taskflow介绍
---

# 前言
TaskFlow是OpenStack中的一个Python库，主要目的是让task(任务)执行更加容易可靠，能将轻量的任务对象组织成一个有序的流。
若未安装taskflow到环境中:

```
pip install taskflow
```

目前TaskFlow支持三种模式：
> - 线性：运行一个任务或流的列表，是一个接一个串行方式运行。
> - 无序：运行一个任务或流的列表，以并行的方式运行，顺序与列表顺序无关，任务之间不存在依赖关系。
> - 图：运行一个图标（组节点和边缘节点）之间组成的任务/流依赖驱动的顺序。

# 任务的状态
就像任何其他的任务流系统一样，每个任务都有一些状态：

- PENDING 
- RUNNING 
- SUCCESS 
- FAILURE

你也可以创建自定义的状态。

# 举个例子
线性：

```
#!/usr/bin/env python
# coding=utf-8

from __future__ import print_function
import taskflow.engines
from taskflow.patterns import linear_flow as lf
from taskflow import task


class A(task.Task):
    def execute(self, a_msg, *args, **kwargs):
        print('A : {}' . format(a_msg))


class B(task.Task):
    def execute(self, b_msg, *args, **kwargs):
        print('B : {}' . format(b_msg))

flow = lf.Flow('simple-linear-listen').add(
    A(),
    B()
    )

engine = taskflow.engines.load(flow, store=dict(a_msg='a', b_msg='b'))
engine.run()
```

Output:

```
A : a
B : b
```

说明：A任务永远都会在B任务之前。
检查任务状态
修改代码：

```
#!/usr/bin/env python
# coding=utf-8

from __future__ import print_function
import taskflow.engines
from taskflow.patterns import linear_flow as lf
from taskflow import task


def flow_watch(state, details):
    print("Flow State:{}".format(state))
    print("Flow Details:{}".format(details))


class A(task.Task):
    def execute(self, a_msg, *args, **kwargs):
        print('A:{}' . format(a_msg))


class B(task.Task):
    def execute(self, b_msg, *args, **kwargs):
        print('B:{}' . format(b_msg))

flow = lf.Flow('simple-linear-listen').add(
    A(),
    B()
    )

engine = taskflow.engines.load(flow, store = dict(a_msg = 'a', b_msg = 'b'))
engine.notifier.register('*', flow_watch)
engine.run()
```

注册了一个监听器将报告给flow_wtach函数。
Output:

```
Flow State:RUNNING
Flow Details:{'engine': <taskflow.engines.action_engine.engine.SerialActionEngine object at 0x0333A2D0>, 'old_state': 'PENDING', 'flow_name': u'simple-linear-listen', 'flow_uuid': '59385218-0fc8-4566-a308-17b0d69cf8b2'}
A:a
B:b
Flow State:SUCCESS
Flow Details:{'engine': <taskflow.engines.action_engine.engine.SerialActionEngine object at 0x0333A2D0>, 'old_state': 'RUNNING', 'flow_name': u'simple-linear-listen', 'flow_uuid': '59385218-0fc8-4566-a308-17b0d69cf8b2'}
```

当流状态发生改变，就会被捕捉到，若只监听流状态，也可以改为:

```
engine.notifier.register('SUCCESS', flow_watch)
```

也可以做到监听任务：

# 任务异常
在一组任务中，若其中一个发生异常，流的任务失败，就需要处理异常工作：

```
#!/usr/bin/env python
# coding=utf-8

from __future__ import print_function
import taskflow.engines
from taskflow.patterns import linear_flow as lf
from taskflow import task


class A(task.Task):
    def execute(self, a_msg, *args, **kwargs):
        print('A : {}' . format(a_msg))

    def revert(self, a_msg, *args, **kwargs):
        print('A {} revert' . format(a_msg))


class B(task.Task):
    def execute(self, b_msg, *args, **kwargs):
        print('B : {}' . format(b_msg))

    def revert(self, b_msg, *args, **kwargs):
        print('B {} revert' .format(b_msg))


class C(task.Task):
    def execute(self, c_msg, *args, **kwargs):
        print('C : {}' . format(c_msg))
        raise IOError('C IOError')


flow = lf.Flow('simple-linear-listen').add(
    A(),
    B(),
    C()
    )

engine = taskflow.engines.load(flow, store = dict(a_msg = 'a', b_msg = 'b',c_msg = 'c'))
try:
    engine.run()
except Exception as e:
    print("flow failed:{}" .format(e))
```

Output:

```
A : a
B : b
C : c
B b revert
A a revert
flow failed:C IOError
```

说明，如果出现异常，会执行revert函数进行清理工作。
相关链接：[TaskFlow维基](https://wiki.openstack.org/wiki/TaskFlow)
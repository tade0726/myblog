# Python SimPy 仿真系列 \(1\)

![](../.gitbook/assets/v2-7d1a96a18fdc4e5d4a4810022b536312_1200x500.jpg)

本系列文章旨在介绍 SimPy 在工业仿真中的应用。

在**物流行业**/**工厂制造业**/**餐饮服务业**存在大量急需优化的场景， 例如：

* 如何最优化快递分拣人员的排班表以满足双十一突发的快递件量
* 如何估算餐厅在用餐高峰的排队时长
* 估算特定工序下，工厂生产所需要的物料成本/人力成本/时间成本

这类场景无法通过常规算法求出最优解, 但是我们可以通过大量业务实践中总结出一些接近的次优解。

实际生产中，随时调整厂房的生产线来试验最优解是非常昂贵的。引进仿真技术，可以给业务研究员无限的自由度去调整验证不同的优化方案。仿真的成本无非是计算机的算力，以及程序员编写业务逻辑的时间。

行业上其实已经存在一些工业仿真软件。但这类仿真软件往往针对某些特定场景高度定制化，数据埋点往往不全，缺乏通用的数据库接口，难以结合真实业务产生的数据进行仿真，这样就失去与真实业务进行比较的可能。

利用 SimPy 我们可以构建一套完全开源的仿真方案，可以完全私有定制业务场景。利用 Python 强大的生态，仿真数据从来源到输出分析，可以衔接所有开源流行的数据分析框架。

目前，我们已经利用 SimPy 仿真模拟**物流核心分拣业务**，结合 MySQL，Tableau，Pandas，Spark 构建一整套完整的报表可视化分析体系，已经能够成功应用于**现代物流**中，为分拣业务提供持续优化改良方案。

我们创造性地解决了一些原有软件仿真中欠缺的环节，这些内容将会在接下来的文章中分享。

作为本系列文章的开篇，我们将简要地介绍 SimPy 框架的基本理念。

### 官方资料

SimPy 是一个基于标准 Python 以进程为基础的离散事件仿真框架。

SimPy 中的进程是由 Python 生成器构成，生成器的特性可以模拟具有主动性的物件，比如客户、汽车、或者中介等等。SimPy也提供多种类的共享资源（shared resource）来描述拥挤点（比如服务器、收银台和隧道）。

仿真运行速度非常快，仿真中的模拟时间长短不影响仿真运行效率，仿真中的模拟时间单位可以任意指定，一秒、一年、一小时都是允许的。

* SimPy 的官方文档可以查询自：[https://simpy.readthedocs.io/en/latest/](https://simpy.readthedocs.io/en/latest/)
* SimPy 代码托管网址：[https://bitbucket.org/simpy/simpy/src/default/docs/index.rst?fileviewer=file-view-default](https://bitbucket.org/simpy/simpy/src/default/docs/index.rst?fileviewer=file-view-default)
* PyPy 网址：[https://pypi.python.org/pypi/simpy](https://pypi.python.org/pypi/simpy)
* 原理性介绍的ppy:  [https://stefan.sofa-rockers.org/downloads/simpy-ep14.pdf](https://stefan.sofa-rockers.org/downloads/simpy-ep14.pdf)

### SimPy 安装

SimPy 可以同时在 Python 2 （&gt;=2.7）以及 Python 3（&gt;=3.2）上运行。只要有pip，轻松安装。

> “$ pip install simpy”

手动安装 SimPy 也非常方便。提取存档，打开存放 SimPy 的 terminal 窗口，然后输入：

> “$ python setup.py install”

你可以选择性地运行 SimPy 测试文件以了解软件是否可行。前提是要安装 pytest 包。并在 SimPy 的安装路径下运行下列命令行：

> “$ py.test --pyargs simpy”

### SimPy 核心概念

SimPy 是离散事件驱动的仿真库。所有活动部件，例如车辆、顾客,、即便是信息，都可以用 `process` \(进程\) 来模拟。这些 `process` 存放在 `environment` \(环境\) 。所有 `process` 之间，以及与`environment` 之间的互动，通过 `event` \(事件\) 来进行.

`process` 表达为 `generators` \(生成器\)， 构建`event`\(事件\)并通过 `yield` 语句抛出事件。

当一个进程抛出事件，进程会被暂停，直到事件被激活\(`triggered`\)。多个进程可以等待同一个事件。 SimPy 会按照这些进程抛出的事件激活的先后， 来恢复进程。

其实中最重要的一类事件是 `Timeout`， 这类事件允许一段时间后再被激活， 用来表达一个进程休眠或者保持当前的状态持续指定的一段时间。这类事件通过 `Environment.timeout`来调用。

#### Environment

`Environment` 决定仿真的起点/终点， 管理仿真元素之间的关联, 主要 `API` 有

* `simpy.Environment.process` - 添加仿真进程
* `simpy.Environment.event` - 创建事件
* `simpy.Environment.timeout` -  提供延时\(timeout\)事件
* `simpy.Environment.until` - 仿真结束的条件（时间或事件）
* `simpy.Environment.run` - 仿真启动

**样例代码说明 API:**

下面是来自官方文档的两个例子:

* 第一个例子， 描述如何定义一个进程， 并添加到 env 内， 简单展示启动仿真的代码结构
* 第二个例子， 描述一个汽车驾驶一段时间后停车充电， 汽车驾驶进程和电池充电进程通过事件的激活来相互影响

**Example 1**

```python
import simpy

# 定义一个汽车进程
def car(env):
    while True:
        print('Start parking at %d' % env.now)
        parking_duration = 5
        yield env.timeout(parking_duration) # 进程延时 5s
        print('Start driving at %d' % env.now)
        trip_duration = 2
        yield env.timeout(trip_duration)   # 延时 2s

# 仿真启动
env = simpy.Environment()   # 实例化环境
env.process(car(env))   # 添加汽车进程
env.run(until=15)   # 设定仿真结束条件, 这里是 15s 后停止
```

**Example 2**

```python
from random import seed, randint
seed(23)

import simpy

class EV:
    def __init__(self, env):
        self.env = env
        self.drive_proc = env.process(self.drive(env))
        self.bat_ctrl_proc = env.process(self.bat_ctrl(env))
        self.bat_ctrl_reactivate = env.event()
        self.bat_ctrl_sleep = env.event()


    def drive(self, env):
        """驾驶进程"""
        while True:
            # 驾驶 20-40 分钟
            print("开始驾驶 时间: ", env.now)
            yield env.timeout(randint(20, 40))
            print("停止驾驶 时间: ", env.now)

            # 停车 1-6 小时
            print("开始停车 时间: ", env.now)
            self.bat_ctrl_reactivate.succeed()  # 激活充电事件
            self.bat_ctrl_reactivate = env.event()
            yield env.timeout(randint(60, 360)) & self.bat_ctrl_sleep # 停车时间和充电程序同时都满足
            print("结束停车 时间:", env.now)

    def bat_ctrl(self, env):
        """电池充电进程"""
        while True:
            print("充电程序休眠 时间:", env.now)
            yield self.bat_ctrl_reactivate  # 休眠直到充电事件被激活
            print("充电程序激活 时间:", env.now)
            yield env.timeout(randint(30, 90))
            print("充电程序结束 时间:", env.now)
            self.bat_ctrl_sleep.succeed()
            self.bat_ctrl_sleep = env.event()

def main():
    env = simpy.Environment()
    ev = EV(env)
    env.run(until=300)

if __name__ == '__main__':
    main()
```

#### Resource 和 Store

`Resource`/`Store` 也是另外一类重要的核心概念, 但凡仿真中涉及的人力资源以及工艺上的物料消耗都会抽象用 `Resource` 来表达, 主要的 `method` 是 `request`. `Store` 处理各种优先级的队列问题, 表现跟 `queue` 一致, 通过 `method` `get` / `put` 存放 `item`

**Store - 抽象队列**

* `simpy.Store` - 存取 `item` 遵循仿真时间上的先到后到
* `simpy.PriorityStore` - 存取 `item` 遵循仿真时间上的先到后到同时考虑人为添加的优先级
* `simpy.FilterStore` - 存取 `item` 遵循仿真时间上的先到后到, 同时队列中存在分类, 按照不同类别进行存取
* `simpy.Container` - 表达连续/不可分的物质, 包括液体/气体的存放, 存取的是一个 `float` 数值

**Resource - 抽象资源**

* `simpy.Resource` - 表达人力资源或某种限制条件, 例如某个工序可调用的工人数, 可以调用的机器数
* `simpy.PriorityResource` - 兼容`Resource`的功能, 添加可以插队的功能, 高优先级的进程可以优先调用资源, 但只能是在前一个被服务的进程结束以后进行插队
* `simpy.PreemptiveResource` - 兼容`Resource`的功能, 添加可以插队的功能, 高优先级的进程可以打断正在被服务的进程进行插队

**样例代码说明 API:**

**Example 3**

```python
"""
银行排队服务例子

情景:
  一个柜台对客户进行服务, 服务耗时, 客户等候过长会离开柜台
"""

import random
import simpy


RANDOM_SEED = 42
NEW_CUSTOMERS = 5 # 客户数
INTERVAL_CUSTOMERS = 10.0 # 客户到达的间距时间
MIN_PATIENCE = 1 # 客户等待时间, 最小
MAX_PATIENCE = 3 # 客户等待时间, 最大

def source(env, number, interval, counter):
    """进程用于生成客户"""
    for i in range(number):
        c = customer(env, 'Customer%02d' % i, counter, time_in_bank=12.0)
        env.process(c)
        t = random.expovariate(1.0 / interval)
        yield env.timeout(t)

def customer(env, name, counter, time_in_bank):
    """一个客户表达为一个协程, 客户到达, 被服务, 然后离开"""

    arrive = env.now
    print('%7.4f %s: Here I am' % (arrive, name))

    with counter.request() as req:
        patience = random.uniform(MIN_PATIENCE, MAX_PATIENCE)
        # 等待柜员服务或者超出忍耐时间离开队伍
        results = yield req | env.timeout(patience)
        wait = env.now - arrive

    if req in results:
        # 到达柜台
        print('%7.4f %s: Waited %6.3f' % (env.now, name, wait))
        tib = random.expovariate(1.0 / time_in_bank)
        yield env.timeout(tib)
        print('%7.4f %s: Finished' % (env.now, name))
    else:
        # 没有服务到位
        print('%7.4f %s: RENEGED after %6.3f' % (env.now, name, wait))

# Setup and start the simulation
print('Bank renege')
random.seed(RANDOM_SEED)
env = simpy.Environment()

# Start processes and run
counter = simpy.Resource(env, capacity=1)
env.process(source(env, NEW_CUSTOMERS, INTERVAL_CUSTOMERS, counter))
env.run()
```

**Example 4**

```python
# python 3.6 with SimPy

"""

工厂工序和传送带

情景:

    一个机器处理物件, 处理完毕后放上传送带, 传送带传送一段时间后到达下个一个机器设备.

    [last_q][machine1] ----[con_belt]----> [next_q][machine2]

"""

import simpy
import random


PROCESS_TIME = 0.5 # 处理时间
CON_BELT_TIME = 3 # 传送带时间
WORKER_NUM = 2 # 每个机器的工人数/资源数
MACHINE_NUM = 2 # 机器数
MEAN_TIME = 0.2 # 平均每个物件的到达时间间距


def con_belt_process(env,
                     con_belt_time,
                     package,
                     next_q):

    """模拟传送带的行为"""

    while True:
        print(f"{round(env.now, 2)} - item: {package} - start moving ")
        yield env.timeout(con_belt_time) # 传送带传送时间
        next_q.put(package)
        print(f"{round(env.now, 2)} - item: {package} - end moving")
        env.exit()

def machine(env: simpy.Environment,
            last_q: simpy.Store,
            next_q: simpy.Store,
            machine_id: str):

    """模拟一个机器, 一个机器就可以同时处理多少物件 取决资源数(工人数)"""

    workers = simpy.Resource(env, capacity=WORKER_NUM)

    def process(item):
        """模拟一个工人的工作进程"""

        with workers.request() as req:
            yield req
            yield env.timeout(PROCESS_TIME)
            env.process(con_belt_process(env, CON_BELT_TIME, item, next_q))
            print(f'{round(env.now, 2)} - item: {item} - machine: {machine_id} - processed')

    while True:
        item = yield last_q.get()
        env.process(process(item))

def generate_item(env,
                  last_q: simpy.Store,
                  item_num: int=100):

    """模拟物件的到达"""

    for i in range(item_num):
        print(f'{round(env.now, 2)} - item: item_{i} - created')
        last_q.put(f'item_{i}')
        t = random.expovariate(1 / MEAN_TIME)
        yield env.timeout(round(t, 1))

if __name__ == '__main__':

    # 实例环境
    env = simpy.Environment()
    # 设备前的物件队列
    last_q = simpy.Store(env)
    next_q = simpy.Store(env)

    env.process(generate_item(env, last_q))

    for i in range(MACHINE_NUM):
        env.process(machine(env, last_q, next_q, machine_id=f'm_{i}'))

    env.run()
```

### 结语

文章暂时结束，文章主要通过代码来展示 SimPy 的仿真能力。

接下来的文章计划是：

* 介绍如何构建一个基于时间动态的人力资源排班表，来模拟不同时段人力的分布，比如流水线上不同岗位在不同的时间段需求的人力资源是不均等，我们可以通过调岗的形式来达到人力资源调优，让人力资源在时间分布上最优。
* 介绍如何一个队列，同时实现时间先后优先和优先级上优先，来模拟比如银行柜台客户排队 vip 进行插队的情景
* 启动一个翻译 SimPy 官方文档的众包计划，希望在国内降低大众学习 SimPy 的语言成本


# Python SimPy 仿真系列 \(2\)



这次文章是关于如何用 [SimPy](https://simpy.readthedocs.io/) 来解决两个仿真需求:

* 如何随时中断恢复 `Process` \(进程\)
* 如何动态设置 `Resource` \(资源\)的数量

相应地这两个需求满足的场景是:

* 仿真过程中, 某一工序被中断, 中断可以依据一个预先设定的时间或者是不确定时间
* 仿真过程中, 人力资源也是依据时间变化, 模拟现实中工人的排班安排

### 回顾资源和进程的概念

`Resource` 和 `Process` 是 SimPy 对人力资源和进程进行抽象的构造. `Resource` 好比一个队列, 其长度就是提前设置好的资源数, 不同的工序就按照时间先后和赋予的优先级进入队列. `Process` 从构造上来说就是一个生成器, 我们可以通过 send 方法传入 `Exception` 对 `Process` 进行打断.

比如某个工序需要占用一个工人, 耗时 30 min 来完成一个进程, 当前所有可以调用的工人数是 10, 代码形式如下:

```python
import simpy
import random


WORKERNUM = 10 # 工人数
PROCESS_TIME = 30 * 60 # 工序耗时, 使用秒作为单位
MEAN_  = 4 * 60 # 平均物件生成时间

def process(env, workers, store):
    """工序"""
    while True:
        with workers.request() as req:
            yield req
            item = yield store.get()
            print(f"{env.now} - {item} - start")
            yield env.timeout(PROCESS_TIME)
            print(f"{env.now} - {item} - done")

def put_item(env, store):
    """每隔一段时间生成一个物品"""
    for i in range(100):
        item = f"{i}_item"
        store.put(item)
        yield env.timeout(random.expovariate(1 / MEAN_))

env = simpy.Environment()
workers = simpy.Resource(env, 10)
store = simpy.Store(env)

env.process(process(env, workers, store))
env.process(put_item(env, store))
env.run()
```

更详细的介绍和资料可以回顾之前的文章 [Python SimPy 仿真系列 \(1\)](https://zhuanlan.zhihu.com/p/31526894)

### `Process` 进程的动态调整

存在以下两种情景:

* 进程随时中断以及恢复
* 按照时间表对进程进行启动或者终止

要区分一件事情, 中断的时候是让当前进程完成后再中断, 还是立即中断. 具体场景可以想象为一个工人被调离当前岗位, 他应该是先完成手头上的工序, 或者他需要停下手头的工作离开工位.

如果是必须实现进程的随时中断, 只能通过 `process.interrupt()` 中断 `process`, 即第一种场景; 假若中断是按照时间表进行, 就可以通过第二种场景, 构建多个不同时间开启的进程来进行模拟.

#### 进程中断的实现

```python
from simpy import interrupt, Environment

env = Environment()

def interrupter(env, victim_proc):
    yield env.timeout(1)
    victim_proc.interrupt('Spam')

def victim(env):
    try:
        yield env.timeout(10)
     except Interrupt as interrupt:
         cause = interrupt.cause
```

#### 多段进程模拟按时间安排的开关

```python
import simpy

PROCESS_TIME = 3

def put_item(env, store):
    for i in range(20):
        yield env.timeout(0.5)
        store.put(f"{i}_item")

def process(i, env, store, start, end):

    yield env.timeout(start)
    while True:
        item = yield store.get()
        # 判断 item 到达时间是否超出本进程关闭时间
        if env.now > end:
            print(f"{env.now} - process {i} - end")
            store.put(item)
            env.exit()
        else:
            print(f"{env.now} - {item} - start")
            yield env.timeout(PROCESS_TIME)
            print(f"{env.now} - {item} - end")

env = simpy.Environment()
store = simpy.Store(env)

env.process(put_item(env, store))

for i, (start, end) in enumerate([(20, 30), (40, 50), (60, 90)]):
    env.process(process(i, env, store, start, end))

env.run()
```

### `Resource` 资源的动态调整

* 资源人数按指定的排版表调配

由于`Resource` 在实例化后, 就没办法修改了. 为了满足在仿真过程中对资源进行修改, 使用了一个反向的思路. 首先所有资源使用 `PriorityResource` 实例, 预先设置一个可以调节的最大资源数, 当需要调节资源数的时候, 使用一个优先级为 `-1` 的 `request` 去占用资源, 而正常的进程默认优先级是 `0`.

通过这样的操作会使得, 我们调节资源的占用进程优先级更高, 正常进程可以调用的资源数会变成 :

`可以调用资源` = `最大资源` - `占用资源`

```python
import simpy

PROCESS_TIME = 2

def put_item(env, store):
    for i in range(20):
        yield env.timeout(0.5)
        store.put(f"{i}_item")

def process(env, store, resource):
    while True:
        item = yield store.get()
        with resource.request() as req:
            yield req
            yield env.timeout(PROCESS_TIME)

def set_resource(env, resource, start_time, end_time):
    """占用资源，模拟资源减少的情况，
    end_time 会出现 np.inf 无穷大，
    simpy 只会用作为排序，可以放在timeout事件里。
    """
    duration = end_time - start_time
    yield env.timeout(start_time)
    with resource.request(priority=-1) as req:
        yield req
        yield env.timeout(duration)

env = simpy.Environment()
store = simpy.Store(env)
res = simpy.PriorityResource(env, 10)

res_time_table = [(10, 20, 5), (20, 30, 6)]
env.process(put_item(env, store))
env.process(process(env, store, res))

for start, end, target_num in res_time_table:
    place_holder = 10 - target_num
    for _ in range(place_holder):
        env.process(set_resource(env, res, start, end))

env.run()
```


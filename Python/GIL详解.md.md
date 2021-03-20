# GIL 详解

## 什么是 GIL


GIL，Global Interpreter Lock，定义如下：

> In CPython, the global interpreter lock, or GIL, is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes at once. This lock is necessary mainly because CPython’s memory management is not thread-safe. (However, since the GIL exists, other features have grown to depend on the guarantees that it enforces.)

翻译过来就是：

> 在 CPython 解释器中，全局解释锁 GIL 是在于执行 Python 字节码时，为了保护访问 Python 对象而阻止多个线程执行的一把互斥锁。这把锁的存在主要是因为 CPython 解释器的内存管理不是线程安全的。然而直到今天 GIL 依旧存在，现在的很多功能已经习惯于依赖它作为执行的保证。


1. GIL 是存在于 CPython 解释器中的，属于解释器层级，而并非属于 Python 的语言特性。也就是说，如果你自己有能力实现一个 Python 解释器，完全可以不使用 GIL
2. GIL 是为了让解释器在执行 Python 代码时，同一时刻只有一个线程在运行，以此保证内存管理是安全的
3. 历史原因，现在很多 Python 项目已经习惯依赖 GIL（开发者认为 Python 就是线程安全的，写代码时对共享资源的访问不会加锁）


## GIL原理
按照 Python 社区的想法，操作系统本身的线程调度已经非常成熟稳定了，没有必要自己搞一套。所以 Python 的线程就是 C 语言的一个 pthread，并通过操作系统调度算法进行调度（例如 Linux 是 CFS）。

Python 2.x 的代码执行是基于 opcode 数量的调度方式，简单来说就是每执行一定数量的字节码，或遇到系统 IO 时，会强制释放 GIL，然后触发一次操作系统的线程调度（当然是否真正进行上下文切换由操作系统自主决定）。

虽然在 Python 3.x 进行了优化，主要是三个方面：将切换颗粒度从基于 opcode 计数改成基于时间片计数（即运行 opcode 的时间）；避免最近一次释放GIL锁的线程再次被立即调度； 新增线程优先级功能（高优先级线程可以迫使其他线程释放所持有的GIL锁）。但是本质仍然保持不变。

但这种线程的调度方式，都会导致同一时刻只有一个线程在运行。而线程在调度时，又依赖系统的 CPU 环境，也就是在单核 CPU 或多核 CPU 下，多线程在调度切换时的成本是不同的。

如果是在单核 CPU 环境下，多线程在执行时，线程 A 释放了 GIL 锁，那么被唤醒的线程 B 能够立即拿到 GIL 锁，线程 B 可以无缝接力继续执行，伪代码如下：

```python
while True:
    acquire GIL
    for i in 1000:
        do something
    release GIL
    /* Give Operating System a chance to do thread scheduling */
```

由于从 release GIL 到 acquire GIL 之间几乎是没有间隙的。如果在在多核 CPU 环境下，当多线程执行时，线程 A 在 CPU0 执行完之后释放 GIL 锁，其他 CPU 上的线程都会进行竞争。

但 CPU0 上的线程 B 极大可能又马上获取到了 GIL，这就导致其他 CPU 上被唤醒的线程，只能眼巴巴地看着 CPU0 上的线程愉快地执行着，而自己只能等待，直到又被切换到待调度的状态，这就会产生多核 CPU 频繁进行线程切换，消耗资源，这种情况也被叫 CPU 颠簸。整个执行流程如下图：

```
Thread1  执行 执行              争抢成功，执行 执行执行  执行      争抢成功，执行...
Thread2           唤醒，开始争抢      线程等待 进入待调度状态  唤醒         等待...
```

这就是多线程在多核 CPU 下，执行效率还不如单线程或单核 CPU 效率高的原因。注意：如果使用多线程运行一个 CPU 密集型任务，那么 Python 多线程是无法提高运行效率的。

但是，如果是一个 IO 密集型的任务，多线程可以显著提高运行效率！

其实原因也很简单，因为 IO 密集型的任务，大部分时间都花在等待 IO 上，并没有一直占用 CPU 的资源，所以并不会像上面的程序那样，进行无效的线程切换。

例如，如果我们想要下载 2 个网页的数据，也就是发起 2 个网络请求，如果使用单线程的方式运行，只能是依次串行执行，其中等待的总耗时是 2 个网络请求的时间之和。而如果采用 2 个线程的方式同时处理，这 2 个网络请求会同时发送，然后同时等待数据返回（IO等待），最终等待的时间取决于耗时最久的线程时间，这会比串行执行效率要高得多。

1. 如果使用多线程运行一个 CPU 密集型任务，那么 Python 多线程是无法提高运行效率的。
2. 如果需要运行 IO 密集型任务，Python 多线程是可以提高运行效率的

## 为什么会有 GIL

这就需要追溯历史原因了。

在 2000 年以前，各个 CPU 厂商为了提高计算机的性能，其努力方向都在提升单个 CPU 的运行频率上，但在之后的几年遇到了天花板，单个 CPU 性能已经无法再得到大幅度提升，所以在 2000 年以后，提升计算机性能的方向便改为向多 CPU 核心方向发展。

为了更有效的利用多核心 CPU，很多编程语言就出现了多线程的编程方式，但也正是有了多线程的存在，随之带来的问题就是多线程之间对于维护数据和状态一致性的困难。

Python 设计者在设计解释器时，可能没有想到 CPU 的性能提升会这么快转为多核心方向发展，所以在当时的场景下，设计一个全局锁是那个时代保护多线程资源一致性最简单经济的设计方案。

而随着多核心时代来临，当大家试图去拆分和去除 GIL 的时候，发现大量库的代码和开发者已经重度依赖 GIL（默认认为 Pythonn 内部对象是线程安全的，无需在开发时额外加锁），所以这个去除 GIL 的任务变得复杂且难以实现。

所以，GIL 的存在更多的是历史原因，在 Python 3 的版本，虽然对 GIL 做了优化，但依旧没有去除掉，Python 设计者的解释是，在去除 GIL 时，会破坏现有的 C 扩展模块，因为这些扩展模块都严重依赖于 GIL，去除 GIL 有可能会导致运行速度会比 Python 2 更慢。

Python 走到现在，已经有太多的历史包袱，所以现在只能背负着它们前行，如果一切推倒重来，想必 Python 设计者会设计得更加优雅一些。

## 如何避免受到GIL的影响

### 用 `multiprocessing` 替代 `Thread`

`multiprocessing` 库的出现很大程度上是为了弥补 `thread` 库因为 GIL 而低效的缺陷。它完整的复制了一套 `thread` 所提供的接口方便迁移。唯一的不同就是它使用了多进程而不是多线程。每个进程有自己的独立的 GIL，因此也不会出现进程之间的 GIL 争抢。

当然 `multiprocessing` 也不是万能良药。它的引入会增加程序实现时线程间数据通讯和同步的困难。就拿计数器来举例子，如果我们要多个线程累加同一个变量，对于 `thread` 来说，申明一个 global 变量，用 `thread.Lock` 的 context 包裹住三行就搞定了。而 `multiprocessing` 由于进程之间无法看到对方的数据，只能通过在主线程申明一个`Queue`，接着进行 `put` 和 `get` 或者用 share memory 的方法。这个额外的实现成本使得本来就非常痛苦的多线程程序编码，变得更加痛苦了。

### 调用 C 库

如果让 Python 在调用 C++ 的代码中中使用线程，那么 C++ 的线程能不能绕过 GIL 的作用呢？

答案是肯定的，因为 GIL 锁的是 Python 解释器，当我们的代码进入到 C++ 中的时候，我们已经不在 Python 解释器中了，这样即使我在 C++ 中声明线程，那也是 C++ 的线程，所以就不会造成无法使用多核的情况。

前面我们知道其实 Python 底层是用 C 写的，所以基本上所以的语法都是基于 C 代码实现加上语法糖来完成的，Python 线程也就是 C 线程，我们能不能模拟一下 Python 来构建 GIL 呢？

当然是可以的，通过 `pybind11` 库去引入  `py::gil\_scoped\_acquire acquire`  和 `py::call\_guard<py::gil\_scoped\_release>()` ，获取 GIL 和释放 GIL。在构建的线程执行函数的前面去获取 GIL，执行完毕再释放 GIL，这样就可以完美的模拟 Python 中的情况了。

所以本质上：GIL 问题其实就是一个死锁的问题，线程获取后不释放锁，导致所有线程相互竞争。使用 C 的解决办法有两个：

1. 使用 Python 的 C 的头文件函数宏 `Py_BEGIN_ALLOW_THREADS`和`Py_END_ALLOW_THREADS`
2. 使用 `pybind11` 库提供的`py::call_guard<py::gil_scoped_release>()`来释放`GIL`


### 换个解释器吧

-   CPython，CPython是官方版本的解释器，这个解释器是使用C语言编写的，也是使用最为广泛的解释器，可以方便地和C/C++的类库进行交互，因此也是最受关注的解释器。

-   Jython，一种由java语言编写的python解释器，是将python编译成Java字节码然后执行的一种解释器，可以方便地和Java的类库进行交互。（不含 GIL）

-   IronPython，将Python代码解释为.Net平台上运行的字节码进行执行，类似Jython解释器，可以方便的和.Net平台上的类库进行交互。（不含GIL）

-   IPython，在交互效果上有所增强，但执行过程和功能方面和CPython是一样的。

-  PyPy，一种使用JIT(just-in-time)技术的编译器，专注于执行速度，对Python代码进行动态编译，从而提高Python的执行速度。PyPy在处理python代码的过程中，一小部分功能的处理和官方的CPython的执行结果是有差异的，如果项目中要使用PyPy来进行执行效率的提升的话，一定要事先了解下PyPy和CPython的区别。







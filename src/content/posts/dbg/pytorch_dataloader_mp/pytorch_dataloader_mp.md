---
title: pytorch Dataloader 多进程与 Sampler
published: 2024-05-15
description: '很可能是调包侠为数不多的读库代码的机会了！'
image: ''
tags: [pytorch]
category: '捉虫'
draft: false 
---

### 问题描述
其实东西没什么问题，是写东西的人比较有问题。

主要还是记一下 Dataloader 这个非常常用的东西具体做了一些什么，在单进程多进程自动手动 batch sampling 方式之下分别会有什么样的行为，否则出了问题单看 traceback 然后两眼一抹黑还是会不禁感到自己很抽象。


### Dataset 和 Sampler
小学的时候我们学过，如果要继承 `torch.utils.data.Dataset` 类，必须要手动重载的两个方法是 `__get_item__` 和 `__len__`。虽然看起来比较理所当然，但 `Dataset` 的设计本身其实只包含一个从 index 到 datum 的映射，也就是 `__get_item__` 所实现的这一功能。它甚至没有硬性要求派生类去重载 `__len__`，也就是说完全可以写出一个能正常索引的 map-style Dataset，而没人知道它长度有多大。

那么究竟是谁需要 `__len__`，这个长度值又是在哪被谁使用了呢？~~答案很显然是程序员本身~~答案很显然写在[库代码](https://pytorch.org/docs/stable/_modules/torch/utils/data/dataset.html#Dataset)当中:
> Subclasses could also optionally overwrite :meth:`__len__`, which is expected to return the size of the dataset by many :class:`~torch.utils.data.Sampler` implementations and the default options of :class:`~torch.utils.data.DataLoader`.

到了这一集，Sampler 正式登场。它做的事情其实比较简单，iterate through 一个 Sampler，可以获得一串索引值，决定了在读入数据时各个 datum 被读入的顺序。

Sampler object 需要传给 Dataloader object 并在其中被使用。在通常情况下，不需要手动指定 Sampler，因为 Dataloader 默认采用 [automatic batching](https://pytorch.org/docs/stable/data.html#automatic-batching-default)，内置一个带 batching 的 Sampler，`torch.utils.data.BatchSampler`。文档中提到 Dataloader 提供一些默认选项，指的就是这个 automatic batching。

至此，在不考虑多进程的情况下，我们已经基本上摸清 Dataloader 到底是怎么 load data 的了。Iterate through 一个 Dataloader 的每一步，Dataloader 会去 iterate through 自己的 Sampler，拿到待取出 data 的索引值，然后索引 Dataset，最终将对应的数据 yield 出来。

### Dataloader 多进程和 Sampler

pytorch 的 DDP is pretty much python multiprocessing 外面封装一层。我们都知道 Dataloader 是一个 iterable，在多进程的情况下，每当 Dataloader 产生一次 iterator，就会同时产生子进程，并将 Dataset 和其他处理数据所需的方法传给子进程。

但这个时候，子进程尚不知道应当取出哪些数据。于是来到了整个过程当中最 tricky 的一部分，Sampler 在亲进程手里，它要为所有的子进程采出 indices，形成一个队列传给子进程，子进程去索引自己上下文的 Dataset 读出数据；这本质上是一个生产者消费者模型。因此，如果某些情况下需要弃用默认 Sampler，就要了解这套过程，以免做多进程的时候卡住。除此之外，pytorch 实现还有很多细节，主要是处理某些进程意外死掉等事故，与 Sampling 流程关系不大。

另外，`torch.utils.data.distributed` 还提供一个 `DistributedSampler` 类，是为了各子进程之间更好地做数据并行而准备的。使用 `DistributedSampler` 可以让各子进程仅持有 Dataset 的一部分，在内存受限的场景下应该会有用。
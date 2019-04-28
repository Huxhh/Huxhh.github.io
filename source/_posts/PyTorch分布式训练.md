---
title: PyTorch分布式训练2
date: 2019-04-28 11:11:10
tags:
- PyTorch
---

# PyTorch分布式训练
分布式训练已经成为如今训练深度学习模型的一个必备工具，但pytorch默认使用单个GPU进行训练，如果想用使用多个GPU乃至多个含有多块GPU的节点进行分布式训练的时候，需要在代码当中进行修改，这里总结一下几种使用pytorch进行分布式训练的方式。

## 环境
本文使用的环境为：

* python =3.7
* pytorch = 1.0
* CUDA = 8.0

## 使用单个GPU
pytorch中`pytorch.cuda`用于设置和运行`CUDA`操作，它会跟踪当前选定的GPU，并且您分配的所有CUDA张量将默认在该设备上创建。所选设备可以使用 `torch.cuda.device` 环境管理器进行更改。
pytorch中想要使用GPU进行模型训练非常简单，首先需要使用代码`torch.cuda.is_available()`判断当前环境是否可以使用GPU，如果返回`False`那么证明GPU不可用，需要检查软件包或驱动等是否安装正确。
当GPU可用时，可以使用`torch.device()`创建一个`torch.device`对象，例如`torch.device('cuda')`或使用`torch.device('cuda:0')`指定GPU，该对象可以将张量移动至GPU上。假设有一个张量`x`，我们可以使用`x.cuda()`或`x.to(device)`的方式将其移动至GPU上，这两种方式可以视作是等价的，并没有太大的区别，Variable与模型同理，也可以使用这样的方式进行移动。接下来按照征程的操作即可以进行训练。

## 单机使用多个GPU
单机使用多个GPU有两种方式，`torch.nn.DataParallel()`与`torch.nn.parallel.DistributedDataParallel`
其中`torch.nn.DataParallel()`智能实现在单机多卡中进行分布式训练，而`torch.nn.parallel.DistributedDataParallel`则是新方法，在单机多卡和多机多卡都可以训练。官方建议使用最新的`torch.nn.parallel.DistributedDataParallel`，因为即使在单机多卡上，新的方法在效率上也要比旧的表现好。
### torch.nn.DataParallel()
使用该方法的方式很简单，假设我们已经构建了一个模型`NET`，则只需要：

```
model = NET()
model = torch.nn.DataParallel()
mode.cuda()
```

将模型分发到各个GPU上，接下来既可以使用多个GPU同时进行训练。

> 值得注意的是，这里torch是把模型输入的batch_size按照GPU的个数进行均分，因此如果希望保持与单个GPU训练时同样的batch_sizze则需要按照GPU的个数n对batch_size进行扩展到batch_size *n

> 还有一点是torch默认输入的第一维是batch_size的大小，因此会直接在第一维上进行分割，因此构造batch数据时需要注意。


### torch.nn.parallel.DistributedDataParallel（）
`torch.nn.parallel.DistributedDataParallel`是pytorch1.0新提供的方法。该计算模型并没有采用主流的Parameter Server结构，而是直接用了Uber Horovod的形式，也是百度开源的RingAllReduce算法。算法细节此处不表。
使用该方法首先需要进行初始化`torch.distributed.init_process_group(backend, init_method='env://', **kwargs)`，其中的参数有：

* backend(str): 后端选择，包括 gloo,nccl,mpi
* init_method(str，optional): 用来初始化包的URL, 我理解是一个用来做并发控制的共享方式
* world_size(int, optional): 参与这个工作的进程数
* rank(int,optional): 当前进程的rank
* group_name(str,optional): 用来标记这组进程名的

其中初始化方式有三种：

1. **TCP initialization**
tcp:// IP组播（要求所有进程都在同一个网络中）比较好理解,   以TCP协议的方式进行不同分布式进程之间的数据交流，需要设置一个端口，不同进程之间公用这一个端口，并且设置host的级别和host的数量。设计两个参数rank和world_size。其中rank为host的编号，默认0为主机，端口应该位于该主机上。world_size为分布式主机的个数。
具体的可以初始化为：`tcp://127.0.0.1:12345`

2. **Shared file-system initialization**
file:// 共享文件系统（要求所有进程可以访问单个文件系统）有共享文件系统可以选择 
提供的第二种方式是文件共享，机器有共享的文件系统，故可以采用这种方式，也避免了基于TCP的网络传输。这里使用方式是使用绝对路径在指定一个共享文件系统下不存在的文件。
具体的：`file://PathToShareFile/MultiNode`

3. **Environment variable initialization**
env:// 环境变量（需要您手动分配等级并知道所有进程可访问节点的地址）默认是这个

```
MASTER_PORT - required; has to be a free port on machine with rank 0
MASTER_ADDR - required (except for rank 0); address of rank 0 node
WORLD_SIZE - required; can be set either here, or in a call to init function
RANK - required; can be set either here, or in a call to init function
```

官方示例：

```
Node 1: (IP: 192.168.1.1, and has a free port: 1234)
 
>>> python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_YOU_HAVE
           --nnodes=2 --node_rank=0 --master_addr="192.168.1.1"
           --master_port=1234 YOUR_TRAINING_SCRIPT.py (--arg1 --arg2 --arg3
           and all other arguments of your training script)
Node 2:
 
>>> python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_YOU_HAVE
           --nnodes=2 --node_rank=1 --master_addr="192.168.1.1"
           --master_port=1234 YOUR_TRAINING_SCRIPT.py (--arg1 --arg2 --arg3
           and all other arguments of your training script)

```

在代码中进行初始化时，需要注意的一点是由于需要创建N个进程分别运行在0到N-1号GPU上，因此需要在代码中手动进行指定代码运行的GPU号，使用如下代码：
> torch.cuda.set_device(i)
其中i是0到N-1中的一个。同时在代码中也应该做如下的指定操作：

```
torch.distributed.init_process_group(backend='nccl', world_size=4, rank=, init_method='...')
model = DistributedDataParallel(model, device_ids=[i], output_device=i)
```

在`torch.distributed.init_process_group()`中word_size与rank参数是必需的。代表了工作的进程数与当前进程的rank。
接下来可以启动进行分布式训练，官方也提供了相应的启动方式。


#### 启动方式
在`torch.distributed`当中提供了一个用于启动的程序`torch.distributed.launch`，此帮助程序可用于为每个节点启动多个进程以进行分布式训练，它在每个训练节点上产生多个分布式训练进程。
这个工具可以用作CPU或者GPU，如果被用于GPU，每个GPU产生一个进程进行训练。

该工具既可以用来做单节点多GPU训练，也可用于多节点多GPU训练。如果是单节点多GPU，将会在单个GPU上运行一个分布式进程，据称可以非常好地改进单节点训练性能。如果用于多节点分布式训练，则通过在每个节点上产生多个进程来获得更好的多节点分布式训练性能。如果有Infiniband接口则加速比会更高。

在单节点分布式训练或多节点分布式训练的两种情况下，该工具将为每个节点启动给定数量的进程（--nproc_per_node）。如果用于GPU培训，则此数字需要小于或等于当前系统上的GPU数量（nproc_per_node），并且每个进程将在从GPU 0到GPU（nproc_per_node - 1）的单个GPU上运行。
启动的命令为：
> python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_YOU_HAVE
           YOUR_TRAINING_SCRIPT.py (--arg1 --arg2 --arg3 and all other
           arguments of your training script

**注意** 使用启动工具进行启动时，会产生`--local_rank`参数，需要在代码中使用如下类似代码处理：

```
import argparser

parser = argparse.ArgumentParser()
parser.add_argument('--local_rank', type=int, ...)
args = parser..parse_args()
```

这一参数的作用是为各个进程分配rank号，因此可以直接使用这个`local_rank`参数作为`torch.distributed.init_process_group()`当中的参数rank，同时也可以作为

```
torch.distributed.init_process_group(backend='nccl', world_size=4, rank=, init_method='...')
model = DistributedDataParallel(model, device_ids=[i], output_device=i)
```

中的i。

## 多机使用多个GPU
相比于单机使用多个GPu，最大的区别在于启动方式的不同，假设有两个节点时：
Node 1: (IP: 192.168.1.1, and has a free port: 1234)

```
>>> python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_YOU_HAVE
           --nnodes=2 --node_rank=0 --master_addr="192.168.1.1"
           --master_port=1234 YOUR_TRAINING_SCRIPT.py (--arg1 --arg2 --arg3
           and all other arguments of your training script)
```

Node 2:

```
>>> python -m torch.distributed.launch --nproc_per_node=NUM_GPUS_YOU_HAVE
           --nnodes=2 --node_rank=1 --master_addr="192.168.1.1"
           --master_port=1234 YOUR_TRAINING_SCRIPT.py (--arg1 --arg2 --arg3
           and all other arguments of your training script)
```

增加的参数有nnodes代表机器即节点的个数，node_rank代表节点的rank，master-addr，master_port代表主节点的地址与端口号用于通信。
至此就介绍完了pytorch的分布式训练的基础内容。


## 参考
https://pytorch.org/docs/master/distributed.html
https://blog.csdn.net/zwqjoy/article/details/89415933
https://github.com/pytorch/pytorch/blob/master/torch/nn/parallel/distributed.py
https://www.jianshu.com/p/be9f8b90a1b8?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=weixin-friends
https://juejin.im/entry/5c5f94fd518825629a779f51





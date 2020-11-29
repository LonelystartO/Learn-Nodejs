# 什么是Event Loop？
Event Loop是Nodejs的一种运行机制。他可以使Nodejs在单线程运行机制下进行非阻塞（non-blocking）的操作。
当执行异步操作时会在后台运行，成功后由内核通知Nodejs执行完毕，并将callback加入到poll阶段的执行队列中，并最终被运行。

# Event Loop执行顺序图

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

* 每个阶段都有各自的FIFO的队列
* 进入到某一个阶段后，会执行各自特定的逻辑，随后会执行队列中的callback
* 当队列用尽或到达执行callback上限后会退出该阶段

# 阶段概述

## timers phase
定时器中的threshold,表示的是callback“可能”执行的时间，并不是实际执行的时间。

```javascript
// event-loop-01.js
const fs = require('fs');


function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;


  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();


  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

结果
```
---------------------------------------
#: node event-loop-01.js
#: 105ms have passed since I was scheduled
```

> 为防止poll阶段starve event loop，libuv设置了最大值来停止poll阶段处理过多的事件。

## pending callbacks
该阶段执行系统操作的callback。例如，TCP类型的错误。

## poll
poll阶段有两个主要功能：
1. 计算是否需要阻塞以及阻塞的时间
2. 处理poll queue中的事件

进入poll阶段后，如果没有timers时，可能会有如下情况：
- 如果poll queue不为空。遍历poll queue中的callback顺序同步执行。直到queue耗尽或者达到系统上限。
- 如果poll queue为空。
    - 有setImmediate()，将会中断poll阶段，进入check阶段执行计划的脚本。
    - 没有setImmediate(),等待callback加入到queue中并立即执行。

当poll queue为空时，event loop检查是否有定时器的threshold到达时间。若果有一个或多个timer准备就绪，event loop会返回timers阶段，并执行timer的callback。

## check

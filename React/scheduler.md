时序图
```
sequenceDiagram
    participant App as 应用程序
    participant Scheduler as React Scheduler
    participant TaskQueue as 任务队列
    participant TimerQueue as 定时器队列
    participant Host as 宿主环境(浏览器/Node)
    
    Note over Scheduler: 初始化
    Scheduler->>Scheduler: 设置 getCurrentTime 函数
    Scheduler->>Scheduler: 初始化任务队列和定时器队列
    
    %% 调度任务
    App->>+Scheduler: unstable_scheduleCallback(priorityLevel, callback, options?)
    Scheduler->>Scheduler: 获取当前时间 currentTime
    Scheduler->>Scheduler: 计算任务开始时间 startTime
    
    alt 有延迟的任务 (startTime > currentTime)
        Scheduler->>Scheduler: 设置 sortIndex = startTime
        Scheduler->>TimerQueue: push(timerQueue, newTask)
        
        alt 是最早的延迟任务
            Scheduler->>Host: requestHostTimeout(handleTimeout, delay)
            Note over Host: 设置定时器
        end
    else 立即执行的任务
        Scheduler->>Scheduler: 计算过期时间 expirationTime
        Scheduler->>Scheduler: 设置 sortIndex = expirationTime
        Scheduler->>TaskQueue: push(taskQueue, newTask)
        
        alt 没有正在执行的回调
            Scheduler->>Host: requestHostCallback()
            Host-->>Scheduler: schedulePerformWorkUntilDeadline()
        end
    end
    Scheduler-->>-App: 返回任务对象
    
    %% 处理定时器回调
    Host->>+Scheduler: handleTimeout(currentTime)
    Scheduler->>Scheduler: isHostTimeoutScheduled = false
    Scheduler->>Scheduler: advanceTimers(currentTime)
    
    loop 检查所有到期的定时器任务
        Scheduler->>TimerQueue: timer = peek(timerQueue)
        alt timer.startTime <= currentTime
            Scheduler->>TimerQueue: pop(timerQueue)
            Scheduler->>TaskQueue: push(taskQueue, timer)
        end
    end
    
    alt 有待执行的任务
        Scheduler->>Host: requestHostCallback()
    else 有未到期的定时器任务
        Scheduler->>Host: requestHostTimeout(handleTimeout, nextDelay)
    end
    Scheduler-->>-Host: 完成定时器处理
    
    %% 执行工作循环
    Host->>+Scheduler: performWorkUntilDeadline()
    Scheduler->>Scheduler: startTime = currentTime
    Scheduler->>+Scheduler: flushWork(currentTime)
    
    Scheduler->>Scheduler: isHostCallbackScheduled = false
    Scheduler->>Scheduler: isPerformingWork = true
    Scheduler->>+Scheduler: workLoop(currentTime)
    
    Scheduler->>Scheduler: advanceTimers(currentTime)
    
    loop 处理任务队列中的任务
        Scheduler->>TaskQueue: currentTask = peek(taskQueue)
        
        alt 应该让出主线程
            Note over Scheduler: 时间片用尽，让出控制权
            Scheduler-->>Scheduler: break
        else 继续执行任务
            Scheduler->>Scheduler: 获取任务回调 callback
            Scheduler->>App: 执行回调 callback(didUserCallbackTimeout)
            App-->>Scheduler: 返回结果 continuationCallback
            
            alt 返回了延续回调
                Scheduler->>Scheduler: currentTask.callback = continuationCallback
                Scheduler-->>Scheduler: return true (有更多工作)
            else 任务完成
                Scheduler->>TaskQueue: pop(taskQueue)
            end
        end
    end
    
    alt 还有更多任务
        Scheduler-->>Scheduler: return true
    else 没有更多任务，但有定时器任务
        Scheduler->>Host: requestHostTimeout(handleTimeout, delay)
        Scheduler-->>Scheduler: return false
    else 没有任何任务
        Scheduler-->>Scheduler: return false
    end
    
    Scheduler-->>-Scheduler: 完成工作循环
    
    Scheduler->>Scheduler: isPerformingWork = false
    Scheduler-->>-Scheduler: 完成工作刷新
    
    alt hasMoreWork
        Scheduler->>Host: schedulePerformWorkUntilDeadline()
        Note over Host: 安排下一次工作
    else 没有更多工作
        Scheduler->>Scheduler: isMessageLoopRunning = false
    end
    Scheduler-->>-Host: 完成本次工作
    
    %% 取消任务
    App->>Scheduler: unstable_cancelCallback(task)
    Scheduler->>Scheduler: task.callback = null
    Note over Scheduler: 任务仍在队列中，但不会执行
    
    %% 优先级管理
    App->>+Scheduler: unstable_runWithPriority(priorityLevel, eventHandler)
    Scheduler->>Scheduler: 保存当前优先级
    Scheduler->>Scheduler: 设置新优先级
    Scheduler->>App: 执行 eventHandler()
    App-->>Scheduler: 返回结果
    Scheduler->>Scheduler: 恢复之前的优先级
    Scheduler-->>-App: 返回结果
```

上面的时序图展示了 React Scheduler 的主要执行流程，包括：

1. 初始化过程 ：设置时间获取函数和初始化队列
2. 任务调度 ：通过 unstable_scheduleCallback 添加任务到队列
3. 定时器处理 ：处理延迟任务的定时器回调
4. 工作循环 ：执行任务队列中的任务，并在适当时机让出控制权
5. 任务取消 ：通过 unstable_cancelCallback 取消任务
6. 优先级管理 ：通过 unstable_runWithPriority 在特定优先级下执行代码
Scheduler 的核心是通过时间分片和优先级调度，在不阻塞主线程的情况下执行任务，这是 React 并发模式的基础


unstable_scheduleCallback 和 workLoop 之间是怎么串起来的？
这两个函数之间的连接涉及几个关键步骤：

1. 调度任务 ：当调用 unstable_scheduleCallback 时，它会创建一个任务对象并将其添加到任务队列中
2. 请求宿主回调 ：如果没有正在执行的回调， unstable_scheduleCallback 会调用 requestHostCallback() 函数
3. 宿主环境调度 ： requestHostCallback() 会通过 schedulePerformWorkUntilDeadline() 在宿主环境（浏览器或Node.js）中安排一个回调
4. 执行工作 ：当宿主环境准备好执行任务时，它会调用 performWorkUntilDeadline() ，这个函数会调用 flushWork()
   1. 工作循环 ： flushWork() 函数会调用 workLoop() ，开始处理任务队列中的任务
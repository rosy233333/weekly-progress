# arceos和rcore-tutorial任务调度模块分析

时间：2024/6/13

## arceos的任务调度模块提供的接口

### 初始化

### 获取当前任务

### 当前任务（线程）控制

设置当前任务优先级；

sleep、yield_now、exit；

### 创建任务

### TaskInner相关操作

获取id和name

join（当前任务等待该TaskInner执行完成）

### WaitQueue相关操作

wait系列（当前任务等待在该队列中，直到被其它线程通知/达到预定的等待时间，再根据是否有唤醒条件/唤醒条件是否达成决定唤醒还是继续等待）

notify系列（通知其中的一个任务/所有任务/特定任务）

## rCore-tutorial的任务调度模块提供的接口

### 当前任务（进程）控制

suspend_current_and_run_next、block_current_and_run_next、exit_current_and_run_next

### 信号操作

check_signals_of_current、current_add_signal

### 任务管理数据结构

add_task、pid2process、remove_from_pid2process、wakeup_task

### 处理器相关数据结构

current_process、current_task、schedule（切换到特定任务）、
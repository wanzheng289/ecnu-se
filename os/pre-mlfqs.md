# `mlfqs-*`
## 一、查看文档
在最后一部分中，文档要求实现类似于4.4BSD调度程序的多级反馈队列调度程序，以减少在系统上运行作业的平均响应时间。这个高级调度程序根据优先级选择要运行的线程，但是不进行优先捐赠。默认情况下使用普通优先级调度，但选择带有-mlfqs内核选项后就切换到4.4BSD调度程序，线程不再直接控制自己的优先级，而是由调度器自动计算
### BSD调度器
这种类型的调度程序维护多个准备运行的线程队列，其中每个队列保存具有不同优先级的线程。在任何给定时间，调度程序都会从优先级最高的非空队列中选择一个线程。如果优先级最高的队列包含多个线程，则它们将按RR时间片运行。
#### 1.nice值
①介绍
每个线程有一个整数nice值，用于确定线程对其他线程的nice程度。nice=0不会影响线程优先级；nice＞0（最大值为20）会降低线程的优先级，并导致它放弃一些原本会接收的CPU时间；nice＜0（最小值为-20）往往会占用其他线程的CPU时间。初始线程以零值开始，其他线程以从其父级继承的nice值开始。
②要求
在`threads/thread.c`中实现2个函数
（1）`int thread_get_nice(void)`，用于返回当前线程的nice值。
（2）`void thread_set_nice(int new_nice)`，将当前线程的nice值设置为new_nice，并根据新值重新计算线程的优先级。如果正在运行的线程不再具有最高优先级，则调用yield让出CPU。
#### 2.计算优先级
调度程序有64个优先级，因此有64个就绪队列，编号为0（PRI_MIN）到63（PRI_MAX）。较低的数字对应较低的优先级，因此优先级0是最低优先级，优先级63是最高优先级。线程优先级最初在线程初始化时计算。对于每个线程，它还每四个时钟刻度重新计算一次。无论哪种情况，它都由以下公式确定：
$priority=PRI_{MAX}-(recent_{cpu}/4)-(nice*2)$
$recent_{cpu}$：线程最近使用的CPU时间的估计值
结果应向下舍入到最接近的整数，计算出的优先级始终调整为位于PRI_MIN到PRI_MAX的有效范围内。
#### 3.计算`recent_cpu`
recent_cpu表示每个进程最近收到了多少CPU时间，越近越重要。其初始值在创建的第一个线程中为0，在其它新线程中为父线程的值。每次发生计时器中断时，除非空闲线程正在运行，否则仅对正在运行的线程的recent_cpu递增1。此外，使用以下公式为每个线程（无论是正在运行、就绪还是阻塞）每秒重新计算一次recent_cpu的值：
$recent_{cpu}=(2*load_avg)/(2*load_{avg}+1)*recent_{cpu}+nice$
$load_{avg}$：准备运行的线程数的移动平均值
当`timer_ticks () % TIMER_FREQ == 0`时更新recent_cpu
需要实现`threads/thread.c`中的`int thread_get_recent_cpu(void)`函数来返回当前线程recent_cpu值的100倍，四舍五入到最接近的整数。
#### 4.计算`load_avg`
系统负载平均值load_avg是系统范围的，用于估计过去一分钟内准备运行的平均线程数。在系统启动时，它被初始化为0。此后，每秒更新一次：
$load_{avg}=(59/60)*load_{avg}+(1/60)*ready_{threads}$
$ready_{threads}$：在更新时正在运行或准备运行的线程数（不包括空闲线程）
当`timer_ticks () % TIMER_FREQ == 0`时更新load_avg
需要实现`threads/thread.c`中的`int thread_get_load_avg(void)`来返回当前系统负载平均值的100倍，四舍五入至最接近的整数。
#### 5.实现调度器所需的计算
①每个线程在它直接控制的值都介于-20到20之间。每个线程还有一个优先级，从0（PRI_MIN）到63（PRI_MAX），每四次用以下公式重新计算：
$priority=PRI_{MAX}-(recent_{cpu}/4)-(nice*2)$
②recent_cpu衡量线程最近接收到的CPU时间。每计时一滴，运行线程就会recent_cpu增加1。每秒一次，每个线程的recent_cpu都被更新如下：
$recent_{cpu}=(2*load_avg)/(2*load_{avg}+1)*recent_{cpu}+nice$
③load_avg估算过去一分钟内平均准备运行的线程数。启动时初始化为0，并每秒重新计算一次，具体如下：
$load_{avg}=(59/60)*load_{avg}+(1/60)*ready_{threads}$
#### 6.不动点实算数
在上述公式中，优先级、nice值和ready_threads是整数，但recent_cpu和load_avg是实数，而Pintos内核不支持浮点运算，因此对实数的计算必须用整数进行模拟，也就是定点数。
定点数的思想是把一个有符号32位整数的最低的14位当成小数部分，这样一个整数x就代表真实数$\frac{x}{2^{14}}$。
比如，在计算$load_{avg}$中会用到$\frac{59}{60}$，那么就转换为$\frac{59}{60}*2^{14}=16110$。
文档中给出了在c中实现不动点算术运算的方法，其中$f=2^{14}=1<<14$：

| 操作                                                             | 功能               |
| -------------------------------------------------------------- | ---------------- |
| $n*f$                                                          | 将n转换为不动点         |
| $x/f$                                                          | 将x转换为整数（四舍五入至零）  |
| `(x + f / 2) / f` if `x >= 0`<br>`(x - f / 2) / f` if `x <= 0` | 将x转换为整数（四舍五入至最近） |
| $x+y$                                                          | 加x和y             |
| $x-y$                                                          | 从x中减去y           |
| $x+n*f$                                                        | 加x和n             |
| $x-n*f$                                                        | 从x中减去n           |
| `((int64_t) x) * y / f`                                        | 将x乘以y            |
| $x*n$                                                          | 将x乘以n            |
| `((int64_t) x) * f / y`                                        | 将x除以y            |
| $x/n$                                                          | 将x除以n            |

## 二、分析源代码&修改
### 1.实现不动点实算数
利用文档给出的公式，新增`fixed_point.h`库：
``` c
#ifndef __THREAD_FIXED_POINT_H
#define __THREAD_FIXED_POINT_H

typedef int fixed_t;//用32位有符号整数表示定点数
#define FP_LOW 14//低14位表示小数

#define FP_CONST(N) ((fixed_t)(N<<FP_LOW))//n*f，将n转换为不动点
#define FP_INT(X) (X>>FP_LOW)//x/f，将x转换为整数（四舍五入至零）
#define FP_ROUND(X) (X>=0?((X+(1<<(FP_LOW-1)))>>FP_LOW):((X-(1<<(FP_LOW-1)))>>FP_LOW))//将x转换为整数（四舍五入至最近），如果x>=0则(x+f/2)/f，反之则(x-f/2)/f
#define FP_ADD(X,Y) (X+Y)//加x和y
#define FP_SUB(X,Y) (X-Y)//从x中减去y
#define FP_ADD_MIX(X,N) (X+(N<<FP_LOW))//加x和n
#define FP_SUB_MIX(X,N) (X-(N<<FP_LOW))//从x中减去n
#define FP_MULT(X,Y) ((fixed_t)(((int64_t)X)*Y>>FP_LOW))//将x乘以y
#define FP_MULT_MIX(X,N) (X*N)//将x乘以n
#define FP_DIV(X,Y) ((fixed_t)((((int64_t)X)<<FP_LOW)/Y))//将x除以y
#define FP_DIV_MIX(X,N) (X/N)//将x除以n

#endif /* thread/fixed_point.h */
```
### 2.修改线程结构体
将文档中的新增字段添加到thread结构体：
``` c
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */
    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */
#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif
    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */
    int64_t block_ticks;
    /*新增：*/
    int original_priority;//线程最初的优先值
    struct list locks_held_by_thread;//记录被线程持有的锁的列表
    struct lock* lock_waited_by_thread;//指向该线程正在等待的锁
    int nice;//nice值
    fixed_t recent_cpu;
  };
```
并添加到线程初始化函数`init_thread()`中
``` c
static void
init_thread (struct thread *t, const char *name, int priority)
{
  enum intr_level old_level;
  ASSERT (t != NULL);
  ASSERT (PRI_MIN <= priority && priority <= PRI_MAX);
  ASSERT (name != NULL);
  memset (t, 0, sizeof *t);
  t->status = THREAD_BLOCKED;
  strlcpy (t->name, name, sizeof t->name);
  t->stack = (uint8_t *) t + PGSIZE;
  t->priority = priority;
  t->magic = THREAD_MAGIC;
  
  /*初始化新增字段*/
  t->original_priority=priority;
  list_init(&t->locks_held_by_thread);
  t->lock_waited_by_thread=NULL;
  t->nice=0;
  t->recent_cpu=FP_CONST(0);
  
  old_level = intr_disable ();
  list_push_back (&all_list, &t->allelem);
  // list_insert_ordered(&all_list,&t->allelem,(list_less_func*)&compareByPriority,NULL);
  intr_set_level (old_level);
}
```
由于文档要求`load_avg`是系统范围的，因此单独将其作为全局变量添加到`thread.c`：
``` c
fixed_t load_avg;
```
并在`thread_start()`中初始化：
``` c
/* Starts preemptive thread scheduling by enabling interrupts.
   Also creates the idle thread. */
void
thread_start (void)
{
  /* Create the idle thread. */
  struct semaphore idle_started;
  sema_init (&idle_started, 0);
  thread_create ("idle", PRI_MIN, idle, &idle_started);
  load_avg = FP_CONST (0);//在线程创建后初始化load_avg
  /* Start preemptive thread scheduling. */
  intr_enable ();
  /* Wait for the idle thread to initialize idle_thread. */
  sema_down (&idle_started);
}
```
### 3.修改函数
#### ①按文档的公式进行计算
（1）计算并更新优先级
``` c
void thread_mlfqs_update_priority(struct thread *t)
{
  if(t==idle_thread)//空闲线程的优先级不更新
  {
	  return;
  }
  ASSERT(thread_mlfqs);//断言处于mlfq队列
  ASSERT(t!=idle_thread);//断言当前线程非空
  t->priority=FP_INT(FP_SUB_MIX(FP_SUB(FP_CONST(PRI_MAX),FP_DIV_MIX(t->recent_cpu,4)),2*t->nice));//计算并更新优先级
  if(t->priority<PRI_MIN)//优先级不能小于最小优先级
  {
	  t->priority=PRI_MIN;
  }
  if(t->priority>PRI_MAX)//优先级不能大于最大优先级
  {
	  t->priority=PRI_MAX;
  }
}
```
（2）计算并更新`recent_cpu`和`load_avg`
``` c
void thread_mlfqs_update_recent_cpu_and_load_avg(void)
{
  ASSERT(thread_mlfqs);//断言处于mlfq队列
  ASSERT(intr_context());//断言处于中断上下文中
  size_t ready_threads=list_size(&ready_list);//ready队列的进程-可运行进程
  if(thread_current()!=idle_thread)//正在运行的线程
  {
	  ready_threads++;//算作可运行进程
  }
  load_avg=FP_ADD(FP_DIV_MIX(FP_MULT_MIX(load_avg, 59),60),FP_DIV_MIX(FP_CONST(ready_threads),60));//按公式计算load_avg并更新
  /*计算recent_cpu*/
  struct thread *t;
  struct list_elem *e=list_begin(&all_list);
  for (;e!=list_end(&all_list);e=list_next(e))//遍历处于所有状态的线程
  {
    t=list_entry(e,struct thread,allelem);
    if(t!=idle_thread)//跳过空闲线程
    {
      t->recent_cpu=FP_ADD_MIX(FP_MULT (FP_DIV(FP_MULT_MIX(load_avg,2),FP_ADD_MIX(FP_MULT_MIX(load_avg,2),1)),t->recent_cpu),t->nice);//计算并更新recent_cpu
      thread_mlfqs_update_priority(t);//用更新后的recent_cpu和load_avg更新priority
    }
  }
}
```
（3）更新正在运行的线程的`recent_cpu`，每tick加1
``` c
void thread_mlfqs_increase_recent_cpu_per_tick(void)
{
  ASSERT(thread_mlfqs);//断言处于mlfq队列
  ASSERT(intr_context());//断言在中断上下文中
  struct thread *current_thread=thread_current();//指向当前正在运行的线程
  if(current_thread==idle_thread)//跳过空闲线程
  {
	  return;
  }
  current_thread->recent_cpu=FP_ADD_MIX(current_thread->recent_cpu,1);//+1
}
```
#### ②更新`timer_interrupt()`
在完成alarm部分的测试后，修改后的`timer_interrupt()`只实现了RR和优先级调度
``` c
/* Timer interrupt handler. */
static void timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_tick ();
  thread_foreach(blocked_thread_check,NULL);//每次中断都对所有线程调用blocked_thread_check()，更新睡眠倒计时
}
```
而文档要求4.4BSD需要每4个tick增加一次priority，每秒更新一下load_avg和recent_cpu，因此需要在时钟中断中实现定期更新：
``` c
static void timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;//tick计数
  enum intr_level old_level=intr_disable();//关中断
  if(thread_mlfqs)//选择--mlfqs选项→启动BSD
  {
    thread_mlfqs_increase_recent_cpu_per_tick();//当前正在运行的线程，每tick，recent_cpu+1
    if(ticks%TIMER_FREQ==0)//每秒更新一次recent_cpu和load_avg
    {
	    thread_mlfqs_update_recent_cpu_and_load_avg();
    }
    else if(ticks%4==0)//每4个tick更新优先级
    {
	    thread_mlfqs_update_priority(thread_current());
    }
  }
  thread_foreach(blocked_thread_check,NULL);//每次中断都对所有线程调用blocked_thread_check()，更新睡眠倒计时
  intr_set_level(old_level);//恢复调用前的中断状态
  thread_tick();//当前线程中断返回后让出CPU
}
```
#### ④补充文档要求的函数
（1）`thread_get_nice()`：返回当前线程的nice值
``` c
int thread_get_nice(void)
{
  return thread_current()->nice;
}
```
（2）`thread_set_nice()`：将当前线程的nice值设置为new_nice，并根据新值重新计算线程的优先级。如果正在运行的线程不再具有最高优先级，则调用yield让出CPU
``` c
void thread_set_nice(int nice)
{
  thread_current()->nice=nice;//更新nice值
  thread_mlfqs_update_priority(thread_current());//更新优先级
  thread_yield();
}
```
（3）`thread_get_recent_cpu()`：返回当前系统负载平均值的100倍，四舍五入至最接近的整数
``` c
int thread_get_recent_cpu(void)
{
  return FP_ROUND(FP_MULT_MIX(thread_current()->recent_cpu,100));
}
```
（4）`thread_get_load_avg()`：返回当前系统负载平均值的100倍，四舍五入至最接近的整数
``` c
int thread_get_load_avg(void)
{
  return FP_ROUND(FP_MULT_MIX(load_avg,100));
}
```
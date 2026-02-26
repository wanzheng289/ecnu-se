# （一）make check截图
![[Pasted image 20251128114415.png]]
# （二）`alarm-priority`
## 一、查看文档
文档在`2.2.2 Alarm Clock`部分给出的任务是重新实现在文件`devices/timer.c`中定义的`timer_sleep()`函数。`timer_sleep()`的功能是让当前线程暂停执行，直到时间至少过去x个时钟节拍。在系统不是空闲的情况下，线程不需要在恰好第x个时钟节拍被唤醒，而只需要在大约合适的时间被放回就绪队列即可。该函数的问题在于它采用忙等待，也就是线程会在一个循环中不断检查当前时间，并持续调用`thread_yield()`，直到足够的时间过去。所以需要重新实现`timer_sleep()`函数来避免忙等待。
## 二、`timer_sleep()`函数
``` c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void timer_sleep (int64_t ticks)
{
  int64_t start = timer_ticks ();
  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks)
    thread_yield ();
}
```
### 1.第1行：调用函数`timer_ticks()`
``` c
/* Returns the number of timer ticks since the OS booted. */
int64_t timer_ticks (void)
{
  enum intr_level old_level = intr_disable ();//关闭中断并保存调用前的中断状态
  int64_t t = ticks;//读取时钟节拍
  intr_set_level (old_level);//恢复调用前的中断状态
  return t;//返回从内核启动以来已经过去的时钟节拍数
}
```
#### ①第1行：声明一个`intr_level`类型的变量`old_level`，并通过`intr_disable()`赋值
 禁止当前行为被中断，保存禁止被中断前的中断状态（用old_level储存）
``` c
/* Interrupts on or off? */
enum intr_level
  {
    INTR_OFF,             /* Interrupts disabled. */
    INTR_ON               /* Interrupts enabled. */
  };
```
`intr_level`是一个枚举类型，用于表示当前中断是否开启
- `INTR_OFF`表示中断被关闭
- `INTR_ON`表示中断开启
``` c
/* Disables interrupts and returns the previous interrupt status. */
enum intr_level intr_disable (void)//获取了当前的中断状态，然后将当前中断状态改为不能被中断，然后返回执行之前的中断状态
{
  enum intr_level old_level = intr_get_level ();
  /* Disable interrupts by clearing the interrupt flag.
     See [IA32-v2b] "CLI" and [IA32-v3a] 5.8.1 "Masking Maskable
     Hardware Interrupts". */
  asm volatile ("cli" : : : "memory");//将标志寄存器的IF位置为0，关闭中断
  return old_level;
}
```
该函数首先调用`intr_get_level()`来记录调用前的中断状态，然后关闭中断，最后返回调用前的中断状态
（1）`intr_get_level()`
``` c
/* Returns the current interrupt status. */
enum intr_level intr_get_level (void)
{
  uint32_t flags;
  /* Push the flags register on the processor stack, then pop the
     value off the stack into `flags'.  See [IA32-v2b] "PUSHF"
     and "POP" and [IA32-v3a] 5.8.1 "Masking Maskable Hardware
     Interrupts". */
  asm volatile ("pushfl; popl %0" : "=g" (flags));//把标志寄存器压入栈上，然后把值pop到代表标志寄存器IF位的flags上
  return flags & FLAG_IF ? INTR_ON : INTR_OFF;//通过判断flags来返回当前中断状态
}
```
最后1行使用掩码`FLAG_IF`来取出IF位
- 如果IF=1→返回`INTR_ON`
- 如果IF=0→返回`INTR_OFF`
也就是该函数通过`INTR_ON`/`INTR_OFF`来返回CPU当前的中断状态
#### ②第2行：用t获取了一个全局变量ticks
``` c
/* Number of timer ticks since OS booted. */
static int64_t ticks;
```
`ticks`用于记录从内核启动以来已经过去了多少个时钟节拍
#### ③第3行：调用`intr_set_level`函数
``` c
/* Enables or disables interrupts as specified by LEVEL and
   returns the previous interrupt status. */
enum intr_level intr_set_level (enum intr_level level)
{
  return level == INTR_ON ? intr_enable () : intr_disable ();
}
```
- 如果`level==INTR_ON`→调用`intr_enable ()`将中断打开，并返回调用前的中断状态
- 如果`level==INTR_OFF`→调用`intr_disable ()`将中断关闭，并返回调用前的中断状态
（1）`intr_enable ()`
``` c
/* Enables interrupts and returns the previous interrupt status. */
enum intr_level intr_enable (void)
{
  enum intr_level old_level = intr_get_level ();//CPU当前的中断状态
  ASSERT (!intr_context ());//断言当前不在中断上下文中，防止在处理中断时调用intr_enable再次开启中断
  /* Enable interrupts by setting the interrupt flag.
     See [IA32-v2b] "STI" and [IA32-v3a] 5.8.1 "Masking Maskable
     Hardware Interrupts". */
  asm volatile ("sti");//将标志寄存器的IF位置为1，开启中断
  return old_level;
}
```
与`intr_disable()`类似，开启中断并返回调用前的中断状态
`intr_context()`
``` c
/* Returns true during processing of an external interrupt
   and false at all other times. */
bool intr_context (void)
{
  return in_external_intr;//in_external_intr
}
```
返回了是否外中断的标志
`in_external_intr`
``` c
static bool in_external_intr;   /* Are we processing an external interrupt? */
```
- 值为`true`表示当前正在处理一个外部中断
- 值为`false`表示普通执行状态
#### ④第4行：返回t

综上，`timer_ticks()`可保证读取时钟节拍的原子性
### 2.第2行：`ASSERT (intr_get_level () == INTR_ON);`
断言当线程调用`timer_sleep()`时，内核的中断状态一定是开启的，防止在中断关闭的情况下调用`timer_sleep()`，导致死锁
### 3.第3行：调用`timer_elapsed()`
``` c
/* Returns the number of timer ticks elapsed since THEN, which
   should be a value once returned by timer_ticks(). */
int64_t timer_elapsed (int64_t then)
{
  return timer_ticks () - then;
}
```
该函数返回了当前时间到`then`的时间间隔。
那么`while (timer_elapsed (start) < ticks)`就是一个忙等待，循环条件是从调用`timer_sleep()`的时间`start`到当前时间还没有经过足够的ticks
### 4.第4行：调用`thread_yield ()`
``` c
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void thread_yield (void)
{
  struct thread *cur = thread_current ();//获取指向当前正在运行的线程的指针
  enum intr_level old_level;//保存调用前的中断状态
  ASSERT (!intr_context ());//断言当前不处于中断处理中，防止在中断处理上下文中再次调用thread_yield，造成死锁
  old_level = intr_disable ();//关闭中断，返回调用前的中断状态
  if (cur != idle_thread)//如果线程不为空
    //list_push_back(&ready_list, &cur->elem);
    list_insert_ordered(&ready_list,&cur->elem,compareByPriority,NULL);//将当前线程按优先级插入ready队列
  cur->status = THREAD_READY;//将线程状态从正在运行→就绪
  schedule ();//选择下一个要运行的线程并切换
  intr_set_level (old_level);//恢复调用前的中断状态
}
```
#### ①第1行：声明一个`thread`类型的变量`*cur`，并通过`thread_current ()`赋值
``` c
/* Returns the running thread.
   This is running_thread() plus a couple of sanity checks.
   See the big comment at the top of thread.h for details. */
struct thread * thread_current (void)
{
  struct thread *t = running_thread ();//返回当前线程起始指针
  /* Make sure T is really a thread.
     If either of these assertions fire, then your thread may
     have overflowed its stack.  Each thread has less than 4 kB
     of stack, so a few big automatic arrays or moderate
     recursion can cause stack overflow. */
  ASSERT (is_thread (t));//断言t是合法线程
  ASSERT (t->status == THREAD_RUNNING);//断言t正在运行
  return t;
}
```
（1）`running_thread ()`
``` c
/* Returns the running thread. */
struct thread * running_thread (void)
{
  uint32_t *esp;
  /* Copy the CPU's stack pointer into `esp', and then round that
     down to the start of a page.  Because `struct thread' is
     always at the beginning of a page and the stack pointer is
     somewhere in the middle, this locates the curent thread. */
  asm ("mov %%esp, %0" : "=g" (esp));//从CPU的寄存器esp获取当前栈顶地址
  return pg_round_down (esp);//向下取整到页边界
}
```
每个线程的内核栈占一个4kB页，并把`struct thread`放在页的最开始，而`esp`一定在该页的中间，所以将地址对齐到页首即可获取`struct thread`的地址，返回当前线程的起始指针
`pg_round_down()`
``` c
/* Round down to nearest page boundary. */
static inline void *pg_round_down (const void *va) {
  return (void *) ((uintptr_t) va & ~PGMASK);
}
```
把`*va`的低12位清零，向下对齐到4kB页边界，并将`PGMASK`低12位为1、高位为0.用`& ~PGMASK`去除低位偏移，返回该页面线程的最开始指针
（2）`is_thread()`
``` c
/* Returns true if T appears to point to a valid thread. */
static bool is_thread (struct thread *t)
{
  return t != NULL && t->magic == THREAD_MAGIC;
}
```
如果指针不为空&&magic字段=固定魔数`THREAD_MAGIC`→指针合法
`THREAD_MAGIC`
``` c
/* Random value for struct thread's `magic' member.
   Used to detect stack overflow.  See the big comment at the top
   of thread.h for details. */
#define THREAD_MAGIC 0xcd6abf4b
```

（3）`thread_status`：正在运行/就绪/被阻塞/即将被销毁
``` c
/* States in a thread's life cycle. */
enum thread_status
  {
    THREAD_RUNNING,     /* Running thread. */
    THREAD_READY,       /* Not running but ready to run. */
    THREAD_BLOCKED,     /* Waiting for an event to trigger. */
    THREAD_DYING        /* About to be destroyed. */
  };
```
综上，`thread_current`返回了当前线程的起始指针位置
#### ②第2行：声明一个`intr_level`类型的变量`old_level`，表示当前中断是否开启

#### ③第3行：断言当前不处于中断处理中，防止在中断处理上下文中再次调用thread_yield，造成死锁

#### ④第4行：将中断状态关闭，并返回调用前的中断状态

#### ⑤第5行：如果如果线程不为空
``` c
/* Idle thread. */
static struct thread *idle_thread;
```
该指针指向空闲线程
#### ⑥第6行：调用`list_push_back`函数（实际已经在上一次作业时修改为了调用`list_insert_ordered()`函数来实现按优先级插入ready队列，此处仍分析修改前的源代码）
将线程插入到ready队列的队尾
``` c
/* Inserts ELEM at the end of LIST, so that it becomes the back in LIST. */
void list_push_back (struct list *list, struct list_elem *elem)
{
  list_insert (list_end (list), elem);
}
```
（1）`list_elem`
``` c
/* List element. */
struct list_elem
  {
    struct list_elem *prev;     /* Previous list element. */
    struct list_elem *next;     /* Next list element. */
  };
```
双向链表节点，由前驱指针+后继指针组成
（2）`list`
``` c
/* List. */
struct list
  {
    struct list_elem head;      /* List head. */
    struct list_elem tail;      /* List tail. */
  };
```
伪头节点+伪尾节点
（3）`list_end()`
``` c
/* Returns LIST's tail.
   list_end() is often used in iterating through a list from
   front to back.  See the big comment at the top of list.h for
   an example. */
struct list_elem * list_end (struct list *list)
{
  ASSERT (list != NULL);
  return &list->tail;
}
```
返回尾节点
（4）`list_insert()`
``` c
/* Inserts ELEM just before BEFORE, which may be either an
   interior element or a tail.  The latter case is equivalent to
   list_push_back(). */
void list_insert (struct list_elem *before, struct list_elem *elem)
{
  ASSERT (is_interior (before) || is_tail (before));
  ASSERT (elem != NULL);
  elem->prev = before->prev;
  elem->next = before;
  before->prev->next = elem;
  before->prev = elem;
}
```
将elem插入before前
#### ⑦第7行：将线程状态从正在运行→就绪

#### ⑧第8行：调用`schedule ()`函数-调度
``` c
/* Schedules a new process.  At entry, interrupts must be off and
   the running process's state must have been changed from
   running to some other state.  This function finds another
   thread to run and switches to it.
   It's not safe to call printf() until thread_schedule_tail()
   has completed. */
static void schedule (void)
{
  struct thread *cur = running_thread ();//获取当前进程
  struct thread *next = next_thread_to_run ();//获取下一个要运行的线程
  struct thread *prev = NULL;
  ASSERT (intr_get_level () == INTR_OFF);//断言中断关闭
  ASSERT (cur->status != THREAD_RUNNING);//断言当前线程不是正在运行
  ASSERT (is_thread (next));//断言下一个线程是合法线程
  if (cur != next)//如果当前线程和下一个要运行的线程不是同一个线程
    prev = switch_threads (cur, next);//保存当前线程的寄存器到其struct thread*中→千幻栈指针esp到next的内核栈→恢复next的寄存器内容→返回到next中switch_threads的返回点→返回cur到prev
  thread_schedule_tail (prev);
}
```
（1）`running_thread ()`
``` c
/* Returns the running thread. */
struct thread * running_thread (void)
{
  uint32_t *esp;
  /* Copy the CPU's stack pointer into `esp', and then round that
     down to the start of a page.  Because `struct thread' is
     always at the beginning of a page and the stack pointer is
     somewhere in the middle, this locates the curent thread. */
  asm ("mov %%esp, %0" : "=g" (esp));
  return pg_round_down (esp);//提取当前运行线程的struct thread*
}
```

（2）`next_thread_to_run ()`
``` c
/* Chooses and returns the next thread to be scheduled.  Should
   return a thread from the run queue, unless the run queue is
   empty.  (If the running thread can continue running, then it
   will be in the run queue.)  If the run queue is empty, return
   idle_thread. */
static struct thread *
next_thread_to_run (void)
{
  if (list_empty (&ready_list))//如果就绪队列为空
    return idle_thread;
  else
    return list_entry (list_pop_front (&ready_list), struct thread, elem);//弹出就绪队列优先级最高的元素，并把list_elem*转回外层的struct thread*
}
```

`list_empty()`
``` c
/* Returns true if LIST is empty, false otherwise. */
bool list_empty (struct list *list)
{
  return list_begin (list) == list_end (list);
}
```
检查链表是否为空
`ready_list`
``` c
/* List of processes in THREAD_READY state, that is, processes
   that are ready to run but not actually running. */
static struct list ready_list;
```
就绪队列，存放所有处于ready状态的线程
`list_entry`
``` c
/* Converts pointer to list element LIST_ELEM into a pointer to
   the structure that LIST_ELEM is embedded inside.  Supply the
   name of the outer structure STRUCT and the member name MEMBER
   of the list element.  See the big comment at the top of the
   file for an example. */
   
#define list_entry(LIST_ELEM, STRUCT, MEMBER)           \
        ((STRUCT *) ((uint8_t *) &(LIST_ELEM)->next     \
                     - offsetof (STRUCT, MEMBER.next)))
```
把`struct list_elem*`还原成包含它的外层结构体指针
`list_pop_front()`
``` c
/* Removes the front element from LIST and returns it.
   Undefined behavior if LIST is empty before removal. */
struct list_elem * list_pop_front (struct list *list)
{
  struct list_elem *front = list_front (list);
  list_remove (front);
  return front;
}
```
获取head之后的第一个节点并移除，返回该节点
`list_front()`
``` c
/* Returns the front element in LIST.
   Undefined behavior if LIST is empty. */
struct list_elem * list_front (struct list *list)
{
  ASSERT (!list_empty (list));
  return list->head.next;
}
```
获取head之后的第一个节点
`list_remove()`
``` c
/* Removes ELEM from its list and returns the element that
   followed it.  Undefined behavior if ELEM is not in a list.
   A list element must be treated very carefully after removing
   it from its list.  Calling list_next() or list_prev() on ELEM
   will return the item that was previously before or after ELEM,
   but, e.g., list_prev(list_next(ELEM)) is no longer ELEM!
   The list_remove() return value provides a convenient way to
   iterate and remove elements from a list:
   for (e = list_begin (&list); e != list_end (&list); e = list_remove (e))
     {
       ...do something with e...
     }
   If you need to free() elements of the list then you need to be
   more conservative.  Here's an alternate strategy that works
   even in that case:
   while (!list_empty (&list))
     {
       struct list_elem *e = list_pop_front (&list);
       ...do something with e...
     }
*/
struct list_elem *
list_remove (struct list_elem *elem)
{
  ASSERT (is_interior (elem));
  elem->prev->next = elem->next;
  elem->next->prev = elem->prev;
  return elem->next;
}
```
将elem删除，并链接其前后节点
`is_interior()`
``` c
/* Returns true if ELEM is an interior element,
   false otherwise. */
static inline bool is_interior (struct list_elem *elem)
{
  return elem != NULL && elem->prev != NULL && elem->next != NULL;
}
```
检查elem是否为链表内部（非头非尾）节点
（3）`switch_threads()`
``` c
/* Switches from CUR, which must be the running thread, to NEXT,
   which must also be running switch_threads(), returning CUR in
   NEXT's context. */
struct thread *switch_threads (struct thread *cur, struct thread *next);
```
`switch_threads()`函数的实现是在`threads/switch.S`中的汇编语言
``` assembly
#include "threads/switch.h"

#### struct thread *switch_threads (struct thread *cur, struct thread *next);
####
#### Switches from CUR, which must be the running thread, to NEXT,
#### which must also be running switch_threads(), returning CUR in
#### NEXT's context.
####
#### This function works by assuming that the thread we're switching
#### into is also running switch_threads().  Thus, all it has to do is
#### preserve a few registers on the stack, then switch stacks and
#### restore the registers.  As part of switching stacks we record the
#### current stack pointer in CUR's thread structure.

.globl switch_threads
.func switch_threads
switch_threads:
	# Save caller's register state.
	#
	# Note that the SVR4 ABI allows us to destroy %eax, %ecx, %edx,
	# but requires us to preserve %ebx, %ebp, %esi, %edi.  See
	# [SysV-ABI-386] pages 3-11 and 3-12 for details.
	#
	# This stack frame must match the one set up by thread_create()
	# in size.
	/*保存当前线程的被调用者的保存寄存器，压入cur线程的内核栈*/
	pushl %ebx
	pushl %ebp
	pushl %esi
	pushl %edi

	# Get offsetof (struct thread, stack).
	/*读取stack的偏移量*/
.globl thread_stack_ofs
	mov thread_stack_ofs, %edx

	# Save current stack pointer to old thread's stack, if any.
	/*将当前线程的栈指针esp保存到cur的stack*/
	movl SWITCH_CUR(%esp), %eax
	movl %esp, (%eax,%edx,1)//将esp写入cur->stack

	# Restore stack pointer from new thread's stack.
	/*切换到next线程的栈*/
	movl SWITCH_NEXT(%esp), %ecx
	movl (%ecx,%edx,1), %esp//将next->stack写入esp

	# Restore caller's register state.
	/*恢复之前保存的寄存器*/
	popl %edi
	popl %esi
	popl %ebp
	popl %ebx
        ret
.endfunc

.globl switch_entry
.func switch_entry
/*首次启动某线程*/
switch_entry:
	# Discard switch_threads() arguments.
	addl $8, %esp

	# Call thread_schedule_tail(prev).
	pushl %eax
.globl thread_schedule_tail
	call thread_schedule_tail
	addl $4, %esp

	# Start thread proper.
	ret
.endfunc

```

`thread_stack_ofs`
``` c
uint32_t thread_stack_ofs = offsetof (struct thread, stack);
```
找到`struct thread`中存储线程栈指针的位置

综上，`schedule()`可切换到next并在next中继续执行，返回cur
（4）`thread_schedule_tail()`
``` c
/* Completes a thread switch by activating the new thread's page
   tables, and, if the previous thread is dying, destroying it.
   At this function's invocation, we just switched from thread
   PREV, the new thread is already running, and interrupts are
   still disabled.  This function is normally invoked by
   thread_schedule() as its final action before returning, but
   the first time a thread is scheduled it is called by
   switch_entry() (see switch.S).
   It's not safe to call printf() until the thread switch is
   complete.  In practice that means that printf()s should be
   added at the end of the function.
   After this function and its caller returns, the thread switch
   is complete. */
void thread_schedule_tail (struct thread *prev)
{
  struct thread *cur = running_thread ();//获取当前正在运行的线程
  ASSERT (intr_get_level () == INTR_OFF);//断言中断关闭
  /* Mark us as running. */
  cur->status = THREAD_RUNNING;//将当前线程状态从就绪→正在运行
  /* Start new time slice. */
  thread_ticks = 0;//重置时间片
#ifdef USERPROG
  /* Activate the new address space. */
  process_activate ();//触发新的地址空间
#endif
  /* If the thread we switched from is dying, destroy its struct
     thread.  This must happen late so that thread_exit() doesn't
     pull out the rug under itself.  (We don't free
     initial_thread because its memory was not obtained via
     palloc().) */
  if (prev != NULL && prev->status == THREAD_DYING && prev != initial_thread)//如果切换的线程即将被销毁
    {
      ASSERT (prev != cur);
      palloc_free_page (prev);//释放
    }
}
```
综上，`thread_schedule_tail()`可获取当前线程并分配恢复之前执行的状态和现场
#### ⑨第9行：调用`intr_set_level()`函数
- 如果`level==INTR_ON`→调用`intr_enable ()`将中断打开，并返回调用前的中断状态
- 如果`level==INTR_OFF`→调用`intr_enable ()`将中断关闭，并返回调用前的中断状态

综上，`thread_yield ()`可保证调度切换的原子性

综上，`timer_sleep()`可以让当前线程在达到ticks个时钟节拍前不断调用`thread_yield()`让出CPU，时间到了之后再继续运行
## 三、缺点分析
通过以上分析可以得出，`timer_sleep()`的缺点忙等待主要体现在：
``` c
  while (timer_elapsed (start) < ticks)
    thread_yield ();
```
`thread_yield ()`的原理是将当前线程移动到就绪队列并调度其他线程，但是并没有真正被阻塞，可能很快又调度回当前线程，就会不断地切换上下文，而切换上下文的花销大，导致CPU效率低下。
## 四、修改：实现阻塞
### 1.在原`thread`结构体中新增加一个`block_ticks`变量，表示剩余ticks数
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
  };
```
#### ①在创建线程函数`thread_create()`中将剩余ticks数初始化为0
``` c
/* Creates a new kernel thread named NAME with the given initial
   PRIORITY, which executes FUNCTION passing AUX as the argument,
   and adds it to the ready queue.  Returns the thread identifier
   for the new thread, or TID_ERROR if creation fails.
   If thread_start() has been called, then the new thread may be
   scheduled before thread_create() returns.  It could even exit
   before thread_create() returns.  Contrariwise, the original
   thread may run for any amount of time before the new thread is
   scheduled.  Use a semaphore or some other form of
   synchronization if you need to ensure ordering.
   The code provided sets the new thread's `priority' member to
   PRIORITY, but no actual priority scheduling is implemented.
   Priority scheduling is the goal of Problem 1-3. */
tid_t thread_create (const char *name, int priority,
               thread_func *function, void *aux)
{
  struct thread *t;
  struct kernel_thread_frame *kf;
  struct switch_entry_frame *ef;
  struct switch_threads_frame *sf;
  tid_t tid;
  ASSERT (function != NULL);
  /* Allocate thread. */
  t = palloc_get_page (PAL_ZERO);
  if (t == NULL)
    return TID_ERROR;
  /* Initialize thread. */
  init_thread (t, name, priority);
  tid = t->tid = allocate_tid ();
  /* Stack frame for kernel_thread(). */
  kf = alloc_frame (t, sizeof *kf);
  kf->eip = NULL;
  kf->function = function;
  kf->aux = aux;
  /* Stack frame for switch_entry(). */
  ef = alloc_frame (t, sizeof *ef);
  ef->eip = (void (*) (void)) kernel_thread;
  /* Stack frame for switch_threads(). */
  sf = alloc_frame (t, sizeof *sf);
  sf->eip = switch_entry;
  sf->ebp = 0;
  /* Add to run queue. */
  thread_unblock (t);
  t->block_ticks=0;
  return tid;
}
```
### 2.在`thread.c`中新增一个函数`blocked_thread_check()`
``` c
void blocked_thread_check(struct thread*t,void *aux UNUSED)
{
  if(t->status==THREAD_BLOCKED&&t->block_ticks>0)//如果线程被阻塞
  {
    t->block_ticks--;//剩余时间-1
    if(t->block_ticks==0)//如果达到目标时间
    {
      thread_unblock(t);//将线程从阻塞→就绪状态，移动到ready队列
    }
  }
}
```

调用`thread_unblock()`：按优先级插入队列
``` c
/* Transitions a blocked thread T to the ready-to-run state.
   This is an error if T is not blocked.  (Use thread_yield() to
   make the running thread ready.)
   This function does not preempt the running thread.  This can
   be important: if the caller had disabled interrupts itself,
   it may expect that it can atomically unblock a thread and
   update other data. */
void thread_unblock (struct thread *t)
{
  enum intr_level old_level;
  ASSERT (is_thread (t));
  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  // list_push_back (&ready_list, &t->elem);
  list_insert_ordered(&ready_list,&t->elem,compareByPriority,NULL);
  t->status = THREAD_READY;
  intr_set_level (old_level);
}
```
### 3.在时钟中断处理函数`timer_interrupt()`中加入对线程要sleep的剩余ticks的检测
``` c
/* Timer interrupt handler. */
static void timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_tick ();
  thread_foreach(blocked_thread_check,NULL);//每次中断都对所有线程调用blocked_thread_check()，更新睡眠倒计时
}
```

调用`thread_foreach()`
``` c
/* Invoke function 'func' on all threads, passing along 'aux'.
   This function must be called with interrupts off. */
void thread_foreach (thread_action_func *func, void *aux)
{
  struct list_elem *e;
  ASSERT (intr_get_level () == INTR_OFF);//断言中断关闭，防止死锁
  for (e = list_begin (&all_list); e != list_end (&all_list);
       e = list_next (e))//遍历
    {
      struct thread *t = list_entry (e, struct thread, allelem);
      func (t, aux);
    }
}
```
### 4.修改`timer_sleep()`
``` c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void timer_sleep (int64_t ticks)
{
  // int64_t start = timer_ticks ();
  // ASSERT (intr_get_level () == INTR_ON);
  // while (timer_elapsed (start) < ticks)
  //   thread_yield ();
  if(ticks>0)//如果还在被阻塞
  {
    ASSERT(intr_get_level()==INTR_ON);//断言中断开启，防止死锁
    enum intr_level old_level=intr_disable();//将中断关闭，返回调用前的中断状态
    struct thread *cur_thread=thread_current();//获取当前线程
    cur_thread->block_ticks=ticks;//设置睡眠时间
    thread_block();//阻塞该线程
    intr_set_level(old_level);//恢复调用前的中断状态
  }
}
```

调用`thread_block()`：阻塞线程
``` c
/* Puts the current thread to sleep.  It will not be scheduled
   again until awoken by thread_unblock().
   This function must be called with interrupts turned off.  It
   is usually a better idea to use one of the synchronization
   primitives in synch.h. */
void thread_block (void)
{
  ASSERT (!intr_context ());//断言不处于中断上下文中
  ASSERT (intr_get_level () == INTR_OFF);//断言中断关闭
  thread_current ()->status = THREAD_BLOCKED;//修改线程状态为阻塞
  schedule ();//调度
}
```
# （三）`priority-*`
## 一、查看文档
通过查看pintos.pdf可知，在这部分需要完成以下工作：
1.实现优先级调度，包括：使ready队列按照优先级调度而非FIFO；若新添加到ready队列的线程的优先级高于当前正在运行的线程，当前线程应立即让出CPU；所有阻塞队列都要按照优先级唤醒
2.实现优先级捐赠，包括：当高优先级的线程等待锁时，应将其优先级捐赠给持有锁的那个低优先级线程；实现嵌套捐赠；实现多个捐赠；当释放锁时，该线程恢复到捐赠前的优先级
3.实现`threads/thread.c`中的`thread_set_priority(int new_priority)`函数和`thread_get_priority(void)`函数。其中，`thread_set_priority(int new_priority)`函数的功能是将当前线程的优先级设置为new_priority，如果当前线程不再具有最高优先级，则yield；`thread_get_priority(void)`函数的功能是返回当前线程的优先级，在存在优先捐赠的情况下，返回被捐赠的优先级。
## 二、源代码分析&修改
通过阅读pdf，可以发现需要确保任何时刻运行的线程都是ready队列中优先级最大的线程
### 1.修改ready队列调度算法为优先级调度
由于需要确保任何时刻运行的线程都是ready队列中优先级最大的线程，所以首先需要确保ready队列本身是一个按优先级排序的队列
#### ①分析源代码
测试用例`alarm-priority`的测试内容是检查当多个线程在同一时刻被alarm唤醒时，是否按照优先级高低的顺序来运行。逻辑是：将主线程的优先级跳到最低，然后循环创建10个有不同优先级的线程，接着对这10个线程调用timer.c中的timer_sleep函数，使这10个进程同时在5秒后被唤醒，并按顺序输出Thread priority 30 woke up至Thread priority 21 woke up。而在执行`pintos --qemu -- -q run alarm-priority`后可以发现高优先级的线程并没有优先被唤醒。

通过分析源代码可知，当多个线程被`timer_sleep`同时唤醒时，线程会通过`thread_unblock`函数重新进入`ready_list`队列，而`ready_list`是FIFO队列，也就是会将线程直接插到队尾，而不按照priority顺序，导致输出是不按照30-21这个顺序来输出的，所以需要调整ready_list队列从FIFO为按照优先级priority从高到低的有序链表，从而将输出调整为30-21倒序而不是杂乱顺序。

在原先的`thread_unblock()`中，函数通过`list_push_back (&ready_list, &t->elem);`这一行来将线程直接插入队尾，并没有按照优先级插入，所以需要增加一个函数来比较两个线程的优先级，选择较高的那个插入到前面去。

在之前分析`timer_sleep()`时，发现`timer_sleep()`调用的`thread_yield()`调用的`list_push_back()`的同文件中有一个`list_insert_ordered()`函数，正好符合优先级调度所需要的按顺序插入
``` c
/* Inserts ELEM in the proper position in LIST, which must be
   sorted according to LESS given auxiliary data AUX.
   Runs in O(n) average case in the number of elements in LIST. */
void list_insert_ordered (struct list *list, struct list_elem *elem,list_less_func *less, void *aux)
{
  struct list_elem *e;
  ASSERT (list != NULL);
  ASSERT (elem != NULL);
  ASSERT (less != NULL);
  for (e = list_begin (list); e != list_end (list); e = list_next (e))
    if (less (elem, e, aux))
      break;
  return list_insert (e, elem);
}
```
- `list_elem`
``` c
/* List element. */
struct list_elem
  {
    struct list_elem *prev;     /* Previous list element. */
    struct list_elem *next;     /* Next list element. */
  };
```
双向链表节点，由前驱指针+后继指针组成
- `list`
``` c
/* List. */
struct list
  {
    struct list_elem head;      /* List head. */
    struct list_elem tail;      /* List tail. */
  };
```
伪头节点+伪尾节点
- `list_less_func`
``` c
/* Compares the value of two list elements A and B, given
   auxiliary data AUX.  Returns true if A is less than B, or
   false if A is greater than or equal to B. */
typedef bool list_less_func (const struct list_elem *a,
                             const struct list_elem *b,
                             void *aux);
```
如果a＜b→return true
- `list_begin()`
``` c
/* Returns the beginning of LIST.  */
struct list_elem * list_begin (struct list *list)
{
  ASSERT (list != NULL);
  return list->head.next;
}
```
返回head之后的第一个节点
- `list_end()`
``` c
/* Returns LIST's tail.
   list_end() is often used in iterating through a list from
   front to back.  See the big comment at the top of list.h for
   an example. */
struct list_elem * list_end (struct list *list)
{
  ASSERT (list != NULL);
  return &list->tail;
}
```
返回尾节点
- `list_next()`
``` c
/* Returns the element after ELEM in its list.  If ELEM is the
   last element in its list, returns the list tail.  Results are
   undefined if ELEM is itself a list tail. */
struct list_elem * list_next (struct list_elem *elem)
{
  ASSERT (is_head (elem) || is_interior (elem));
  return elem->next;
}
```
返回下一个节点
- `list_insert()`
``` c
/* Inserts ELEM just before BEFORE, which may be either an
   interior element or a tail.  The latter case is equivalent to
   list_push_back(). */
void list_insert (struct list_elem *before, struct list_elem *elem)
{
  ASSERT (is_interior (before) || is_tail (before));
  ASSERT (elem != NULL);
  elem->prev = before->prev;
  elem->next = before;
  before->prev->next = elem;
  before->prev = elem;
}
```
将elem插入before前

因此，可以利用`list_insert_ordered()`来替换`thread_unblock()`中的直接插入，但在此之前还需要设计一个函数来比较优先级：
``` c
bool compareByPriority(const struct list_elem *a,const struct list_elem *b,void *aux UNUSED)
{
  int pa=list_entry(a,struct thread,elem)->priority;
  int pb=list_entry(b,struct thread,elem)->priority;
  return pa>pb;
}
```
#### ②修改`thread_unblock()`函数
使用`list_insert_ordered()`来替换`thread_unblock()`中的直接插入，并调用`compareByPriority`函数来比较优先级
``` c
/* Transitions a blocked thread T to the ready-to-run state.
   This is an error if T is not blocked.  (Use thread_yield() to
   make the running thread ready.)
   This function does not preempt the running thread.  This can
   be important: if the caller had disabled interrupts itself,
   it may expect that it can atomically unblock a thread and
   update other data. */
void thread_unblock (struct thread *t)
{
  enum intr_level old_level;
  ASSERT (is_thread (t));
  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  // list_push_back (&ready_list, &t->elem);
  list_insert_ordered(&ready_list,&t->elem,compareByPriority,NULL);
  t->status = THREAD_READY;
  intr_set_level (old_level);
}
```
### 2.当线程调用`thread_yield()`时需要按照优先级重新进入ready队列
线程调用`thread_yield()`时会主动让出CPU，这样也会改变ready队列，但是仍需要重新按照优先级进入队列，避免破坏调度顺序。因此按照与`thread_unblock()`同样的逻辑进行修改。
``` c
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void thread_yield (void)
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;
  ASSERT (!intr_context ());
  old_level = intr_disable ();
  if (cur != idle_thread)
    //list_push_back (&ready_list, &cur->elem);
    list_insert_ordered(&ready_list,&cur->elem,compareByPriority,NULL);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
```
### 3.若新添加到ready队列的线程的优先级高于当前正在运行的线程，当前线程应立即让出CPU
#### ①分析源代码
添加线程也会修改ready队列，也需要保证现在正在运行的队列是优先级最大的队列。通过阅读源代码发现，`thread_create()`是用于实现这个功能的函数。
``` c
/* Creates a new kernel thread named NAME with the given initial
   PRIORITY, which executes FUNCTION passing AUX as the argument,
   and adds it to the ready queue.  Returns the thread identifier
   for the new thread, or TID_ERROR if creation fails.
   If thread_start() has been called, then the new thread may be
   scheduled before thread_create() returns.  It could even exit
   before thread_create() returns.  Contrariwise, the original
   thread may run for any amount of time before the new thread is
   scheduled.  Use a semaphore or some other form of
   synchronization if you need to ensure ordering.
   The code provided sets the new thread's `priority' member to
   PRIORITY, but no actual priority scheduling is implemented.
   Priority scheduling is the goal of Problem 1-3. */
tid_t thread_create (const char *name, int priority,
               thread_func *function, void *aux)
{
  struct thread *t;
  struct kernel_thread_frame *kf;
  struct switch_entry_frame *ef;
  struct switch_threads_frame *sf;
  tid_t tid;
  ASSERT (function != NULL);//断言新线程有一个可执行函数
  /* Allocate thread. */
  t = palloc_get_page (PAL_ZERO);//分配线程所占的一整页内存
  if (t == NULL)
    return TID_ERROR;
  /* Initialize thread. */
  init_thread (t, name, priority);//初始化线程
  tid = t->tid = allocate_tid ();//分配线程号
  /* Stack frame for kernel_thread(). *///构造内核页帧
  kf = alloc_frame (t, sizeof *kf);
  kf->eip = NULL;
  kf->function = function;
  kf->aux = aux;
  /* Stack frame for switch_entry(). */
  ef = alloc_frame (t, sizeof *ef);
  ef->eip = (void (*) (void)) kernel_thread;
  /* Stack frame for switch_threads(). */
  sf = alloc_frame (t, sizeof *sf);
  sf->eip = switch_entry;
  sf->ebp = 0;
  /* Add to run queue. */
  thread_unblock (t);//将新线程添加到ready队列
  t->block_ticks=0;
  return tid;
}
```
该函数最后将新线程添加到ready队列，而上一步已经将`thread_unblock()`函数修改为按优先级插入队列。但是pdf要求：若新添加到ready队列的线程的优先级高于当前正在运行的线程，当前线程应立即让出CPU。现在的`thread_create()`函数并不能够实现这个功能，所以需要增加一段代码来比较新线程的优先级和当前正在运行的线程的优先级，如果新线程的优先级更高，就将CPU分配给它。
#### ②修改`thread_create()`函数
``` c
/* Creates a new kernel thread named NAME with the given initial
   PRIORITY, which executes FUNCTION passing AUX as the argument,
   and adds it to the ready queue.  Returns the thread identifier
   for the new thread, or TID_ERROR if creation fails.
   If thread_start() has been called, then the new thread may be
   scheduled before thread_create() returns.  It could even exit
   before thread_create() returns.  Contrariwise, the original
   thread may run for any amount of time before the new thread is
   scheduled.  Use a semaphore or some other form of
   synchronization if you need to ensure ordering.
   The code provided sets the new thread's `priority' member to
   PRIORITY, but no actual priority scheduling is implemented.
   Priority scheduling is the goal of Problem 1-3. */
tid_t thread_create (const char *name, int priority,
               thread_func *function, void *aux)
{
  struct thread *t;
  struct kernel_thread_frame *kf;
  struct switch_entry_frame *ef;
  struct switch_threads_frame *sf;
  tid_t tid;
  ASSERT (function != NULL);//断言新线程有一个可执行函数
  /* Allocate thread. */
  t = palloc_get_page (PAL_ZERO);//分配线程所占的一整页内存
  if (t == NULL)
    return TID_ERROR;
  /* Initialize thread. */
  init_thread (t, name, priority);//初始化线程
  tid = t->tid = allocate_tid ();//分配线程号
  /* Stack frame for kernel_thread(). *///构造内核页帧
  kf = alloc_frame (t, sizeof *kf);
  kf->eip = NULL;
  kf->function = function;
  kf->aux = aux;
  /* Stack frame for switch_entry(). */
  ef = alloc_frame (t, sizeof *ef);
  ef->eip = (void (*) (void)) kernel_thread;
  /* Stack frame for switch_threads(). */
  sf = alloc_frame (t, sizeof *sf);
  sf->eip = switch_entry;
  sf->ebp = 0;
  /* Add to run queue. */
  thread_unblock (t);//将新线程添加到ready队列
  t->block_ticks=0;
  return tid;
}
```
### 4.设置优先级
pdf要求实现`thread_set_priority(int new_priority)`函数，该函数的功能是将当前线程的优先级设置为new_priority，如果当前线程不再具有最高优先级，则yield。而源代码只实现了第一个功能，没有实现第二个功能，所以需要添加一行`thread_yield()`的调用。
``` c
/* Sets the current thread's priority to NEW_PRIORITY. */
void thread_set_priority (int new_priority)
{
  thread_current ()->priority = new_priority;
  thread_yield();//新增
}
```
### 5.优先级捐赠
在上述工作结束后，优先级剩余的9个测试用例还没有通过。由于pdf对于这一部分的要求较多，为了能够更加全面地分析需要修改和增加的功能，这里通过分析priority-condvar、priority-donate-chain、priority-donate-lower、priority-donate-multiple、priority-donate-multiple2、priority-donate-nest、priority-donate-one、priority-donate-sema、priority-sema这9个测试用例的.c测试文件和.ck期望输出。
#### ①priority-donate-one
首先先从最基础的单层无嵌套开始
（1）.c测试文件
``` c
/* The main thread acquires a lock.  Then it creates two
   higher-priority threads that block acquiring the lock, causing
   them to donate their priorities to the main thread.  When the
   main thread releases the lock, the other threads should
   acquire it in priority order.
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */
#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"
static thread_func acquire1_thread_func;
static thread_func acquire2_thread_func;
void
test_priority_donate_one (void)
{
  struct lock lock;
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言当前不是多级反馈队列调度模式
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&lock);//初始化锁
  lock_acquire (&lock);//当前线程获取锁
  thread_create ("acquire1", PRI_DEFAULT + 1, acquire1_thread_func, &lock);//创建线程acquire1，优先级比当前线程大1
  msg ("This thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 1, thread_get_priority ());//当前线程应该有PRI_DEFAULT + 1的优先级，因为当前线程拥有锁，所以acquire1应该将其优先级捐赠给当前线程
  thread_create ("acquire2", PRI_DEFAULT + 2, acquire2_thread_func, &lock);//创建线程acquire2，优先级比当前线程大2
  msg ("This thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 2, thread_get_priority ());//当前线程应该有PRI_DEFAULT + 2的优先级，因为当前线程拥有锁，所以acquire2应该将其优先级捐赠给当前线程
  lock_release (&lock);//当前线程释放锁，此时应先acquire2获取锁，再acquire1获取锁
  msg ("acquire2, acquire1 must already have finished, in that order.");//acquire1和acquire2已经执行结束，且因为acquire2的优先级大于acquire1，所以先执行acquire2，再执行acquire1
  msg ("This should be the last line before finishing this test.");
}

static void
acquire1_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);//acquire1试图获取锁，会将其优先级捐赠给当前线程
  msg ("acquire1: got the lock");//acquire1成功获取锁
  lock_release (lock);//acquire1释放锁
  msg ("acquire1: done");
}

static void
acquire2_thread_func (void *lock_)//与上一个函数同理，但由于acquire2的优先级更高，应该先于acquire1获取锁
{
  struct lock *lock = lock_;
  lock_acquire (lock);
  msg ("acquire2: got the lock");
  lock_release (lock);
  msg ("acquire2: done");
}
```
获取锁需要调用`lock_acquire()`函数
``` c
/* Acquires LOCK, sleeping until it becomes available if
   necessary.  The lock must not already be held by the current
   thread.
   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but interrupts will be turned back on if
   we need to sleep. */

void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);//断言锁非空
  ASSERT (!intr_context ());//断言不在中断上下文中
  ASSERT (!lock_held_by_current_thread (lock));//断言当前线程不持有该锁
  sema_down (&lock->semaphore);//相当于wait(mutex)
  lock->holder = thread_current ();//当前线程成功获取锁后，将当前线程命为锁的持有者
}
```
该函数又调用了`sema_down()`函数来获取锁
``` c
/* Down or "P" operation on a semaphore.  Waits for SEMA's value
   to become positive and then atomically decrements it.
   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but if it sleeps then the next scheduled
   thread will probably turn interrupts back on. */
void
sema_down (struct semaphore *sema)
{
  enum intr_level old_level;
  ASSERT (sema != NULL);//断言信号量非空
  ASSERT (!intr_context ());//断言不在中断上下文中
  old_level = intr_disable ();//将中断状态关闭，返回调用前的中断状态
  while (sema->value == 0)//如果信号量值为0，则阻塞
    {
      list_push_back (&sema->waiters, &thread_current ()->elem);//进入FIFO队列
      thread_block ();//阻塞
    }
  sema->value--;//当信号量非负，则申请一个资源，信号量减1
  intr_set_level (old_level);//恢复调用前的中断状态
}
```
（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-one) begin
(priority-donate-one) This thread should have priority 32.  Actual priority: 32.//第1次捐赠
(priority-donate-one) This thread should have priority 33.  Actual priority: 33.//第2次捐赠
(priority-donate-one) acquire2: got the lock//acquire2先获取锁
(priority-donate-one) acquire2: done
(priority-donate-one) acquire1: got the lock
(priority-donate-one) acquire1: done
(priority-donate-one) acquire2, acquire1 must already have finished, in that order.//acquire2先于acquire1执行结束
(priority-donate-one) This should be the last line before finishing this test.
(priority-donate-one) end
EOF
pass;
```
要求是要实现2次捐赠，且acquire2先于acquire1获取锁、执行结束，因此需要将FIFO队列修改为优先级队列
#### ②priority-donate-multiple
接下来分析多个锁的情况
（1）.c测试文件
``` c
/* The main thread acquires locks A and B, then it creates two
   higher-priority threads.  Each of these threads blocks
   acquiring one of the locks and thus donate their priority to
   the main thread.  The main thread releases the locks in turn
   and relinquishes its donated priorities.
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */

#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"
static thread_func a_thread_func;
static thread_func b_thread_func;

void
test_priority_donate_multiple (void)
{
  struct lock a, b;//两把锁
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&a);//初始化锁
  lock_init (&b);
  lock_acquire (&a);//当前线程获取两把锁
  lock_acquire (&b);
  thread_create ("a", PRI_DEFAULT + 1, a_thread_func, &a);//创建a线程，优先级比当前线程大1
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 1, thread_get_priority ());//a应该将其优先级捐赠给当前线程
  thread_create ("b", PRI_DEFAULT + 2, b_thread_func, &b);//创建a线程，优先级比当前线程大2
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 2, thread_get_priority ());//b应该将其优先级捐赠给当前线程
  lock_release (&b);//当前线程释放锁b
  msg ("Thread b should have just finished.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 1, thread_get_priority ());//当前线程恢复到第二次捐赠前的优先级
  lock_release (&a);//当前线程释放锁a
  msg ("Thread a should have just finished.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT, thread_get_priority ());//当前线程恢复到第一次捐赠前的优先级
}

static void
a_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);//a线程试图获取锁，应将其优先级捐赠给当前线程
  msg ("Thread a acquired lock a.");
  lock_release (lock);//a线程释放锁
  msg ("Thread a finished.");
}

static void
b_thread_func (void *lock_)//和上一个函数同理，由于b的优先级高于a，因此b应先于a获取锁并先于a执行
{
  struct lock *lock = lock_;
  lock_acquire (lock);
  msg ("Thread b acquired lock b.");
  lock_release (lock);
  msg ("Thread b finished.");
}
```
（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-multiple) begin
(priority-donate-multiple) Main thread should have priority 32.  Actual priority: 32.//第一次捐赠
(priority-donate-multiple) Main thread should have priority 33.  Actual priority: 33.//第二次捐赠
(priority-donate-multiple) Thread b acquired lock b.//b应先于a获取锁并先于a执行
(priority-donate-multiple) Thread b finished.
(priority-donate-multiple) Thread b should have just finished.
(priority-donate-multiple) Main thread should have priority 32.  Actual priority: 32.//当前线程恢复到第二次捐赠前的优先级
(priority-donate-multiple) Thread a acquired lock a.
(priority-donate-multiple) Thread a finished.
(priority-donate-multiple) Thread a should have just finished.
(priority-donate-multiple) Main thread should have priority 31.  Actual priority: 31.//当前线程恢复到第一次捐赠前的优先级
(priority-donate-multiple) end
EOF
pass;
```
要求是在one的基础上实现释放锁后的优先级恢复，因此需要记录两次优先级，即记录对当前线程捐赠过优先级的线程
#### ③priority-donate-multiple2
接下来分析多锁&&错位释放锁的情况
（1）.c测试文件
``` c
/* The main thread acquires locks A and B, then it creates three
   higher-priority threads.  The first two of these threads block
   acquiring one of the locks and thus donate their priority to
   the main thread.  The main thread releases the locks in turn
   and relinquishes its donated priorities, allowing the third thread
   to run.
   In this test, the main thread releases the locks in a different
   order compared to priority-donate-multiple.c.
   Written by Godmar Back <gback@cs.vt.edu>.
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */

#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"

static thread_func a_thread_func;
static thread_func b_thread_func;
static thread_func c_thread_func;

void
test_priority_donate_multiple2 (void)
{
  struct lock a, b;//两把锁
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&a);//初始化锁
  lock_init (&b);
  lock_acquire (&a);//当前线程获取两把锁
  lock_acquire (&b);
  thread_create ("a", PRI_DEFAULT + 3, a_thread_func, &a);//创建a线程，优先级比当前线程大3
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 3, thread_get_priority ());//a将其优先级捐赠给当前线程
  thread_create ("c", PRI_DEFAULT + 1, c_thread_func, NULL);//创建c线程，但由于其优先级不够高，所以不捐赠
  thread_create ("b", PRI_DEFAULT + 5, b_thread_func, &b);//创建b线程，优先级比当前线程大5
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 5, thread_get_priority ());//b线程优先级最高，将其优先级捐赠给当前线程
  lock_release (&a);//当前线程释放锁a，但由于b线程优先级最高，所以b获取锁
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 5, thread_get_priority ());//不撤回b的捐赠
  lock_release (&b);////当前线程释放锁b，但由于a线程优先级最高，所以a获取锁，最后c获取锁
  msg ("Threads b, a, c should have just finished, in that order.");
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT, thread_get_priority ());//当前线程恢复到最初的优先级
}

//与之前的测试用例的代码逻辑相同，按照b→a→c的顺序获取锁
static void
a_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);
  msg ("Thread a acquired lock a.");
  lock_release (lock);
  msg ("Thread a finished.");
}
static void
b_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);
  msg ("Thread b acquired lock b.");
  lock_release (lock);
  msg ("Thread b finished.");
}

static void
c_thread_func (void *a_ UNUSED)
{
  msg ("Thread c finished.");
}
```
（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-multiple2) begin
(priority-donate-multiple2) Main thread should have priority 34.  Actual priority: 34.//第一次捐赠
(priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 36.//第二次捐赠
(priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 36.

//按优先级获取锁
(priority-donate-multiple2) Thread b acquired lock b.
(priority-donate-multiple2) Thread b finished.
(priority-donate-multiple2) Thread a acquired lock a.
(priority-donate-multiple2) Thread a finished.
(priority-donate-multiple2) Thread c finished.
(priority-donate-multiple2) Threads b, a, c should have just finished, in that order.
(priority-donate-multiple2) Main thread should have priority 31.  Actual priority: 31.//当前线程恢复到最初的优先级
(priority-donate-multiple2) end
EOF
pass;
```
multiple2要求在multiple的基础上实现：当锁的释放顺序和优先级捐赠来源不一致时，恢复优先级时需要在初始优先级和剩余捐献列表中的优先值取最大值进行更新。
#### ④priority-donate-nest
接下来来看嵌套捐赠
（1）.ck测试文件
``` c
/* Low-priority main thread L acquires lock A.  Medium-priority
   thread M then acquires lock B then blocks on acquiring lock A.
   High-priority thread H then blocks on acquiring lock B.  Thus,
   thread H donates its priority to M, which in turn donates it
   to thread L.
   Based on a test originally submitted for Stanford's CS 140 in
   winter 1999 by Matt Franklin <startled@leland.stanford.edu>,
   Greg Hutchins <gmh@leland.stanford.edu>, Yu Ping Hu
   <yph@cs.stanford.edu>.  Modified by arens. */

#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"

struct locks
  {
    struct lock *a;
    struct lock *b;
  };
static thread_func medium_thread_func;
static thread_func high_thread_func;

void
test_priority_donate_nest (void)
{
  struct lock a, b;//两把锁
  struct locks locks;//指向两把锁的指针
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&a);//初始化锁
  lock_init (&b);
  lock_acquire (&a);//当前线程获取锁a 
  locks.a = &a;//锁指针指向对应的锁
  locks.b = &b;
  thread_create ("medium", PRI_DEFAULT + 1, medium_thread_func, &locks);//创建线程M，优先级比当前线程大1，并调用medium_thread_func抢占锁b，再抢占锁a，但由于锁a被当前线程占有，所以线程M被阻塞，等待当前线程释放锁a
  thread_yield ();//当前线程将CPU让给M，直到M被阻塞
  msg ("Low thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 1, thread_get_priority ());//M将优先级捐赠给当前线程
  thread_create ("high", PRI_DEFAULT + 2, high_thread_func, &b);//创建H线程，优先级比当前线程大2，试图获取锁b，被阻塞
  thread_yield ();//主线程将CPU让给H，直到H被阻塞
  msg ("Low thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 2, thread_get_priority ());//H将优先级捐赠给M，而由于M也被阻塞，所以M再将更新后的优先级捐赠给当前线程
  //此时H等待M释放锁b，M等待当前线程释放锁a
  lock_release (&a);//当前线程释放锁a
  thread_yield ();
  //M释放锁b后，H获得锁b，并先于M执行
  msg ("Medium thread should just have finished.");
  msg ("Low thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT, thread_get_priority ());//当前线程恢复到最初优先级
}

static void
medium_thread_func (void *locks_)
{
  struct locks *locks = locks_;
  lock_acquire (locks->b);//M获取锁b
  lock_acquire (locks->a);//M再获取锁a
  msg ("Medium thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 2, thread_get_priority ());//H将优先级捐赠给M
  msg ("Medium thread got the lock.");
  lock_release (locks->a);//M释放锁a
  thread_yield ();
  lock_release (locks->b);//M释放锁b
  thread_yield ();
  msg ("High thread should have just finished.");//H先于M执行
  msg ("Middle thread finished.");
}

static void
high_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);
  msg ("High thread got the lock.");
  lock_release (lock);
  msg ("High thread finished.");
}
```
（2）.ck期望输出
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-nest) begin
(priority-donate-nest) Low thread should have priority 32.  Actual priority: 32.//当前线程被M捐赠优先级
(priority-donate-nest) Low thread should have priority 33.  Actual priority: 33.//M被H捐赠优先级后，将更新的优先级再次捐赠给当前线程
(priority-donate-nest) Medium thread should have priority 33.  Actual priority: 33.//H将优先级捐赠给M
(priority-donate-nest) Medium thread got the lock.//M获得锁a
(priority-donate-nest) High thread got the lock.//H获得锁b
(priority-donate-nest) High thread finished.//H先于M执行
(priority-donate-nest) High thread should have just finished.
(priority-donate-nest) Middle thread finished.//M先于当前线程执行
(priority-donate-nest) Medium thread should just have finished.
(priority-donate-nest) Low thread should have priority 31.  Actual priority: 31.//当前线程恢复到最初的优先级
(priority-donate-nest) end
EOF
pass;
```
这个用例要求更加复杂，需要实现优先级沿着嵌套捐赠的捐赠链传递更新，并要求让每次捐赠后拥有最高优先级的线程先运行，释放锁后不再拥有锁的线程恢复最初优先级，但是剩余线程需要在初优先级和捐赠列表中的优先级取最大值进行更新，因此需要设计一个数据结构记录捐赠线程以及正在等待哪个锁。
#### ⑤priority-donate-sema
接下来分析加上信号量的情况
（1）.c测试文件
``` c
/* Low priority thread L acquires a lock, then blocks downing a
   semaphore.  Medium priority thread M then blocks waiting on
   the same semaphore.  Next, high priority thread H attempts to
   acquire the lock, donating its priority to L.
   Next, the main thread ups the semaphore, waking up L.  L
   releases the lock, which wakes up H.  H "up"s the semaphore,
   waking up M.  H terminates, then M, then L, and finally the
   main thread.
   Written by Godmar Back <gback@cs.vt.edu>. */

#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"

struct lock_and_sema
  {
    struct lock lock;
    struct semaphore sema;
  };
static thread_func l_thread_func;
static thread_func m_thread_func;
static thread_func h_thread_func;

void
test_priority_donate_sema (void)
{
  struct lock_and_sema ls;//包含锁+信号量的结构体
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&ls.lock);//初始化锁
  sema_init (&ls.sema, 0);//初始化信号量为0
  thread_create ("low", PRI_DEFAULT + 1, l_thread_func, &ls);//创建L线程，获得锁和信号量，但由于信号量=0而被阻塞
  thread_create ("med", PRI_DEFAULT + 3, m_thread_func, &ls);//创建M线程，优先级比L大2，试图获取信号量，但由于信号量=0而被阻塞
  thread_create ("high", PRI_DEFAULT + 5, h_thread_func, &ls);//创建H线程，优先级比L大4，试图获取锁，但因锁被L持有而被阻塞，因此将优先级捐赠给L
  sema_up (&ls.sema);//由于此时L优先级最高，先唤醒L，接着L释放锁而恢复优先级，然后唤醒H，H获取锁，释放锁后结束运行，然后唤醒M，M结束运行，最后L结束运行
  msg ("Main thread finished.");
}

static void
l_thread_func (void *ls_)
{
  struct lock_and_sema *ls = ls_;
  lock_acquire (&ls->lock);//L获取锁
  msg ("Thread L acquired lock.");
  sema_down (&ls->sema);//L获取信号量
  msg ("Thread L downed semaphore.");
  lock_release (&ls->lock);//L释放锁
  msg ("Thread L finished.");//L结束运行
}

static void
m_thread_func (void *ls_)
{
  struct lock_and_sema *ls = ls_;
  sema_down (&ls->sema);//M试图获取信号量，被阻塞
  msg ("Thread M finished.");//获取后结束运行
}

static void
h_thread_func (void *ls_)
{
  struct lock_and_sema *ls = ls_;
  lock_acquire (&ls->lock);//H试图获取锁，被阻塞
  msg ("Thread H acquired lock.");
  sema_up (&ls->sema);//H试图获取信号量
  lock_release (&ls->lock);//H释放锁
  msg ("Thread H finished.");//H结束运行
}
```
（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-sema) begin
(priority-donate-sema) Thread L acquired lock.
(priority-donate-sema) Thread L downed semaphore.
(priority-donate-sema) Thread H acquired lock.
(priority-donate-sema) Thread H finished.
(priority-donate-sema) Thread M finished.
(priority-donate-sema) Thread L finished.
(priority-donate-sema) Main thread finished.
(priority-donate-sema) end
EOF
pass;

```
在这个测试用例中，L虽然是阻塞的，但是仍被H捐赠优先级，因此需要实现对被阻塞的进程捐赠优先级；信号量等待队列同样也是需要从FIFO改为按优先级管理；释放锁时按与之前分析得出的更新逻辑相同。
#### ⑥priority-donate-lower
接下来分析测试更新优先级的测试用例
（1）.c测试文件
``` c
/* The main thread acquires a lock.  Then it creates a
   higher-priority thread that blocks acquiring the lock, causing
   it to donate their priorities to the main thread.  The main
   thread attempts to lower its priority, which should not take
   effect until the donation is released. */
   
#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"

static thread_func acquire_thread_func;

void
test_priority_donate_lower (void)
{
  struct lock lock;
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);//断言当前线程的优先级是默认值
  lock_init (&lock);//初始化锁
  lock_acquire (&lock);//当前线程获取锁
  thread_create ("acquire", PRI_DEFAULT + 10, acquire_thread_func, &lock);//创建acquire线程，试图获取锁，但因锁被当前线程持有而被阻塞
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 10, thread_get_priority ());//acquire线程将其优先级捐赠给当前线程
  msg ("Lowering base priority...");
  thread_set_priority (PRI_DEFAULT - 10);//当前线程被捐赠优先级后，调低自身优先值
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT + 10, thread_get_priority ());//但是在捐赠结束前，不应该立刻更新优先值，仍应取被捐赠的优先值和下调后的优先值的最大值
  lock_release (&lock);//当前线程释放锁
  msg ("acquire must already have finished.");//acquire获取锁，并先于当前线程执行结束
  msg ("Main thread should have priority %d.  Actual priority: %d.",
       PRI_DEFAULT - 10, thread_get_priority ());//当前线程更新为下调后的优先值
}

static void
acquire_thread_func (void *lock_)
{
  struct lock *lock = lock_;
  lock_acquire (lock);//acquire试图获取锁
  msg ("acquire: got the lock");
  lock_release (lock);//acquire释放锁
  msg ("acquire: done");
}
```
（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-lower) begin
(priority-donate-lower) Main thread should have priority 41.  Actual priority: 41.
(priority-donate-lower) Lowering base priority...
(priority-donate-lower) Main thread should have priority 41.  Actual priority: 41.
(priority-donate-lower) acquire: got the lock
(priority-donate-lower) acquire: done
(priority-donate-lower) acquire must already have finished.
(priority-donate-lower) Main thread should have priority 21.  Actual priority: 21.
(priority-donate-lower) end
EOF
pass;

```
要求是当捐赠链未结束时，如果当前线程优先级下调，应该在下调后的优先级和捐赠列表中的优先值中取最大值，而不应该立即下调。
#### ⑦priority-sema
接下来分析信号量等待队列
（1）.c测试文件
``` c
/* Tests that the highest-priority thread waiting on a semaphore
   is the first to wake up. */
#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/malloc.h"
#include "threads/synch.h"
#include "threads/thread.h"
#include "devices/timer.h"
static thread_func priority_sema_thread;
static struct semaphore sema;
void
test_priority_sema (void)
{
  int i;
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  sema_init (&sema, 0);//将信号量初始化为0
  thread_set_priority (PRI_MIN);//将当前线程的优先级设为最低值
  for (i = 0; i < 10; i++)
    {
      int priority = PRI_DEFAULT - (i + 3) % 10 - 1;//打乱优先级顺序
      char name[16];//线程名
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, priority_sema_thread, NULL);//创建线程并wait，但由于信号量为0而被阻塞
    }
  for (i = 0; i < 10; i++)
    {
      sema_up (&sema);//相当于signal，释放一个资源、信号量+1，唤醒被阻塞的线程
      msg ("Back in main thread.");//CPU归还给当前线程
    }
}

static void
priority_sema_thread (void *aux UNUSED)
{
  sema_down (&sema);//wait
  msg ("Thread %s woke up.", thread_name ());
}
```

（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-sema) begin
(priority-sema) Thread priority 30 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 29 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 28 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 27 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 26 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 25 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 24 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 23 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 22 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 21 woke up.
(priority-sema) Back in main thread.
(priority-sema) end
EOF
pass;

```
由此可见线程需要按优先级从大到小被唤醒，也就是要求将信号量的等待队列从FIFO修改为按优先级管理。
#### ⑧priority-condvar
接下来分析条件变量
（1）.c测试文件
``` c
/* Tests that cond_signal() wakes up the highest-priority thread
   waiting in cond_wait(). */
#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/malloc.h"
#include "threads/synch.h"
#include "threads/thread.h"
#include "devices/timer.h"
  
static thread_func priority_condvar_thread;
static struct lock lock;
static struct condition condition;
void
test_priority_condvar (void)
{
  int i;
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  lock_init (&lock);//初始化锁
  cond_init (&condition);//初始化条件变量
  thread_set_priority (PRI_MIN);//将当前线程优先级设为最低值
  for (i = 0; i < 10; i++)
    {
      int priority = PRI_DEFAULT - (i + 7) % 10 - 1;//打乱优先级顺序
      char name[16];//线程名
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, priority_condvar_thread, NULL);//创建线程，试图获取锁，被阻塞在条件变量上
    }
  for (i = 0; i < 10; i++)
    {
      lock_acquire (&lock);//当前线程获取锁
      msg ("Signaling...");
      cond_signal (&condition, &lock);//唤醒一个在条件变量上等待的线程
      lock_release (&lock);//当前线程释放锁
    }
}

static void
priority_condvar_thread (void *aux UNUSED)
{
  msg ("Thread %s starting.", thread_name ());
  lock_acquire (&lock);//子线程试图获取锁
  cond_wait (&condition, &lock);//在条件变量上被阻塞
  msg ("Thread %s woke up.", thread_name ());//被唤醒后获取锁
  lock_release (&lock);//子线程释放锁
}
```

（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-condvar) begin
(priority-condvar) Thread priority 23 starting.
(priority-condvar) Thread priority 22 starting.
(priority-condvar) Thread priority 21 starting.
(priority-condvar) Thread priority 30 starting.
(priority-condvar) Thread priority 29 starting.
(priority-condvar) Thread priority 28 starting.
(priority-condvar) Thread priority 27 starting.
(priority-condvar) Thread priority 26 starting.
(priority-condvar) Thread priority 25 starting.
(priority-condvar) Thread priority 24 starting.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 30 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 29 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 28 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 27 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 26 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 25 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 24 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 23 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 22 woke up.
(priority-condvar) Signaling...
(priority-condvar) Thread priority 21 woke up.
(priority-condvar) end
EOF
pass;

```
和信号量的等待队列同理，条件变量的等待队列也需要从FIFO被修改为按优先级管理，从而使子线程按优先级唤醒。
#### ⑨priority-donate-chain
最后分析捐赠链
（1）.c测试文件
``` c
/* The main thread set its priority to PRI_MIN and creates 7 threads
   (thread 1..7) with priorities PRI_MIN + 3, 6, 9, 12, ...
   The main thread initializes 8 locks: lock 0..7 and acquires lock 0.
   When thread[i] starts, it first acquires lock[i] (unless i == 7.)
   Subsequently, thread[i] attempts to acquire lock[i-1], which is held by
   thread[i-1], except for lock[0], which is held by the main thread.
   Because the lock is held, thread[i] donates its priority to thread[i-1],
   which donates to thread[i-2], and so on until the main thread
   receives the donation.
   After threads[1..7] have been created and are blocked on locks[0..7],
   the main thread releases lock[0], unblocking thread[1], and being
   preempted by it.
   Thread[1] then completes acquiring lock[0], then releases lock[0],
   then releases lock[1], unblocking thread[2], etc.
   Thread[7] finally acquires & releases lock[7] and exits, allowing
   thread[6], then thread[5] etc. to run and exit until finally the
   main thread exits.
   In addition, interloper threads are created at priority levels
   p = PRI_MIN + 2, 5, 8, 11, ... which should not be run until the
   corresponding thread with priority p + 1 has finished.
   Written by Godmar Back <gback@cs.vt.edu> */

#include <stdio.h>
#include "tests/threads/tests.h"
#include "threads/init.h"
#include "threads/synch.h"
#include "threads/thread.h"
#define NESTING_DEPTH 8

struct lock_pair//指向锁的指针
  {
    struct lock *second;
    struct lock *first;
  };
static thread_func donor_thread_func;
static thread_func interloper_thread_func;

void
test_priority_donate_chain (void)
{
  int i;  
  struct lock locks[NESTING_DEPTH - 1];
  struct lock_pair lock_pairs[NESTING_DEPTH];
  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);//断言不在多级反馈队列中
  thread_set_priority (PRI_MIN);//将当前线程优先级设为最低值
  for (i = 0; i < NESTING_DEPTH - 1; i++)
    lock_init (&locks[i]);//初始化锁
  lock_acquire (&locks[0]);//当前线程获取锁0
  msg ("%s got lock.", thread_name ());
  for (i = 1; i < NESTING_DEPTH; i++)
    {
      char name[16];
      int thread_priority;
      snprintf (name, sizeof name, "thread %d", i);
      thread_priority = PRI_MIN + i * 3;//创建线程，优先级增大
      lock_pairs[i].first = i < NESTING_DEPTH - 1 ? locks + i: NULL;//如果第一把锁不为空，获取第一把锁
      lock_pairs[i].second = locks + i - 1;//试图获取第二把锁，但被前一个线程持有，所以被阻塞，并向前一个线程捐赠优先级
      thread_create (name, thread_priority, donor_thread_func, lock_pairs + i);//创建子线程
      msg ("%s should have priority %d.  Actual priority: %d.",
          thread_name (), thread_priority, thread_get_priority ());//当前线程的优先级逐层提升
      snprintf (name, sizeof name, "interloper %d", i);
      thread_create (name, thread_priority - 1, interloper_thread_func, NULL);//创建interloper线程，优先级比子线程小1，所以需要在子线程之后执行
    }
  lock_release (&locks[0]);//当前线程释放锁0，被子线程1获取
  msg ("%s finishing with priority %d.", thread_name (),
                                         thread_get_priority ());
}

static void
donor_thread_func (void *locks_)
{
  struct lock_pair *locks = locks_;
  if (locks->first)//如果第一把锁不为空，则获取第一把锁
    lock_acquire (locks->first);
  lock_acquire (locks->second);//试图获取第二把锁，被阻塞，向前一个线程捐赠优先级
  msg ("%s got lock", thread_name ());//获取第二把锁
  lock_release (locks->second);//释放第二把锁
  msg ("%s should have priority %d. Actual priority: %d",
        thread_name (), (NESTING_DEPTH - 1) * 3,//子线程的优先级为整条捐献链中最高的
        thread_get_priority ());
  if (locks->first)
    lock_release (locks->first);//释放第一把锁，被后一个线程获取
  msg ("%s finishing with priority %d.", thread_name (),
                                         thread_get_priority ());
}

static void
interloper_thread_func (void *arg_ UNUSED)
{
  msg ("%s finished.", thread_name ());
}
// vim: sw=2
```

（2）.ck期望结果
``` text
# -*- perl -*-
use strict;
use warnings;
use tests::tests;
check_expected ([<<'EOF']);
(priority-donate-chain) begin
(priority-donate-chain) main got lock.
(priority-donate-chain) main should have priority 3.  Actual priority: 3.
(priority-donate-chain) main should have priority 6.  Actual priority: 6.
(priority-donate-chain) main should have priority 9.  Actual priority: 9.
(priority-donate-chain) main should have priority 12.  Actual priority: 12.
(priority-donate-chain) main should have priority 15.  Actual priority: 15.
(priority-donate-chain) main should have priority 18.  Actual priority: 18.
(priority-donate-chain) main should have priority 21.  Actual priority: 21.
(priority-donate-chain) thread 1 got lock
(priority-donate-chain) thread 1 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 2 got lock
(priority-donate-chain) thread 2 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 3 got lock
(priority-donate-chain) thread 3 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 4 got lock
(priority-donate-chain) thread 4 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 5 got lock
(priority-donate-chain) thread 5 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 6 got lock
(priority-donate-chain) thread 6 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 7 got lock
(priority-donate-chain) thread 7 should have priority 21. Actual priority: 21
(priority-donate-chain) thread 7 finishing with priority 21.
(priority-donate-chain) interloper 7 finished.
(priority-donate-chain) thread 6 finishing with priority 18.
(priority-donate-chain) interloper 6 finished.
(priority-donate-chain) thread 5 finishing with priority 15.
(priority-donate-chain) interloper 5 finished.
(priority-donate-chain) thread 4 finishing with priority 12.
(priority-donate-chain) interloper 4 finished.
(priority-donate-chain) thread 3 finishing with priority 9.
(priority-donate-chain) interloper 3 finished.
(priority-donate-chain) thread 2 finishing with priority 6.
(priority-donate-chain) interloper 2 finished.
(priority-donate-chain) thread 1 finishing with priority 3.
(priority-donate-chain) interloper 1 finished.
(priority-donate-chain) main finishing with priority 0.
(priority-donate-chain) end
EOF
pass;

```
综合以上9个测试用例来看，需要实现以下功能：
- 将阻塞队列由FIFO修改为按优先级管理
- 增加一个记录对当前线程捐赠过优先级的线程+捐赠线程以及正在等待哪个锁的数据结构
- 当锁的释放顺序和优先级捐赠来源不一致时，恢复优先级时需要在初始优先级和剩余捐献列表中的优先值取最大值进行更新
- 实现优先级沿着嵌套捐赠的捐赠链传递更新，并要求让每次捐赠后拥有最高优先级的线程先运行，释放锁后不再拥有锁的线程恢复最初优先级
- 当捐赠链未结束时，如果当前线程优先级下调，应该在下调后的优先级和捐赠列表中的优先值中取最大值，而不应该立即下调
#### ⑩修改
##### （1）数据结构
- thread
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
  };
```
`original_priority`是线程被捐赠前最初的优先值，用于释放锁后恢复当前线程的优先值
`locks_held_by_thread`是记录当前线程持有的锁的列表
`lock_waited_by_thread`是指向当前线程正在等待的锁的指针
- lock
``` c
/* Lock. */
struct lock
  {
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
    
    /*新增：*/
    struct list_elem lock_elem;//被某个线程持有的锁，lock_held_by_thread列表的成员
    int max_priority;//阻塞队列中的线程的最大优先值
  };
```
`list_elem lock_elem`是`lock_held_by_thread`列表的成员
`max_priority`是等待该锁的线程列表中的最大优先值

将以上新增字段添加到线程的初始化函数`init_thread()`中：
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
  
  old_level = intr_disable ();
  list_push_back (&all_list, &t->allelem);
  // list_insert_ordered(&all_list,&t->allelem,(list_less_func*)&compareByPriority,NULL);
  intr_set_level (old_level);
}
```
除此之外，在设置线程优先值时也需要更新新增字段：
``` c
/* Sets the current thread's priority to NEW_PRIORITY. */
void
thread_set_priority (int new_priority) //设置优先值
{
  if(!thread_mlfqs)
  {
    enum intr_level old_level=intr_disable();//关中断
    struct thread* current_thread=thread_current();
    int old_priority=current_thread->priority;//当前线程当前的优先值
    current_thread->original_priority=new_priority;//修改优先值，但不立刻更新
    if(list_empty(&current_thread->locks_held_by_thread)||new_priority>old_priority)//如果当前线程没有被捐赠优先级&&新的优先值更大→更新优先值
    {
      current_thread->priority=new_priority;
      thread_yield();//将CPU让给优先级更高的线程
    }
    intr_set_level(old_level);//恢复调用前的中断状态
  }
  // thread_current ()->priority = new_priority;
  // thread_yield();
}
```
##### （2）函数
- `lockCmpByPriority()`：比较阻塞在锁上的线程的优先值
``` c
bool lockCmpByPriority(const struct list_elem *a,const struct list_elem *b,void *aux UNUSED)
{
  return list_entry(a,struct lock,lock_elem)->max_priority>list_entry(b,struct lock,lock_elem)->max_priority;
}
```
- `thread_update_priority()`：更新当前线程优先值
``` c
void thread_update_priority(struct thread*current_thread)//更新当前线程优先值
{
  enum intr_level old_level=intr_disable();//关中断
  int max_priority=current_thread->original_priority;//将最大优先值初始化为初始优先值
  int lock_priority;
  if(!list_empty(&current_thread->locks_held_by_thread))//如果当前线程持有一些锁
  {
    list_sort(&current_thread->locks_held_by_thread,lockCmpByPriority,NULL);//将锁按优先级排序
    lock_priority=list_entry(list_front(&current_thread->locks_held_by_thread),struct lock,lock_elem)->max_priority;//取其中最大的优先值
    if(lock_priority>max_priority)//如果比之前的最大优先值还大
    {
      max_priority=lock_priority;//更新原最大优先值
    }
  }
  current_thread->priority=max_priority;//更新当前线程的优先值为最大优先值
  intr_set_level(old_level);//恢复调用前的中断状态

}
```
- `lock_holder()`：记录当前线程持有的锁
``` c
void lock_holder(struct lock* lock)//记录当前线程持有的锁
{
  enum intr_level old_level=intr_disable();//关中断
  list_insert_ordered(&thread_current()->locks_held_by_thread,&lock->lock_elem,lockCmpByPriority,NULL);//将该锁按优先级插入锁列表
  if(lock->max_priority>thread_current()->priority)//如果该锁的优先值比当前线程的优先值高
  {
    thread_current()->priority=lock->max_priority;//更新当前线程的优先值
    thread_yield();//将CPU让给更高优先级的线程
  }
  intr_set_level(old_level);//恢复调用前的中断状态
}
```
- `priority_donator()`：进行优先级捐赠
``` c
void priority_donator(struct thread* donator)//进行优先级捐赠
{
  enum intr_level old_level=intr_disable();//关中断
  thread_update_priority(donator);//更新线程的优先值
  if(donator->status==THREAD_READY)//如果该线程被阻塞
  {
    list_remove(&donator->elem);//由于优先值改变，需要先将该线程移出ready队列
    list_insert_ordered(&ready_list,&donator->elem,compareByPriority,NULL);//然后按优先级重新插入ready队列
  }
  intr_set_level(old_level);//恢复调用前的中断状态
}
```

###### 将以上4个新增函数用于对`lock_acquire()`的更新中：
``` c
void
lock_acquire (struct lock *lock)//获取锁
{
  struct thread* current_thread=thread_current();//指向当前线程的指针
  struct lock *current_lock;//指向锁的指针
  enum intr_level old_level;//中断状态
  ASSERT (lock != NULL);//断言锁非空
  ASSERT (!intr_context ());//断言不在中断上下文中
  ASSERT (!lock_held_by_current_thread (lock));//断言当前线程不持有该锁
  if(current_lock->holder&&!thread_mlfqs)//如果锁已被持有&&不是多级反馈队列→进行优先级捐赠
  {
    current_thread->lock_waited_by_thread=current_lock;//将该锁标记为当前线程正在等待的锁
    current_lock=lock;
    while(current_thread->priority>current_lock->max_priority&&current_lock)//在捐赠链结束前持续更新优先值
    {
      current_lock->max_priority=current_thread->priority;//更新该锁的最大优先值
      priority_donator(current_lock->holder);//将优先值捐赠给持有者
      current_lock=current_lock->holder->lock_waited_by_thread;//递归捐赠
    }
  }
  sema_down (&lock->semaphore);//signal后试图获取锁
  old_level=intr_disable();//关中断
  current_thread=thread_current();
  if(!thread_mlfqs)
  {
    current_thread->lock_waited_by_thread=NULL;//获取锁后不再等待任何锁
    lock->max_priority=current_thread->priority;//更新该锁的优先值
    lock_holder(lock);//将锁添加到该线程持有的锁的列表中
  }
  lock->holder=current_thread;//将该线程标记为锁的持有者
  intr_set_level(old_level);//恢复为调用前的中断状态
}
```

- `thread_remove_lock()`：将线程从ready队列中移除
``` c
void thread_remove_lock(struct lock*lock)//将线程暂时从ready队列中移除
{
  enum intr_level old_level=intr_disable();//关中断
  list_remove(&lock->lock_elem);//删除当前线程
  thread_update_priority(thread_current());//更新当前线程优先值，用于稍后将该线程按优先级重新插入ready队列
  intr_set_level(old_level);//恢复调用前的中断状态
}
```

###### 将以上函数用于修改`lock_release()`中：
``` c
void
lock_release (struct lock *lock)//释放锁
{
  ASSERT (lock != NULL);//断言锁非空
  ASSERT (lock_held_by_current_thread (lock));//断言当前线程持有该锁
  lock->holder = NULL;//该锁不属于任何线程
  sema_up (&lock->semaphore);//signal，唤醒ready队列中优先级最高的线程
  if(!thread_mlfqs)//撤销捐赠
  {
    thread_remove_lock(lock);
  }
}
```
- `semaAndConditionCmpByPriority`：比较阻塞在信号量和条件变量的线程的优先值
``` c
bool semaAndConditionCmpByPriority(const struct list_elem *a,const struct list_elem *b,void *aux UNUSED)
{
  struct semaphore_elem *sema1=list_entry(a,struct semaphore_elem,elem);//线程的信号量
  struct semaphore_elem *sema2=list_entry(b,struct semaphore_elem,elem);
  return list_entry(list_front(&sema1->semaphore.waiters),struct thread,elem)->priority>list_entry(list_front(&sema2->semaphore.waiters),struct thread,elem)->priority;//取出被阻塞的线程队列中优先级最高的线程进行比较
}
```
- 修改`sema_down()`
```c
/* Down or "P" operation on a semaphore.  Waits for SEMA's value
   to become positive and then atomically decrements it.
   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but if it sleeps then the next scheduled
   thread will probably turn interrupts back on. */
void
sema_down (struct semaphore *sema)
{
  enum intr_level old_level;
  ASSERT (sema != NULL);//断言信号量非空
  ASSERT (!intr_context ());//断言不在中断上下文中
  old_level = intr_disable ();//关中断
  while (sema->value == 0) //信号量为0则阻塞线程
    {
      // list_push_back (&sema->waiters, &thread_current ()->elem);
      list_insert_ordered(&sema->waiters,&thread_current()->elem,compareByPriority,NULL);//将FIFO修改为按优先级插入
      thread_block ();//阻塞当前线程
    }
  sema->value--;//信号量-1
  intr_set_level (old_level);//恢复调用前的中断状态
}
```
- 修改`sema_up()`
``` c
/* Up or "V" operation on a semaphore.  Increments SEMA's value
   and wakes up one thread of those waiting for SEMA, if any.
   This function may be called from an interrupt handler. */
void
sema_up (struct semaphore *sema)
{
  enum intr_level old_level;
  ASSERT (sema != NULL);//断言信号量非空
  old_level = intr_disable ();//关中断
  if (!list_empty (&sema->waiters)) //如果有线程在等待该信号量
  {
    list_sort(&sema->waiters,compareByPriority,NULL);//将线程按优先级排序
    thread_unblock (list_entry (list_pop_front (&sema->waiters),struct thread, elem));//唤醒优先级最高的线程
  }
  sema->value++;//信号量+1
  thread_yield();//将CPU让给优先级更高的线程
  intr_set_level (old_level);//恢复调用前的中断状态
}
```
###### 将以上函数用于修改`cond_signal()`中：
``` c
/* If any threads are waiting on COND (protected by LOCK), then
   this function signals one of them to wake up from its wait.
   LOCK must be held before calling this function.
   An interrupt handler cannot acquire a lock, so it does not
   make sense to try to signal a condition variable within an
   interrupt handler. */
void
cond_signal (struct condition *cond, struct lock *lock UNUSED)
{
  ASSERT (cond != NULL);//断言条件变量非空
  ASSERT (lock != NULL);//断言锁非空
  ASSERT (!intr_context ());//断言不在中断上下文中
  ASSERT (lock_held_by_current_thread (lock));//断言当前线程持有该锁
  if (!list_empty (&cond->waiters)) //如果有线程在等待条件变量
  {
    list_sort(&cond->waiters,semaAndConditionCmpByPriority,NULL);//将线程按优先级排序
    sema_up (&list_entry (list_pop_front (&cond->waiters),struct semaphore_elem, elem)->semaphore);//唤醒优先级最高的线程
  }
}
```

# （四）`mlfqs-*`
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
$recent_{cpu}=(2*load_{avg})/(2*load_{avg}+1)*recent_{cpu}+nice$
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
$recent_{cpu}=(2*load_{avg})/(2*load_{avg}+1)*recent_{cpu}+nice$
③load_avg估算过去一分钟内平均准备运行的线程数。启动时初始化为0，并每秒重新计算一次，具体如下：
$load_{avg}=(59/60)*load_{avg}+(1/60)*ready_{threads}$
#### 6.不动点实算数
在上述公式中，优先级、nice值和ready_threads是整数，但recent_cpu和load_avg是实数，而Pintos内核不支持浮点运算，因此对实数的计算必须用整数进行模拟，也就是定点数。
定点数的思想是把一个有符号32位整数的最低的14位当成小数部分，这样一个整数x就代表真实数$\frac{x}{2^{14}}$。
比如，在计算$load_{avg}$中会用到$\frac{59}{60}$，那么就转换为$\frac{59}{60}*2^{14}=16110$。
文档中给出了在c中实现不动点算术运算的方法，其中$f=2^{14}=1<<14$：

| 操作                                             | 功能               |
| ---------------------------------------------- | ---------------- |
| $n*f$                                          | 将n转换为不动点         |
| $x/f$                                          | 将x转换为整数（四舍五入至零）  |
| `(x+f/2)/f` if `x>=0`<br>`(x-f/2)/f` if `x<=0` | 将x转换为整数（四舍五入至最近） |
| $x+y$                                          | 加x和y             |
| $x-y$                                          | 从x中减去y           |
| $x+n*f$                                        | 加x和n             |
| $x-n*f$                                        | 从x中减去n           |
| `((int64_t)x)*y/f`                             | 将x乘以y            |
| $x*n$                                          | 将x乘以n            |
| `((int64_t)x)*f/y`                             | 将x除以y            |
| $x/n$                                          | 将x除以n            |
|                                                |                  |

## 二、分析源代码&修改
### 1.实现不动点实算数
利用文档给出的公式，新增`thread/fixed_point.h`库：
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

#endif
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
（3）`thread_get_recent_cpu()`：返回当前recent_cpu的100倍，四舍五入至最接近的整数
``` c
int thread_get_recent_cpu(void)
{
  return FP_ROUND(FP_MULT_MIX(thread_current()->recent_cpu,100));
}
```
（4）`thread_get_load_avg()`：返回当前load_avg的100倍，四舍五入至最接近的整数
``` c
int thread_get_load_avg(void)
{
  return FP_ROUND(FP_MULT_MIX(load_avg,100));
}
```
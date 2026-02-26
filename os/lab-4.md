# （一）all pass截图
![[Pasted image 20251219104550.png]]

# （二）阅读文档
## ①project2主要需要修改userprog文件夹中的文件的函数，包括以下`.h/.c`文件，其中加粗的是需要进行修改的文件：

- **`process`**：加载 ELF 二进制文件并启动进程
- `pagedir`：一个简单的80x86硬件页表管理器，可调用其中的函数
- **`syscall`**：每当用户进程想要访问某些内核功能时，它都会调用系统调用；目前只是打印一条消息并终止用户进程，需要添加代码以执行系统调用所需的所有其他操作
- **`exception`**：当用户进程执行特权或禁止操作时，它会作为异常或错误进入内核，`expection`文件夹中的文件用于处理异常。目前，所有异常都只是打印一条消息并终止进程。project2的部分解决方案需要使用`page_fault()`在此文件中进行修改
- `gdt`：80x86是一种分段架构。全局描述符表GDT是一个描述正在使用的段的表。这些文件设置了全局描述符表
- `tss`：任务状态段TSS用于80x86架构任务切换，Pintos仅在用户进程进入中断处理程序时使用TSS来切换堆栈

## ②pintos在`filesys`文件夹中提供了一个简单但完整的文件系统
`filesys.h`和`file.h`中提供了接口，以了解如何使用文件系统
## ③pintos将虚拟内存分为用户虚拟内存和内核虚拟内存

1.用户虚拟内存

- 范围从虚拟地址0到`PHYS_BASE`（最多4GB），默认为`0xc0000000`（3GB）
- 是按进程计算的。当内核从一个进程切换到另一个进程时，它也会通过改变处理器的页面目录基寄存器来切换用户虚拟地址空间。`struct thread`包含指向进程页面表的指针
- 用户进程只能访问其自身的用户虚拟内存，访问内核虚拟内存会引发页面错误，通过`page_fault（）`处理，进程将被终止


2.内核虚拟内存

- 占用剩余的虚拟地址空间
- 是全局的。无论运行哪个用户进程或内核线程，它始终以相同的方式映射。在Pintos中，内核虚拟内存一对一映射到物理内存，虚拟地址`PHYS_BASE`访问物理地址0，虚拟地址`PHYS_BASE+0x1234`访问物理地址0x1234 ，依此类推，直到机器物理内存大小
- 内核进程可以访问内核虚拟内存，如果用户进程正在运行，也可以访问该进程的用户虚拟内存。然而，即使在内核中，尝试访问未映射用户虚拟地址的内存也会导致页面错误

## ④实施顺序
1.参数传递：每个用户程序都会在实现参数传递之前，立即发生页错误。在实现参数传递之前，你应该只运行程序，不传递命令行参数。尝试将参数传递给程序时，这些参数会包含在程序名称中，但程序很可能会失败。
2.用户内存访问：所有系统调用都需要读取用户内存，很少有系统调用需要写入用户内存。
3.系统调用基础设施：实现从用户栈读取系统调用号，并据此向处理程序发送。
4.`exit()`系统调用：每个正常结束的用户程序都会调用`exit()`。
5.`write()`系统调用：用于写入 fd 1（系统控制台）。所有的测试程序都是写入控制台，所以在`write()`可用之前，它们都会出故障。
6.把`process_wait()` 改成无限循环永远等待。现在提供的实现会立即返回，因此 Pintos 在任何进程实际运行之前就会关闭，需要提供正确的实现。
## ⑤通过阅读project2部分的内容，可以得知需要完成以下工作：
### （1）进程终止消息
每当用户进程终止时，由于调用`exit()`系统调用等原因，会打印进程名称和退出代码，(格式：`printf("%s: exit(%d)\n", ...);`)，打印的名称应为传递给 `process_execute()` 的全名，省略命令行参数。当非用户进程的内核线程终止或调用`halt()`系统调用时，不要打印这些消息。当进程加载失败时，该消息是可选的。
### （2）参数传递
目前`process_execute()`不支持将参数传递给新的进程，需要通过扩展`process_execute()`来通过空格将程序文件名划分为单词：第一个词是程序名，第二个词是第一个参数，依此类推。
e.g.`process_execute("grep foo bar")`应运行`grep`来传递`foo`和`bar`
在命令行中，多个空格等同于一个空格，可以对命令行参数的长度设定合理的限制，可以用任何方式解析参数字符串（提示：`strtok_r()`）。
### （3）访问用户内存
作为系统调用的一部分，内核通常必须通过用户程序提供的指针访问内存。内核必须非常小心，因为用户可以传递空指针、指向未映射虚拟内存的指针，或指向内核虚拟地址空间（`PHYS_BASE`上方）的指针。所有这些类型的无效指针都必须被拒绝，且不会损害内核或其他正在运行的进程，方法是终止有问题的进程并释放其资源。
解决方法：

- 验证先判断用户提供的指针的有效性，然后解引用
- 只检查用户指针指向`PHYS_BASE`下方，然后解引用

无论哪种情况，都需要确保不要泄露资源
### （4）系统调用
需要将以下系统调用的框架填补完整：

- `void halt(void)`
- `void exit(int status)`
- `pit_t exec(const char *cmd_line)`
- `int wait(pid_t pid)`
- `bool create(const char *file, unsigned initial_size)`
- `bool remove(const char *file)`
- `int open(const char *file)`
- `int filesize(int fd)`
- `int read(int fd,void *buffer,unsigned size)`
- `int write(int fd,const void *buffer,unsigned size)`
- `void seek(int fd,unsigned position)`
- `unsigned tell(int fd)`
- `void close(int fd)`

### （5）拒绝写入可执行文件
添加代码来阻止作为可执行文件的文件写入。许多作系统这样做是因为如果进程试图运行正在磁盘上更改的代码，结果会变得不可预测。可以用`file_deny_write()` 来防止写入打开的文件。调用文件上的 `file_allow_write()` 会重新启用它们（除非被其他打开器拒绝写入）。关闭文件也会重新启用写入。因此，要拒绝对进程可执行文件的写入，必须在进程仍在运行时保持该程序开启。
# （三）实现
接下来按文档中的建议实现顺序进行修改
### 一、参数传递
在现版本中，系统不支持识别用户输入的参数，可通过解析字符串和操作栈指针`esp`来正确解析命令字符串、加载可执行文件并将参数正确压入新进程的用户栈，以便程序启动后可通过`int main(int argc,char *argv[])`进行访问。
#### 1.`strtok_k()`
文档提示使用`lib/string.h`中的`strtok_r()`函数来解析参数字符串,用于将字符串按分隔符拆分成词，其中第一次调用时s为原始字符串，后续调用时s=NULL。
``` c
char *strtok_r (char *s, const char *delimiters, char **save_ptr);
```
#### 2.修改`process_execute()`
原`process_execute()`只能支持无参数的用户程序
``` c
/* Starts a new thread running a user program loaded from  
   FILENAME.  The new thread may be scheduled (and may even exit)  
   before process_execute() returns.  Returns the new process's  
   thread id, or TID_ERROR if the thread cannot be created. */  
tid_t  
process_execute (const char *file_name)  
{  
  char *fn_copy;  
  tid_t tid;  
  /* Make a copy of FILE_NAME.  
     Otherwise there's a race between the caller and load(). */  
  fn_copy = palloc_get_page (0);  
  if (fn_copy == NULL)  
    return TID_ERROR;  
  strlcpy (fn_copy, file_name, PGSIZE); //file_name为可执行文件名   
  /* Create a new thread to execute FILE_NAME. */  
  tid = thread_create (file_name, PRI_DEFAULT, start_process, fn_copy);  
  if (tid == TID_ERROR)   
    palloc_free_page (fn_copy);  
  return tid;   
}
```
并且调用的`start_process()`不支持解析参数，也就是传入的`file_name`整个会被视为一个线程名
``` c
/** A thread function that loads a user process and starts it
   running. */
static void
start_process (void *file_name_)
{
  char *file_name = file_name_;
  struct intr_frame if_;
  bool success;

  /* Initialize interrupt frame and load executable. */
  memset (&if_, 0, sizeof if_);
  if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
  if_.cs = SEL_UCSEG;
  if_.eflags = FLAG_IF | FLAG_MBS;
  success = load (file_name, &if_.eip, &if_.esp);

  /* If load failed, quit. */
  palloc_free_page (file_name);
  if (!success) 
    thread_exit ();

  /* Start the user process by simulating a return from an
     interrupt, implemented by intr_exit (in
     threads/intr-stubs.S).  Because intr_exit takes all of its
     arguments on the stack in the form of a `struct intr_frame',
     we just point the stack pointer (%esp) to our stack frame
     and jump to it. */
  asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
  NOT_REACHED ();
}
```
由于`strtok_r()`会将分隔符替换成`\0`，会修改原始字符串，如果像原`process_execute()`那样只执行一次`fn_copy()`和`strlcpy()`，会导致父进程和子进程同时读写同一块内存
``` c
char *
strtok_r (char *s, const char *delimiters, char **save_ptr) 
{
  char *token;
  
  ASSERT (delimiters != NULL);
  ASSERT (save_ptr != NULL);

  /* If S is nonnull, start from it.
     If S is null, start from saved position. */
  if (s == NULL)
    s = *save_ptr;
  ASSERT (s != NULL);

  /* Skip any DELIMITERS at our current position. */
  while (strchr (delimiters, *s) != NULL) 
    {
      /* strchr() will always return nonnull if we're searching
         for a null byte, because every string contains a null
         byte (at the end). */
      if (*s == '\0')
        {
          *save_ptr = s;
          return NULL;
        }

      s++;
    }

  /* Skip any non-DELIMITERS up to the end of the string. */
  token = s;
  while (strchr (delimiters, *s) == NULL)
    s++;
  if (*s != '\0') 
    {
      *s = '\0';//将分隔符替换为\0
      *save_ptr = s + 1;
    }
  else 
    *save_ptr = s;
  return token;
}
```
因此在对传入`process_execute()`传入的`file_name`进行处理前，需要两次调用`palloc_get_page()`，一份用于解析字符串，一份用于传给`start_process()`
``` c
tid_t
process_execute (const char *file_name) 
{
  char *fn_copy1,*fn_copy2;//命令行拷贝的指针
  tid_t tid;//新线程id
  /*为fn_copy0申请一页内存*/
  fn_copy1=palloc_get_page(0);
  if(fn_copy1==NULL)//申请失败
	  return TID_ERROR;//防止对空指针strlcpy
  /* Make a copy of FILE_NAME.
    Otherwise there's a race between the caller and load(). */
  fn_copy2=palloc_get_page(0);
  if(fn_copy2==NULL)//申请失败
  {
    palloc_free_page(fn_copy1);//释放已分配成功的fn_copy1
    return TID_ERROR;
  }

  /*复制2次file_name*/
  strlcpy(fn_copy1,file_name,PGSIZE);//用于解析
  strlcpy(fn_copy2,file_name,PGSIZE);//用于传给start_process

  char *save_ptr;//strtok_r的状态保存指针
  char *cmd=strtok_r(fn_copy1," ",&save_ptr);//可执行文件名
  
  tid=thread_create(cmd,PRI_DEFAULT,start_process,fn_copy2);//创建新进程
  palloc_free_page(fn_copy1);//释放
  if(tid==TID_ERROR)//如果创建失败
  {
    palloc_free_page(fn_copy2); //释放
    return tid;
  }
  return tid;
}
```
#### 3.修改`start_process()`
在加载进程时需要将参数压入用户栈，需要构建像文档中描述的用户栈：
![[Pasted image 20251219171903.png]]
虚拟内存的地址空间=4GB（0x00000000-0xffffffff）

- 用户虚拟内存=3GB（0x00000000-0xc0000000）
- 内核虚拟内存=1GB（0xc0000000-0xffffffff）

初始栈顶`PHYS_BASE`=0xc0000000，向下增长，esp--
![[Pasted image 20251219172049.png]]
文档中给出了一个例子：
命令：`/bin/ls -l foo bar`
esp初始化为0xbfffffcc

- 第1-4行：字符串参数
- 第5行：对齐填充
- 第6-10行：argv指针数组，指向参数地址，以null结尾（`argv[argc]=0`）
- 第11行：argv，指向argv[0]
- 第12行：argc：参数个数
- 第13行：返回地址占位

按照要求的栈结构构建`push_argument()`
```c
void push_argument (void **esp, int argc, int argv[])
{
  *esp=(int)*esp&0xfffffffc;//将esp4字节对齐
  *esp=*esp-4;//esp下移
  *(int*)*esp=0;//argv以null结尾：argv[argc]=0
  for(int i=argc-1;i>=0;i--)//由于栈从上向下增长，因此倒序压入argv数组
  {
    *esp=*esp-4;//esp下移
    *(int*)*esp=argv[i];//写入地址
  }
  *esp=*esp-4;//esp下移
  *(int *)*esp=(int)*esp+4;//写入**argv（指向argv[0]）
  *esp=*esp-4;//esp下移
  *(int *)*esp=argc;//写入argc
  *esp=*esp-4;//esp下移
  *(int *)*esp=0;//写入返回地址占位
}
```
将字符串解析和`push_argument()`添加到`start_process()`中
``` c
static void
start_process (void *file_name_)
{
  char *file_name = file_name_;//指向命令行字符串
  struct intr_frame if_;//保存将来切到用户态需要的寄存器状态的中断帧
  bool success;//是否成功加载

  /*新增*/
  char *fn_copy=malloc(strlen(file_name)+1);//为命令行字符串申请大小为len+'\0'的堆内存
  strlcpy(fn_copy,file_name,strlen(file_name)+1);//复制命令行字符串

  /* Initialize interrupt frame 初始化中断帧*/
  memset (&if_, 0, sizeof if_);
  if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
  if_.cs = SEL_UCSEG;
  if_.eflags = FLAG_IF | FLAG_MBS;

  char *token;//遍历参数
  char *save_ptr;//strtok_r的状态指针
  file_name=strtok_r(file_name," ",&save_ptr);//解析字符串，分割出可执行文件名
  success=load(file_name,&if_.eip,&if_.esp);//将用户程序二进制文件加载进内存
  
  /*新增*/
  if(success)//成功加载→压栈
  {
    int argc=0;//参数个数
    /* The number of parameters can't be more than 50 in the test case */
    int argv[50];//暂存参数
    for(token=strtok_r(fn_copy," ",&save_ptr);token!=NULL; token=strtok_r(NULL," ",&save_ptr))//将参数复制到栈上并记录位置
    {
      if_.esp=if_.esp-(strlen(token)+1);//移动esp
      memcpy(if_.esp,token,strlen(token)+1);//将参数写入esp指向的位置
      argv[argc++]=(int)if_.esp;//记录参地址
    }
    push_argument(&if_.esp, argc, argv);//将参数的地址压入用户栈
  }
  /* If load failed, quit. */
  palloc_free_page (file_name);//释放页
  free(fn_copy);//释放指针
  if (!success) //加载失败则退出线程
  {
    thread_exit ();
  }
    
  /* Start the user process by simulating a return from an
     interrupt, implemented by intr_exit (in
     threads/intr-stubs.S).  Because intr_exit takes all of its
     arguments on the stack in the form of a `struct intr_frame',
     we just point the stack pointer (%esp) to our stack frame
     and jump to it. */
  asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
  NOT_REACHED ();
}
```
### 二、进程终止消息
需要在触发进程终止时打印终止消息
#### 1.`thread`结构体
在`thread`结构体中新增`int st_exit`字段，记录退出状态
``` c
int st_exit;
```
并在`init_thread()`初始化
``` c
 t->st_exit=UINT32_MAX;
```
#### 2.`kill()`
`exception.c`中的`kill()`函数用于在程序运行出错触发中断时，判断该错误发生在用户态/内核态，并采取相应的处理措施。
``` c
/** Handler for an exception (probably) caused by a user process. */
static void
kill (struct intr_frame *f) 
{
  /* This interrupt is one (probably) caused by a user process.
     For example, the process might have tried to access unmapped
     virtual memory (a page fault).  For now, we simply kill the
     user process.  Later, we'll want to handle page faults in
     the kernel.  Real Unix-like operating systems pass most
     exceptions back to the process via signals, but we don't
     implement them. */
     
  /* The interrupt frame's code segment value tells us where the
     exception originated. */
  switch (f->cs)
    {
    case SEL_UCSEG://用户态异常
      /* User's code segment, so it's a user exception, as we
         expected.  Kill the user process.  */
      printf ("%s: dying due to interrupt %#04x (%s).\n",
              thread_name (), f->vec_no, intr_name (f->vec_no));//打印调试信息
      intr_dump_frame (f);//打印当前寄存器状态
      thread_exit (); //终止用户进程

    case SEL_KCSEG://内核态异常
      /* Kernel's code segment, which indicates a kernel bug.
         Kernel code shouldn't throw exceptions.  (Page faults
         may cause kernel exceptions--but they shouldn't arrive
         here.)  Panic the kernel to make the point.  */
      intr_dump_frame (f);
      PANIC ("Kernel bug - unexpected interrupt in kernel"); 

    default:
      /* Some other code segment?  Shouldn't happen.  Panic the
         kernel. */
      printf ("Interrupt %#04x (%s) in unknown segment %04x\n",
             f->vec_no, intr_name (f->vec_no), f->cs);
      thread_exit ();
    }
}
```
而被调用的`thread_exit()`不支持在触发进程终止时打印终止消息
```c
/** Deschedules the current thread and destroys it.  Never
   returns to the caller. */
void
thread_exit (void) 
{
  ASSERT (!intr_context ());

#ifdef USERPROG
  process_exit ();
#endif

  /* Remove thread from all threads list, set our status to dying,
     and schedule another process.  That process will destroy us
     when it calls thread_schedule_tail(). */
  intr_disable ();
  list_remove (&thread_current()->allelem);
  thread_current ()->status = THREAD_DYING;
  schedule ();
  NOT_REACHED ();
}
```
因此添加打印终止信息：
```c
/** Deschedules the current thread and destroys it.  Never
   returns to the caller. */
void
thread_exit (void) 
{
  ASSERT (!intr_context ());

#ifdef USERPROG
  process_exit ();
#endif

  /* Remove thread from all threads list, set our status to dying,
     and schedule another process.  That process will destroy us
     when it calls thread_schedule_tail(). */
  intr_disable ();
  
  printf ("%s:exit(%d)\n",thread_name(),thread_current()->st_exit);//终止信息
  
  list_remove (&thread_current()->allelem);
  thread_current ()->status = THREAD_DYING;
  schedule ();
  NOT_REACHED ();
}
```
### 三、访问用户内存
作为系统调用的一部分，内核通常必须通过用户程序提供的指针访问内存。而用户可以传递空指针、指向未映射虚拟内存的指针，或指向内核虚拟地址空间（`PHYS_BASE` 上方）的指针，需要终止有问题的进程来拒绝这些无效指针，并确保不会损害内核或其他正在运行的进程。
文档中提供了2种方法，其中第2种方法是检查指向`PHYS_BASE`下方的用户指针，然后取消引用。无效的用户指针会导致页面错误，可以通过修改代码中的`page_fault()`来处理 `userprog/exception.c`。
文档中提供了2个函数：
①`get_user()`：从用户态地址`uaddr`读取一个字节
``` c
/* Reads a byte at user virtual address UADDR.
   UADDR must be below PHYS_BASE.
   Returns the byte value if successful, -1 if a segfault
   occurred. */
static int
get_user (const uint8_t *uaddr)
{
  int result;
  asm ("movl $1f, %0; movzbl %1, %0; 1:"
       : "=&a" (result) : "m" (*uaddr));
  return result;
}
``` 

- `uaddr`＜`PHYS_BASE`-成功，返回字节值
- `uaddr`非法-失败，CPU触发中断，返回`-1`

②`put_user()`：向用户态地址`udst`写入一个字节
``` c
/* Writes BYTE to user address UDST.
   UDST must be below PHYS_BASE.
   Returns true if successful, false if a segfault occurred. */
static bool
put_user (uint8_t *udst, uint8_t byte)
{
  int error_code;
  asm ("movl $1f, %0; movb %b2, %1; 1:"
       : "=&a" (error_code), "=m" (*udst) : "q" (byte));
  return error_code != -1;
}
```

- 写入成功-`error_code!=-1`
- 写入失败-触发页错误，`error_code=-1`

>这些函数都假设用户地址已经被验证为低于`PHYS_BASE`。它们还假设已经修改了`page_fault()`，使得内核中的页错误仅仅是将`eax`设置为`0xffffffff`，并将它的旧值复制到`eip`中。

因此首先需要按这段要求修改`page_fault()`-用于处理缺页异常
``` c
/** Page fault handler.  This is a skeleton that must be filled in
   to implement virtual memory.  Some solutions to project 2 may
   also require modifying this code.

   At entry, the address that faulted is in CR2 (Control Register
   2) and information about the fault, formatted as described in
   the PF_* macros in exception.h, is in F's error_code member.  The
   example code here shows how to parse that information.  You
   can find more information about both of these in the
   description of "Interrupt 14--Page Fault Exception (#PF)" in
   [IA32-v3a] section 5.15 "Exception and Interrupt Reference". */
static void
page_fault (struct intr_frame *f) 
{
  bool not_present;  /**< True: not-present page, false: writing r/o page. */
  bool write;        /**< True: access was write, false: access was read. */
  bool user;         /**< True: access by user, false: access by kernel. */
  void *fault_addr;  /**< Fault address. */

  /* Obtain faulting address, the virtual address that was
     accessed to cause the fault.  It may point to code or to
     data.  It is not necessarily the address of the instruction
     that caused the fault (that's f->eip).
     See [IA32-v2a] "MOV--Move to/from Control Registers" and
     [IA32-v3a] 5.15 "Interrupt 14--Page Fault Exception
     (#PF)". */
  asm ("movl %%cr2, %0" : "=r" (fault_addr));

  /* Turn interrupts back on (they were only off so that we could
     be assured of reading CR2 before it changed). */
  intr_enable ();

  /* Count page faults. */
  page_fault_cnt++;

  /* Determine cause. */
  not_present = (f->error_code & PF_P) == 0;
  write = (f->error_code & PF_W) != 0;
  user = (f->error_code & PF_U) != 0;

  /* To implement virtual memory, delete the rest of the function
     body, and replace it with code that brings in the page to
     which fault_addr refers. */
  printf ("Page fault at %p: %s error %s page in %s context.\n",
          fault_addr,
          not_present ? "not present" : "rights violation",
          write ? "writing" : "reading",
          user ? "user" : "kernel");
  kill (f);
}
```
原函数会终止进程并打印故障信息，但如果缺页错误发生在内核态，直接终止进程会导致系统崩溃
``` c
/* Page fault handler.  This is a skeleton that must be filled in
   to implement virtual memory.  Some solutions to project 2 may
   also require modifying this code.

   At entry, the address that faulted is in CR2 (Control Register
   2) and information about the fault, formatted as described in
   the PF_* macros in exception.h, is in F's error_code member.  The
   example code here shows how to parse that information.  You
   can find more information about both of these in the
   description of "Interrupt 14--Page Fault Exception (#PF)" in
   [IA32-v3a] section 5.15 "Exception and Interrupt Reference". */
static void
page_fault (struct intr_frame *f) 
{
  bool not_present;  /* True: not-present page, false: writing r/o page. */
  bool write;        /* True: access was write, false: access was read. */
  bool user;         /* True: access by user, false: access by kernel. */
  void *fault_addr;  /* Fault address. */

  /* Obtain faulting address, the virtual address that was
     accessed to cause the fault.  It may point to code or to
     data.  It is not necessarily the address of the instruction
     that caused the fault (that's f->eip).
     See [IA32-v2a] "MOV--Move to/from Control Registers" and
     [IA32-v3a] 5.15 "Interrupt 14--Page Fault Exception
     (#PF)". */
  asm ("movl %%cr2, %0" : "=r" (fault_addr));

  /* Turn interrupts back on (they were only off so that we could
     be assured of reading CR2 before it changed). */
  intr_enable ();

  /* Count page faults. */
  page_fault_cnt++;

  /* Determine cause. */
  not_present = (f->error_code & PF_P) == 0;
  write = (f->error_code & PF_W) != 0;
  user = (f->error_code & PF_U) != 0;
  
  if (!user)
  {
      f->eip = f->eax;//将eax的旧值复制到eip中
      f->eax = -1;//将eax设置为0xffffffff
      return;
  }
  /* To implement virtual memory, delete the rest of the function
     body, and replace it with code that brings in the page to
     which fault_addr refers. */
  printf ("Page fault at %p: %s error %s page in %s context.\n",
          fault_addr,
          not_present ? "not present" : "rights violation",
          write ? "writing" : "reading",
          user ? "user" : "kernel");
  kill (f);
}
```
此外还需要将出错的线程的`st_exit=-1`并终止线程：
``` c
void 
exit_special (void)
{
  thread_current()->st_exit=-1;
  thread_exit ();
}
```
新增`check_ptr2()`函数，调用以上函数来判断指针是否有效：
``` c
void * check_ptr2(const void *vaddr)
{ 
  if (!is_user_vaddr(vaddr))//如果地址不合法
  {
    exit_special();//将st_exit设为-1并终止该线程
  }
  void *ptr = pagedir_get_page (thread_current()->pagedir, vaddr);//在当前进程的页表中查找该虚拟地址对应的物理地址
  if (!ptr)////如果是指向未映射虚拟内存的指针
  {
    exit_special ();//将st_exit设为-1并终止该线程
  }
  //检查跨页参数的地址
  uint8_t *check_byteptr = (uint8_t *) vaddr;//以字节为单位进行指针加减
  for (uint8_t i = 0; i < 4; i++) //4：系统调用参数以4字节为单位
  {
    if (get_user(check_byteptr + i) == -1)//如果地址不合法
    {
      exit_special ();//将st_exit设为-1并终止该线程
    }
  }

  return ptr;
}
```
### 四、系统调用
#### 1.原理
<font color="#4f6128">在80x86架构中，指令</font>`int`<font color="#4f6128">是调用系统调用的最常用方式。该指令的处理方式与其他软件异常相同。在 Pintos 中，用户程序调用 </font>`int $0x30` <font color="#4f6128">系统调用。系统调用号及任何额外参数应在调用中断前以正常方式推送到栈中。</font>
<font color="#4f6128">因此，当系统调用处理程序</font>`syscall_handler()`<font color="#4f6128">获得控制权时，系统调用号位于调用者的栈指针处的32位字中，第一个参数位于下一个更高地址的32位字中，以此类推。调用者的栈指针可以通过</font>`syscall_handler()`<font color="#4f6128">传递给它的</font>`struct intr_frame`<font color="#4f6128">的</font>`esp`<font color="#4f6128">成员来访问。（</font>`struct intr_frame`<font color="#4f6128">位于内核栈上。）</font>
<font color="#4f6128">80x86函数返回值的约定是将它们放在</font>`EAX`<font color="#4f6128">寄存器中。返回值的系统调用可以通过修改</font>`struct intr_frame`<font color="#4f6128">的</font>`eax`<font color="#4f6128">成员来实现。</font>

#### 2.要求
<font color="#4f6128">在</font>`userprog/syscall.c`<font color="#4f6128">中实现系统调用处理程序。我们提供的框架实现通过终止进程来处理系统调用。它需要获取系统调用编号，然后获取任何系统调用参数，并执行适当的操作。</font>
<font color="#4f6128">你必须同步系统调用，以便任何数量的用户进程可以同时进行。特别是，从多个线程同时调用</font>`filesys`<font color="#4f6128">目录中提供的文件系统代码是不安全的。你的系统调用实现必须将文件系统代码视为一个临界区。别忘了</font>`process_execute()`<font color="#4f6128">也会访问文件。目前，我们不建议修改</font>`filesys`<font color="#4f6128">目录中的代码。</font>
<font color="#4f6128">我们在</font> `lib/user/syscall.c` <font color="#4f6128">中为每个系统调用提供了一个用户级函数。这些函数为用户进程提供了从 C 程序中调用每个系统调用的方法。每个函数都使用一小段内联汇编代码来调用系统调用，并在适当的情况下返回系统调用的返回值。</font>
<font color="#4f6128">实现以下系统调用。列出的原型是包含</font> `lib/user/syscall.h` <font color="#4f6128">的用户程序所看到的原型。(这个头以及 </font>`lib/user` <font color="#4f6128">中的所有其他头只供用户程序使用。) 每个系统调用的系统调用编号在</font> `lib/syscall-nr.h` <font color="#4f6128">中定义：</font>

``` c
#ifndef __LIB_SYSCALL_NR_H
#define __LIB_SYSCALL_NR_H

/** System call numbers. */
enum 
  {
    /* Projects 2 and later. */
    SYS_HALT,                   /**< Halt the operating system. */
    SYS_EXIT,                   /**< Terminate this process. */
    SYS_EXEC,                   /**< Start another process. */
    SYS_WAIT,                   /**< Wait for a child process to die. */
    SYS_CREATE,                 /**< Create a file. */
    SYS_REMOVE,                 /**< Delete a file. */
    SYS_OPEN,                   /**< Open a file. */
    SYS_FILESIZE,               /**< Obtain a file's size. */
    SYS_READ,                   /**< Read from a file. */
    SYS_WRITE,                  /**< Write to a file. */
    SYS_SEEK,                   /**< Change position in a file. */
    SYS_TELL,                   /**< Report current position in a file. */
    SYS_CLOSE,                  /**< Close a file. */

    /* Project 3 and optionally project 4. */
    SYS_MMAP,                   /**< Map a file into memory. */
    SYS_MUNMAP,                 /**< Remove a memory mapping. */

    /* Project 4 only. */
    SYS_CHDIR,                  /**< Change the current directory. */
    SYS_MKDIR,                  /**< Create a directory. */
    SYS_READDIR,                /**< Reads a directory entry. */
    SYS_ISDIR,                  /**< Tests if a fd represents a directory. */
    SYS_INUMBER                 /**< Returns the inode number for a fd. */
  };

#endif /**< lib/syscall-nr.h */
```
系统调用涉及到对用户内存的访问，因此需要在操作前检查地址和指针的合法性。对于只需从用户栈读取参数的系统调用，需要检查参数所在的地址是否合法；对于可能跨页的参数（比如字符串），除了检查参数所在的地址是否合法外，还需要检查指针是否合法；对于需要读取或写入缓冲区的系统调用，需要检查用户栈是否包含足够可访问的内存区域并更激进地检查从用户栈中获取的用户提供的指针。
##### (1)`System Call: void halt (void)`
<font color="#4f6128">通过调用</font> `shutdown_power_off()`<font color="#4f6128">（声明在</font> `devices/shutdown.h` 中）<font color="#4f6128">终止 Pintos。这应该很少使用，因为你可能会丢失一些关于可能死锁情况的信息等。</font>
直接调用`shutdown_power_off()`：
``` c
void 
sys_halt (struct intr_frame* f)
{
  shutdown_power_off();
}
```

##### (2)`System Call: void exit (int status)`
`exit()`<font color="#4f6128">会立即停止当前用户程序的运行，并将一个整数状态码返回给内核。如果该进程的父进程正在通过</font>`wait()`<font color="#4f6128">等待它，父进程将会接收到这个状态码。通常情况下，状态码0表示程序成功执行，非零值则表示发生了各种错误。</font>
``` c
void 
sys_exit (struct intr_frame* f)
{
  uint32_t *user_ptr=f->esp;//获取栈指针
  check_ptr2(user_ptr+1);//系统调用的第一个参数状态码位于栈顶偏移4字节处，确保该地址在PHYS_BASE之下，且已正确映射，防止内核访问非法内存
  *user_ptr++;//指针指向状态码
  thread_current()->st_exit=*user_ptr;//读取当前线程的状态码
  thread_exit();//终止当前线程
}
```
##### (3)`System Call: pid_t exec (const char *cmd_line)`
##### Ⅰ执行由 `cmd_line` 指定的可执行文件，传递任何给定的参数，并返回新进程的程序 ID (pid)。
``` c
void 
sys_exec (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查参数所在的地址是否合法
  check_ptr2 (*(user_ptr + 1));//检查字符串本身的地址是否合法
  *user_ptr++;//指针指向第一个参数
  f->eax = process_execute((char*)* user_ptr);//返回pid
}
```
##### Ⅱ如果程序因任何原因无法加载或运行，必须返回`pid -1`，否则该 </font>`pid` 不应有效。
对于该要求，参数传递部分的`process_execute()`中调用的`exit_special()`实现了返回`pid -1`的功能：
``` c
void 
exit_special (void)
{
  thread_current()->st_exit=-1;
  thread_exit ();
}
```
##### Ⅲ因此，父进程无法从 `exec` 返回，直到它知道子进程是否成功加载了其可执行文件。你必须使用适当的同步机制来确保这一点。
对于这部分要求，在`process_execute()`中添加同步机制：
首先在`thread`结构体中增加sema字段，用于同步：
``` c
struct semaphore sema;
```
并在`init_thread()`初始化为0：
``` c
 sema_init(&t->sema,0);
```
在结尾添加同步机制：
``` c
tid_t
process_execute (const char *file_name) 
{
  char *fn_copy1,*fn_copy2;//命令行拷贝的指针
  tid_t tid;//新线程id
  /*为fn_copy0申请一页内存*/
  fn_copy1=palloc_get_page(0);
  if(fn_copy1==NULL)//申请失败
	  return TID_ERROR;//防止对空指针strlcpy
  /* Make a copy of FILE_NAME.
    Otherwise there's a race between the caller and load(). */
  fn_copy2=palloc_get_page(0);
  if(fn_copy2==NULL)//申请失败
  {
    palloc_free_page(fn_copy1);//释放已分配成功的fn_copy1
    return TID_ERROR;
  }

  /*复制2次file_name*/
  strlcpy(fn_copy1,file_name,PGSIZE);//用于解析
  strlcpy(fn_copy2,file_name,PGSIZE);//用于传给start_process

  char *save_ptr;//strtok_r的状态保存指针
  char *cmd=strtok_r(fn_copy1," ",&save_ptr);//可执行文件名
  
  tid=thread_create(cmd,PRI_DEFAULT,start_process,fn_copy2);//创建新进程
  palloc_free_page(fn_copy1);//释放
  if(tid==TID_ERROR)//如果创建失败
  {
    palloc_free_page(fn_copy2); //释放
    return tid;
  }
  
  /*新增*/
  sema_down(&thread_current()->sema);//阻塞父进程
  if (!thread_current()->success)//如果子进程未成功加载其可执行文件
  {
	  return TID_ERROR;//返回error
  }
  return tid;//如果子进程成功加载其可执行文件-返回子进程pid
}
```
##### (4)`System Call: int wait (pid_t pid)`
<font color="#4f6128">等待子进程pid并获取子进程的退出状态。</font>
<font color="#4f6128">如果pid仍然存活，则等待它终止。然后，返回pid传递给</font>`exit`<font color="#4f6128">的状态。如果pid没有调用</font>`exit()`<font color="#4f6128">，但被内核终止（例如由于异常被杀死），</font>`wait(pid)`<font color="#4f6128">必须返回-1。父进程等待在父进程调用</font>`wait`<font color="#4f6128">时已经终止的子进程是完全合法的，但内核必须仍然允许父进程检索其子进程的退出状态，或者得知子进程被内核终止。</font>
<font color="#4f6128">如果以下任一条件为真，则必须立即失败并返回-1：</font>

- <font color="#4f6128">pid 并不直接指向调用进程的子进程。只有当调用进程从成功的</font> `exec` <font color="#4f6128">调用中接收到 pid 作为返回值时，pid 才是调用进程的直接子进程。请注意，子进程不会被继承：如果 A 创建了子进程 B，而 B 又创建了子进程 C，那么 A 无法等待 C，即使 B 已经死亡。进程 A 对 </font>`wait(C)` <font color="#4f6128">的调用必须失败。类似地，如果父进程在子进程之前退出，孤儿进程不会被分配给新的父进程。</font>
- <font color="#4f6128">调用</font>`wait`<font color="#4f6128">的进程已经在pid上调用过</font>`wait`<font color="#4f6128">了。也就是说，一个进程最多只能等待任何一个子进程一次。</font>

<font color="#4f6128">进程可以产生任意数量的子进程，可以以任何顺序等待它们，甚至可以在没有等待所有或部分子进程的情况下退出。你的设计应该考虑所有等待发生的方式。无论父进程是否等待它，以及子进程是在父进程之前还是之后退出，进程的所有资源，包括其</font> `struct thread` <font color="#4f6128">结构，都必须被释放。</font>
<font color="#4f6128">你必须确保Pintos在初始进程退出之前不会终止。提供的Pintos代码试图通过从</font>`main()`<font color="#4f6128">（在</font>`threads/init.c`<font color="#4f6128">中）调用</font>`process_wait()`<font color="#4f6128">（在</font>`userprog/process.c`<font color="#4f6128">中）来实现这一点。我们建议你根据函数顶部的注释实现</font>`process_wait()`<font color="#4f6128">，然后根据</font>`process_wait()`<font color="#4f6128">实现等待系统调用。</font>

>可以整理为以下要求：

- `wait(pid)`只能等待直接子进程，不能等待间接子进程，并且一个进程最多只能等待任何一个子进程一次
- pid没有调用`wait()`但被内核终止/pid不是直接调用进程的子进程的pid/进程已经调用过`wait()`但仍等待→返回-1
- 即使子进程已退出，父进程等待该子进程时也可以获取其退出状态

首先按要求实现`process_wait()`的完整函数：
``` c
/** Waits for thread TID to die and returns its exit status.  If
   it was terminated by the kernel (i.e. killed due to an
   exception), returns -1.  If TID is invalid or if it was not a
   child of the calling process, or if process_wait() has already
   been successfully called for the given TID, returns -1
   immediately, without waiting.

   This function will be implemented in problem 2-2.  For now, it
   does nothing. */
int
process_wait (tid_t child_tid UNUSED) 
{
  return -1;
}
```
函数顶部注释的意思是：等待线程TID结束并返回其退出状态。如果它是由内核终止的（即由于异常被杀死），则返回-1。如果TID无效，或者它不是调用进程的子进程，或者如果对于给定的TID已经成功调用了process_wait()，则立即返回-1，不等待。
也就是说，当内核终止进程/子进程tid无效or不是调用进程的子进程→`return -1`
else在进程结束后return退出状态。
为了实现这些功能，首先增加子进程结构体：
``` c
struct child
{
    tid_t tid;//子进程id
    bool hasBeenWaited;//子进程是否被等待过
    struct list_elem child_elem;//其父进程的子进程列表的节点
    struct semaphore sema;//子进程信号量，用于父进程等待子进程退出
    int store_exit;//子进程退出状态
};
```
并在`thread_create()`中初始化：
``` c
t->thread_child = malloc(sizeof(struct child));//分配内存
t->thread_child->tid = tid;//继承其父进程的tid 
sema_init (&t->thread_child->sema, 0);//初始化信号量 
list_push_back (&thread_current()->childs, &t->thread_child->child_elem); //存入子进程列表
t->thread_child->store_exit = UINT32_MAX;//初始化退出状态为最大值
t->thread_child->hasBeenWaited= false;//未等待
```
然后在`thread`中新增以下字段：
``` c
struct list childs;//当前进程的所有直接子进程列表
struct child * thread_child;//当前进程作为其父进程的子进程的节点
bool success;//是否成功同步
struct thread* parent;//其父进程  
```
并在`init_thread()`初始化：
``` c
if (t==initial_thread)//如果是初始进程-没有父进程
{
	t->parent=NULL;
}
else//如果不是初始进程-则记录其父进程
{
	t->parent = thread_current ();
}
list_init (&t->childs);//初始化子进程列表
t->success = true;//可成功同步
t->st_exit = UINT32_MAX;//初始化退出状态为最大值
```
按要求实现`process_wait()`：
``` c
int
process_wait (tid_t child_tid UNUSED)
{
  struct list *l = &thread_current()->childs;//当前进程的直接子进程列表
  struct list_elem *child_elem_ptr;//遍历列表
  child_elem_ptr = list_begin (l);//指向第1个元素
  struct child *child_ptr = NULL;//指向当前遍历到的子进程
  while (child_elem_ptr != list_end (l))//遍历
  {
    child_ptr = list_entry (child_elem_ptr, struct child, child_elem);//获取子进程
    if (child_ptr->tid == child_tid)//如果该子进程tid匹配
    {
      if (!child_ptr->hasBeenWaited)//如果该子进程没有被等待过
      {
        child_ptr->hasBeenWaited= true;//设置为已被等待
        sema_down (&child_ptr->sema);//阻塞父进程，等待子进程退出
        break;
      } 
      else//如果该子进程已经被等待过-返回-1
      {
        return -1;
      }
    }
    child_elem_ptr = list_next (child_elem_ptr);//如果该子进程tid不匹配-指向下一个节点
  }
  if (child_elem_ptr == list_end (l))//遍历结束仍未找到→tid无效，返回-1
  {
    return -1;
  }
  list_remove (child_elem_ptr);//移除该子进程，避免二次等待
  return child_ptr->store_exit;//返回子进程退出状态
}
```
然后在`thread_exit()`打印终止信息后添加`signal()`：
``` c
/** Deschedules the current thread and destroys it.  Never
   returns to the caller. */
void
thread_exit (void) 
{
  ASSERT (!intr_context ());

#ifdef USERPROG
  process_exit ();
#endif

  /* Remove thread from all threads list, set our status to dying,
     and schedule another process.  That process will destroy us
     when it calls thread_schedule_tail(). */
  intr_disable ();
  
  printf ("%s:exit(%d)\n",thread_name(),thread_current()->st_exit);//终止信息

  thread_current ()->thread_child->store_exit = thread_current()->st_exit;//将当前进程的退出状态保存到其父进程的子进程列表中
  sema_up (&thread_current()->thread_child->sema);//唤醒父进程
  
  list_remove (&thread_current()->allelem);
  thread_current ()->status = THREAD_DYING;
  schedule ();
  NOT_REACHED ();
}
```
最后在`sys_wait()`中调用`process_wait()`：
``` c
void 
sys_wait (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数pid的地址是否合法
  *user_ptr++;//指向pid
  f->eax = process_wait(*user_ptr);//将子进程退出状态写入eax
}
```

##### (5)`System Call: bool create (const char *file, unsigned initial_size)`
<font color="#4f6128">创建一个名为</font>`file`<font color="#4f6128">的新文件，初始大小为</font>`initial_size`<font color="#4f6128">字节。如果成功，返回true，否则返回false。创建新文件并不打开它：打开新文件是一个单独的操作，需要调用</font>`open`<font color="#4f6128">系统调用。</font>
create属于关于文件的系统调用，对这一部分，文档给出的要求是实现同步系统调用，即将文件系统代码视为临界区。
因此首先建立锁机制，确保任何时候只有一个线程能访问文件：
##### Ⅰ在`thread.h`中新增锁
``` c
static struct lock lock_f;
```
并在`thread_init()`中初始化：
``` c
lock_init(&lock_f);
```
##### Ⅱ调用`lock_acquire()`和`lock_release()`来获取、释放锁：
``` c
void 
acquire_lock_f ()
{
  lock_acquire(&lock_f);
}

void 
release_lock_f ()
{
  lock_release(&lock_f);
}
```
`filesys/filesys.c`中提供了`filesys_create()`函数，用于创建一个文件名为`name`、大小为`initial_size`的文件：
``` c
/* Creates a file named NAME with the given INITIAL_SIZE.
   Returns true if successful, false otherwise.
   Fails if a file named NAME already exists,
   or if internal memory allocation fails. */
bool
filesys_create (const char *name, off_t initial_size) 
{
  block_sector_t inode_sector = 0;
  struct dir *dir = dir_open_root ();
  bool success = (dir != NULL
                  && free_map_allocate (1, &inode_sector)
                  && inode_create (inode_sector, initial_size)
                  && dir_add (dir, name, inode_sector));
  if (!success && inode_sector != 0) 
    free_map_release (inode_sector, 1);
  dir_close (dir);

  return success;
}
```
调用以上函数来实现`sys_create()`：
由于文档在之前的部分要求：检测无效或未映射的用户栈地址，并在访问用户提供的数据时防止内核页面错误。因此需要在读取系统调用参数之前验证用户栈是否包含足够可访问的内存区域。
并且还需要确保从用户空间传递的任何指针都不会引用内核空间或未映射的内存，因此需要检查从用户栈中获取的用户提供的指针。
``` c
void 
sys_create(struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  /*检查指针是否合法*/
  check_ptr2 (user_ptr + 5);//检查用户栈是否包含足够可访问的内存区域
  check_ptr2 (*(user_ptr + 4));//检查从用户栈中获取的用户提供的指针
  
  *user_ptr++;//指向第一个参数文件名
  acquire_lock_f ();//获取锁
  f->eax = filesys_create ((const char *)*user_ptr, *(user_ptr+1));//创建文件
  release_lock_f ();//释放锁
}
```

##### (6)`System Call: bool remove (const char *file)`
<font color="#4f6128">删除名为file的文件。如果成功则返回true，否则返回false。文件可以删除，无论它是否打开，删除一个打开的文件不会关闭它。</font>
`filesys/filesys.c`中提供了`filesys_remove()`函数，用于删除名为`NAME`的文件。如果成功，返回true；如果失败，返回false。如果不存在名为`NAME`的文件，或者内部内存分配失败，则失败。
``` c
/* Deletes the file named NAME.
   Returns true if successful, false on failure.
   Fails if no file named NAME exists,
   or if an internal memory allocation fails. */
bool
filesys_remove (const char *name) 
{
  struct dir *dir = dir_open_root ();
  bool success = dir != NULL && dir_remove (dir, name);
  dir_close (dir); 

  return success;
}
```
调用这个函数和之前实现的`lock_acquire()`和`lock_release()`来实现`sys_remove()`：
``` c
void 
sys_remove(struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数地址是否合法
  check_ptr2 (*(user_ptr + 1));//检查指针是否合法
  *user_ptr++;//指向第一个参数文件名
  acquire_lock_f ();//获取锁
  f->eax = filesys_remove ((const char *)*user_ptr);//删除文件
  release_lock_f ();//释放锁
}
```
##### (7)`System Call: int open (const char *file)`
<font color="#4f6128">打开名为file的文件。</font>

- <font color="#4f6128">如果文件成功打开，返回一个非负整数句柄“文件描述符”fd</font>
- <font color="#4f6128">如果文件无法打开，返回-1</font>

<font color="#4f6128">编号为0和1的文件描述符保留用于控制台，打开系统调用永远不会返回这些文件描述符，它们仅作为系统调用参数在下面明确描述的情况下才有效。</font>

- `fd 0 (STDIN_FILENO)` <font color="#4f6128">是标准输入</font>
- `fd 1 (STDOUT_FILENO)` <font color="#4f6128">是标准输出</font>

<font color="#4f6128">每个进程都有一个独立的文件描述符集。文件描述符不会被子进程继承。</font>
<font color="#4f6128">当单个文件被多次打开时，无论是单个进程还是不同进程，每次打开都会返回一个新的文件描述符。</font>
<font color="#4f6128">对于单个文件的不同文件描述符，在单独的</font>`close`<font color="#4f6128">调用中会独立关闭，它们不共享文件位置。</font>
首先添加`thread_file`结构体，表示线程打开的一个文件：
``` c
struct thread_file
  {
    int fd;//文件描述符
    struct file* file;//指向真正的文件
    struct list_elem file_elem;//用于存入打开文件列表
  };
```
在`thread`结构体中新增以下条目：
``` c
struct list files;//线程的打开文件列表，即thread_file的集合
int max_file_fd;//文件描述符fd
```
用来管理线程打开的文件，实现同步操作
并在`init_thread()`中初始化：
``` c
list_init (&t->files);
t->max_file_fd=2;//非负&&不能取0或1，所以从2开始
```
`filesys/filesys.c`中提供了`filesys_open()`函数，使用给定的NAME打开文件。如果成功，则返回新的文件；否则返回空指针。如果不存在名为NAME的文件，或者内部内存分配失败，则失败。
``` c
/** Opens the file with the given NAME.
   Returns the new file if successful or a null pointer
   otherwise.
   Fails if no file named NAME exists,
   or if an internal memory allocation fails. */
struct file *
filesys_open (const char *name)
{
  struct dir *dir = dir_open_root ();
  struct inode *inode = NULL;

  if (dir != NULL)
    dir_lookup (dir, name, &inode);
  dir_close (dir);

  return file_open (inode);
}
```
调用以上函数来实现`sys_open()`：
``` c
void 
sys_open (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数地址是否合法
  check_ptr2 (*(user_ptr + 1));//检查指针是否合法
  *user_ptr++;//指向第一个参数文件名
  acquire_lock_f ();//获取锁
  struct file * file_opened = filesys_open((const char *)*user_ptr);//打开文件
  release_lock_f ();//释放锁
  struct thread * t = thread_current();//指向当前线程
  if (file_opened)//如果成功打开文件
  {
    struct thread_file *thread_file_temp = malloc(sizeof(struct thread_file));//分配内存
    thread_file_temp->fd = t->max_file_fd++;//分配fd并自增，为下次open准备
    thread_file_temp->file = file_opened;//指向真正的文件
    list_push_back (&t->files, &thread_file_temp->file_elem);//将该节点加入当前线程打开文件列表
    f->eax = thread_file_temp->fd;//保存系统调用返回值返回值
  } 
  else//如果打开失败-返回-1
  {
    f->eax = -1;
  }
}
```
##### (8)`System Call: int filesize (int fd)`
<font color="#4f6128">返回以 fd 打开文件的字节大小。</font>
`filesys/file.c`提供了`file_length()`函数，用于以字节为单位返回文件的大小：
``` c
/** Returns the size of FILE in bytes. */
off_t
file_length (struct file *file) 
{
  ASSERT (file != NULL);
  return inode_length (file->inode);
}
```
该函数传入的参数是指向真正的文件的指针，但是文档要求传入fd，因此需要设计一个函数来通过fd找`*file`。在之前新增的的部分中，可以通过`fd`找到对应的`thread_file`节点，再找到对应的真正文件`file*`，最后找到对应的`length`
因此添加`find_file_id()`函数，通过fd从打开文件列表中找到对应的节点：
``` c
struct thread_file * 
find_file_id (int file_id)
{
  struct list_elem *e;//遍历
  struct thread_file * thread_file_temp = NULL;//保存对应的thread_file节点
  struct list *files = &thread_current ()->files;//获取当前线程打开文件列表
  for (e = list_begin (files); e != list_end (files); e = list_next (e))//遍历寻找
  {
    thread_file_temp = list_entry (e, struct thread_file, file_elem);//获取file_elem对应的thread_file
    if (file_id == thread_file_temp->fd)//如果fd匹配-返回
      return thread_file_temp;
  }
  return false;//未找到匹配节点
}
```
调用以上函数实现`sys_filesize()`：
``` c
void 
sys_filesize (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数地址是否合法
  *user_ptr++;//指向第一个参数fd
  struct thread_file * thread_file_temp = find_file_id (*user_ptr);//根据fd查找对应的thread_file
  if (thread_file_temp)//如果找到
  {
    acquire_lock_f ();//获取锁
    f->eax = file_length (thread_file_temp->file);//保存文件大小
    release_lock_f ();//释放锁
  } 
  else//如果没找到-返回-1
  {
    f->eax = -1;
  }
}
```
##### (9)`System Call: int read (int fd, void *buffer, unsigned size)`
<font color="#4f6128">从文件 fd 中读取 size 个字节到缓冲区 buffer。</font>

- <font color="#4f6128">成功读取-返回实际读取的字节数</font>
- <font color="#4f6128">读取到文件末尾-返回0 </font>
- <font color="#4f6128">文件无法读取-返回-1</font>
- <font color="#4f6128">fd=0-使用</font>`input_getc()`<font color="#4f6128">从键盘读取</font>

`devices/input.c`<font color="#4f6128">提供了</font>`input_getc()`<font color="#4f6128">函数，用于从输入缓冲区中获取一个字符。如果缓冲区为空，则阻塞等待按键：</font>
``` c
/** Retrieves a key from the input buffer.
   If the buffer is empty, waits for a key to be pressed. */
uint8_t
input_getc (void) 
{
  enum intr_level old_level;
  uint8_t key;

  old_level = intr_disable ();
  key = intq_getc (&buffer);
  serial_notify ();
  intr_set_level (old_level);
  
  return key;
}
```
`read`需要向缓冲区写入数据，而在写入之前需要检查写入目标地址是否合法，因此添加`is_valid_pointer()`函数来检查地址：
``` c
bool 
is_valid_pointer (void* esp,uint8_t argc){
  for (uint8_t i = 0; i < argc; ++i)//检查一段地址范围
  {
    if((!is_user_vaddr (esp)) || 
      (pagedir_get_page (thread_current()->pagedir, esp)==NULL))//如果地址不＜PHYS_BASE或用户虚拟地址没有映射到物理页-非法地址
    {
      return false;
    }
  }
  return true;//反之则为合法地址
}
```
`filesys/file.c`提供了`file_read()`函数，用于从FILE中读取SIZE个字节到BUFFER中，从文件的当前位置开始。返回实际读取的字节数，如果到达文件末尾，则可能小于SIZE。将FILE的位置向前移动读取的字节数。
``` c
/* Reads SIZE bytes from FILE into BUFFER,
   starting at the file's current position.
   Returns the number of bytes actually read,
   which may be less than SIZE if end of file is reached.
   Advances FILE's position by the number of bytes read. */
off_t
file_read (struct file *file, void *buffer, off_t size) 
{
  off_t bytes_read = inode_read_at (file->inode, buffer, size, file->pos);
  file->pos += bytes_read;
  return bytes_read;
}
```
调用以上函数实现`sys_read()`：
``` c
void 
sys_read (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  *user_ptr++;//指向fd
  int fd = *user_ptr;//用户传入的文件描述符
  uint8_t * buffer = (uint8_t*)*(user_ptr+1);//用户栈上存储的buffer指针值
  off_t size = *(user_ptr+2);//要读取的字节数
  if (!is_valid_pointer (buffer, 1) || !is_valid_pointer (buffer + size,1))//如果buffer起始地址不合法或末端地址不合法-退出并返回-1
  {
    exit_special ();
  }
  if (fd == 0)//fd=0-调用input_getc()从键盘读取
  {
    for (int i = 0; i < size; i++)//读取size个字节
    {
	    buffer[i] = input_getc();
    }
    f->eax = size;//返回读取字节数
  }
  else//fd!=0-读取文件
  {
    struct thread_file * thread_file_temp = find_file_id (*user_ptr);//获取fd对应的thread_file
    if (thread_file_temp)//如果可以读取
    {
      acquire_lock_f ();//获取锁
      f->eax = file_read (thread_file_temp->file, buffer, size);//读文件
      release_lock_f ();//释放锁
    } 
    else//如果不能读取-返回-1
    {
      f->eax = -1;
    }
  }
}
```
##### (10)`System Call: int write (int fd, const void *buffer, unsigned size)`
<font color="#4f6128">将 buffer 中的 size 个字节写入已打开的文件 fd。</font>

- <font color="#4f6128">返回实际写入的字节数，如果有些字节无法写入，则返回的字节数可能小于 size</font>
- <font color="#4f6128">尽可能多地写入字节到文件末尾，并返回实际写入的字节数</font>
- <font color="#4f6128">如果完全无法写入字节，则返回0</font>

- <font color="#4f6128">fd=1-写入控制台。调用</font>`putbuf()`<font color="#4f6128">写入整个缓冲区，并尽量一次写完</font>
- <font color="#4f6128">fd≥2-找到对应的</font>`file*`<font color="#4f6128">，调用</font>`file_write()`<font color="#4f6128">，返回其返回值</font>
- <font color="#4f6128">fd非法-返回0</font>

`filesys/file.c`提供了`file_write()`函数，用于从文件的当前位置开始将BUFFER中的SIZE个字节写入FILE。返回实际写入的字节数，如果到达文件末尾，则可能小于SIZE。将FILE的位置向前移动读取的字节数。
``` c
/* Writes SIZE bytes from BUFFER into FILE,
   starting at the file's current position.
   Returns the number of bytes actually written,
   which may be less than SIZE if end of file is reached.
   (Normally we'd grow the file in that case, but file growth is
   not yet implemented.)
   Advances FILE's position by the number of bytes read. */
off_t
file_write (struct file *file, const void *buffer, off_t size) 
{
  off_t bytes_written = inode_write_at (file->inode, buffer, size, file->pos);
  file->pos += bytes_written;
  return bytes_written;
}
```
由于文档在之前的部分要求：检测无效或未映射的用户栈地址，并在访问用户提供的数据时防止内核页面错误。因此需要在读取系统调用参数之前验证用户栈是否包含足够可访问的内存区域。
并且还需要确保从用户空间传递的任何指针都不会引用内核空间或未映射的内存，因此需要检查从用户栈中获取的用户提供的指针。
调用以上函数实现`sys_write()`：
``` c
void 
sys_write (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;
  check_ptr2 (user_ptr + 7);//检查用户栈是否包含足够可访问的内存区域
  check_ptr2 (*(user_ptr + 6));//检查从用户栈中获取的用户提供的指针
  *user_ptr++;//指向fd
  int fd = *user_ptr;//获取fd
  const char * buffer = (const char *)*(user_ptr+1);//获取buffer指针
  off_t size = *(user_ptr+2);//获取要写入的字节数
  if (fd == 1) //fd=1→写入控制台
  {
    putbuf(buffer,size);//写入
    f->eax = size;//返回实际写入字节数
  }
  else//fd!=1
  {
    struct thread_file * thread_file_temp = find_file_id (*user_ptr);//找到fd对应的thread_file
    if (thread_file_temp)//如果fd合法-写入
    {
      acquire_lock_f ();//获取锁
      f->eax = file_write (thread_file_temp->file, buffer, size);//写入
      release_lock_f ();//释放锁
    } 
    else//如果fd不合法-无法写入-返回0
    {
      f->eax = 0;
    }
  }
}
```
##### (11)`System Call: void seek (int fd, unsigned position)`
<font color="#4f6128">将打开的文件fd中下一个要读取或写入的位置更改为position，position=0是文件的开始。</font>
`filesys/file.c`提供了`file_seek()`函数，用于将文件位置设置为从文件开头偏移 NEW_POS 字节的位置。
``` c
/* Sets the current position in FILE to NEW_POS bytes from the
   start of the file. */
void
file_seek (struct file *file, off_t new_pos)
{
  ASSERT (file != NULL);
  ASSERT (new_pos >= 0);
  file->pos = new_pos;
}
```
由于文档在之前的部分要求：检测无效或未映射的用户栈地址，并在访问用户提供的数据时防止内核页面错误。因此需要在读取系统调用参数之前验证用户栈是否包含足够可访问的内存区域。
调用`file_seek()`来实现`sys_seek()`：
``` c
void 
sys_seek(struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 5);//检查用户栈是否包含足够可访问的内存区域
  *user_ptr++;//指向fd
  struct thread_file *file_temp = find_file_id (*user_ptr);//找到fd对应的thread_file
  if (file_temp)//如果fd合法
  {
    acquire_lock_f ();//获取锁
    file_seek (file_temp->file, *(user_ptr+1));//修改位置
    release_lock_f ();//释放锁
  }
}
```
##### (12)`System Call: unsigned tell (int fd)`
<font color="#4f6128">返回在打开的文件 fd 中下一个要读取或写入的字节的位置，以从文件开头开始计算的字节数表示。</font>
`filesys/file.c`提供了`file_tell()`函数，用于返回FILE的当前位置，以字节偏移量表示，从文件开始处计算。
``` c
/* Returns the current position in FILE as a byte offset from the
   start of the file. */
off_t
file_tell (struct file *file) 
{
  ASSERT (file != NULL);
  return file->pos;
}
```
调用`file_tell()`来实现`sys_tell()`：
``` c
void 
sys_tell (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数地址是否合法
  *user_ptr++;//指向fd
  struct thread_file *thread_file_temp = find_file_id (*user_ptr);//找到fd对应的thread_file
  if (thread_file_temp)//如果fd合法
  {
    acquire_lock_f ();//获取锁
    f->eax = file_tell (thread_file_temp->file);//获取当前位置
    release_lock_f ();//释放锁
  }
  else//如果fd不合法-返回-1
  {
    f->eax = -1;
  }
}
```
##### (13)`System Call: void close (int fd)`
<font color="#4f6128">关闭文件描述符 fd。退出或终止进程会隐式关闭其所有打开的文件描述符，就像为每一个调用此函数一样。</font>
`filesys/file.c`提供了`file_close()`函数，用于关闭文件
``` c
/* Closes FILE. */
void
file_close (struct file *file) 
{
  if (file != NULL)
    {
      file_allow_write (file);
      inode_close (file->inode);
      free (file); 
    }
}
```
调用`file_close()`来实现`sys_close()`：
``` c
void 
sys_close (struct intr_frame* f)
{
  uint32_t *user_ptr = f->esp;//获取栈指针
  check_ptr2 (user_ptr + 1);//检查第一个参数地址是否合法
  *user_ptr++;//指向fd
  struct thread_file * opened_file = find_file_id (*user_ptr);//找到fd对应的thread_file
  if (opened_file)//如果fd合法
  {
    acquire_lock_f ();//获取锁
    file_close (opened_file->file);//关闭文件
    release_lock_f ();//释放锁
    list_remove (&opened_file->file_elem);//删除该节点
    free (opened_file);//释放内存
  }
}
```
而线程在调用`thread_exit()`终止时需要释放所有文件，因此也调用`file_close()`来遍历打开文件列表并释放文件：
``` c
/** Deschedules the current thread and destroys it.  Never
   returns to the caller. */
void
thread_exit (void) 
{
  ASSERT (!intr_context ());

#ifdef USERPROG
  process_exit ();
#endif

  /* Remove thread from all threads list, set our status to dying,
     and schedule another process.  That process will destroy us
     when it calls thread_schedule_tail(). */
  intr_disable ();
  
  printf ("%s:exit(%d)\n",thread_name(),thread_current()->st_exit);//终止信息

  thread_current ()->thread_child->store_exit = thread_current()->st_exit;//将当前进程的退出状态保存到其父进程的子进程列表中
  sema_up (&thread_current()->thread_child->sema);//唤醒父进程
  
  /*释放文件*/
  struct list_elem *e;//用于遍历
  struct list *files = &thread_current()->files;//当前线程的打开文件列表
  while(!list_empty (files))//遍历释放
  {
    e = list_pop_front (files);//获取并取出第一个打开文件
    struct thread_file *f = list_entry (e, struct thread_file, file_elem);//获取file_elem对应的thread_file
    acquire_lock_f ();//获取锁
    file_close (f->file);//关闭文件
    release_lock_f ();//释放锁
    /*Free the resource the file obtain*/
    free (f);//释放节点内存
  }
  
  list_remove (&thread_current()->allelem);
  thread_current ()->status = THREAD_DYING;
  schedule ();
  NOT_REACHED ();
}
```
#### 初始化
最后定义函数指针数组，用于保存以上系统调用函数的入口地址：
``` c
static void (*syscalls[max_syscall])(struct intr_frame *);
```
在`syscall_init()`中初始化对应的函数地址：
``` c
void
syscall_init (void) 
{
  intr_register_int (0x30, 3, INTR_ON, syscall_handler, "syscall");
 
  syscalls[SYS_EXEC] = &sys_exec;
  syscalls[SYS_HALT] = &sys_halt;
  syscalls[SYS_EXIT] = &sys_exit;
  syscalls[SYS_WAIT] = &sys_wait;
  syscalls[SYS_CREATE] = &sys_create;
  syscalls[SYS_REMOVE] = &sys_remove;
  syscalls[SYS_OPEN] = &sys_open;
  syscalls[SYS_WRITE] = &sys_write;
  syscalls[SYS_SEEK] = &sys_seek;
  syscalls[SYS_TELL] = &sys_tell;
  syscalls[SYS_CLOSE] =&sys_close;
  syscalls[SYS_READ] = &sys_read;
  syscalls[SYS_FILESIZE] = &sys_filesize;
}
```
触发中断时，`syscall_handler()`根据系统调用号自动选择对应的系统调用函数：
``` c
static void
syscall_handler (struct intr_frame *f UNUSED) 
{
  int * p = f->esp;//获取栈指针
  check_ptr2 (p + 1);//检验第一个参数地址合法
  int type = * (int *)f->esp;//读取系统调用号
  if(type <= 0 || type >= max_syscall)//如果系统调用号不在合法范围-退出并返回-1
  {
    exit_special ();
  }
  syscalls[type](f);//如果系统调用号合法-选择对应的系统调用函数进行处理
}
```
### 五、拒绝写入可执行文件
<font color="#4f6128">为正在作为可执行文件使用的文件添加代码以拒绝写入。您可以使用</font>`file_deny_write()`<font color="#4f6128">来阻止对打开的文件进行写入。对文件调用</font>`file_allow_write()`<font color="#4f6128">将重新启用它们（除非该文件被其他打开者拒绝写入）。关闭文件也会重新启用写入。因此，要拒绝向进程的可执行文件写入，您必须在进程仍在运行时保持其打开状态。</font>

文档中在系统部分给出了“<font color="#4f6128">别忘了</font>`process_execute()`<font color="#4f6128">也会访问文件</font>”的提醒
`process.c`中的`load()`函数会执行`filesys/file.c`中提供的文件系统操作，在这些系统操作中会读文件；而多个进程可能会并发执行系统调用，但原`load()`不支持临界区：
``` c
/** Loads an ELF executable from FILE_NAME into the current thread.
   Stores the executable's entry point into *EIP
   and its initial stack pointer into *ESP.
   Returns true if successful, false otherwise. */
bool
load (const char *file_name, void (**eip) (void), void **esp) 
{
  struct thread *t = thread_current ();
  struct Elf32_Ehdr ehdr;
  struct file *file = NULL;
  off_t file_ofs;
  bool success = false;
  int i;

  /* Allocate and activate page directory. */
  t->pagedir = pagedir_create ();
  if (t->pagedir == NULL) 
    goto done;
  process_activate ();

  /* Open executable file. */
  file = filesys_open (file_name);
  if (file == NULL) 
    {
      printf ("load: %s: open failed\n", file_name);
      goto done; 
    }

  /* Read and verify executable header. */
  if (file_read (file, &ehdr, sizeof ehdr) != sizeof ehdr
      || memcmp (ehdr.e_ident, "\177ELF\1\1\1", 7)
      || ehdr.e_type != 2
      || ehdr.e_machine != 3
      || ehdr.e_version != 1
      || ehdr.e_phentsize != sizeof (struct Elf32_Phdr)
      || ehdr.e_phnum > 1024) 
    {
      printf ("load: %s: error loading executable\n", file_name);
      goto done; 
    }

  /* Read program headers. */
  file_ofs = ehdr.e_phoff;
  for (i = 0; i < ehdr.e_phnum; i++) 
    {
      struct Elf32_Phdr phdr;

      if (file_ofs < 0 || file_ofs > file_length (file))
        goto done;
      file_seek (file, file_ofs);

      if (file_read (file, &phdr, sizeof phdr) != sizeof phdr)
        goto done;
      file_ofs += sizeof phdr;
      switch (phdr.p_type) 
        {
        case PT_NULL:
        case PT_NOTE:
        case PT_PHDR:
        case PT_STACK:
        default:
          /* Ignore this segment. */
          break;
        case PT_DYNAMIC:
        case PT_INTERP:
        case PT_SHLIB:
          goto done;
        case PT_LOAD:
          if (validate_segment (&phdr, file)) 
            {
              bool writable = (phdr.p_flags & PF_W) != 0;
              uint32_t file_page = phdr.p_offset & ~PGMASK;
              uint32_t mem_page = phdr.p_vaddr & ~PGMASK;
              uint32_t page_offset = phdr.p_vaddr & PGMASK;
              uint32_t read_bytes, zero_bytes;
              if (phdr.p_filesz > 0)
                {
                  /* Normal segment.
                     Read initial part from disk and zero the rest. */
                  read_bytes = page_offset + phdr.p_filesz;
                  zero_bytes = (ROUND_UP (page_offset + phdr.p_memsz, PGSIZE)
                                - read_bytes);
                }
              else 
                {
                  /* Entirely zero.
                     Don't read anything from disk. */
                  read_bytes = 0;
                  zero_bytes = ROUND_UP (page_offset + phdr.p_memsz, PGSIZE);
                }
              if (!load_segment (file, file_page, (void *) mem_page,
                                 read_bytes, zero_bytes, writable))
                goto done;
            }
          else
            goto done;
          break;
        }
    }

  /* Set up stack. */
  if (!setup_stack (esp))
    goto done;

  /* Start address. */
  *eip = (void (*) (void)) ehdr.e_entry;

  success = true;

 done:
  /* We arrive here whether the load is successful or not. */
  file_close (file);
  return success;
}
```
在之前的部分已经实现了锁机制，并且`filesys/file.c`提供了`load()`函数，用于防止对 FILE 的底层 inode 执行写操作，直到调用 `file_allow_write()` 或 FILE 被关闭。
```c
/* Prevents write operations on FILE's underlying inode
   until file_allow_write() is called or FILE is closed. */
void
file_deny_write (struct file *file) 
{
  ASSERT (file != NULL);
  if (!file->deny_write) 
    {
      file->deny_write = true;
      inode_deny_write (file->inode);
    }
}
```
而进程运行期间需要保持ELF文件一直打开，防止写禁止失效或者其他进程修改ELF文件的内容，导致不同步。因此可以利用之前实现的打开文件列表，将指向ELF文件的指针存入`thread_file`节点并加入到打开文件列表中，这样在进程运行时，ELF就会保持在打开状态，确保写禁止生效，在进程终止后，`thread_exit()`会统一调用`file_close()`，使写禁止解除。

首先在打开ELF文件前后添加锁机制；然后为ELF文件创建`thread_file`节点并加入到打开文件列表中；最后调用`file_deny_write()`来确保同一时刻只有一个线程写入文件
``` c
/* Loads an ELF executable from FILE_NAME into the current thread.
   Stores the executable's entry point into *EIP
   and its initial stack pointer into *ESP.
   Returns true if successful, false otherwise. */
bool
load (const char *file_name, void (**eip) (void), void **esp) 
{
  struct thread *t = thread_current ();
  struct Elf32_Ehdr ehdr;
  struct file *file = NULL;
  off_t file_ofs;
  bool success = false;
  int i;

  /* Allocate and activate page directory. */
  t->pagedir = pagedir_create ();
  if (t->pagedir == NULL) 
    goto done;
  process_activate ();
  acquire_lock_f ();//获取锁
  /* Open executable file. */
  file = filesys_open (file_name);

  if (file == NULL) 
    {
      printf ("load: %s: open failed\n", file_name);
      goto done; 
    }
    
  struct thread_file *thread_file_temp = malloc(sizeof(struct thread_file));//为thread_file节点分配内存
  thread_file_temp->file = file;//将指向真正文件的指针存入节点
  list_push_back (&thread_current()->files, &thread_file_temp->file_elem);//将节点存入打开文件列表
  
  file_deny_write(file);//确保同一时刻只有一个线程写入文件
  
  /* Read and verify executable header. */
  if (file_read (file, &ehdr, sizeof ehdr) != sizeof ehdr
      || memcmp (ehdr.e_ident, "\177ELF\1\1\1", 7)
      || ehdr.e_type != 2
      || ehdr.e_machine != 3
      || ehdr.e_version != 1
      || ehdr.e_phentsize != sizeof (struct Elf32_Phdr)
      || ehdr.e_phnum > 1024) 
    {
      printf ("load: %s: error loading executable\n", file_name);
      goto done; 
    }

  /* Read program headers. */
  file_ofs = ehdr.e_phoff;
  for (i = 0; i < ehdr.e_phnum; i++) 
    {
      struct Elf32_Phdr phdr;

      if (file_ofs < 0 || file_ofs > file_length (file))
        goto done;
      file_seek (file, file_ofs);

      if (file_read (file, &phdr, sizeof phdr) != sizeof phdr)
        goto done;
      file_ofs += sizeof phdr;
      switch (phdr.p_type) 
        {
        case PT_NULL:
        case PT_NOTE:
        case PT_PHDR:
        case PT_STACK:
        default:
          /* Ignore this segment. */
          break;
        case PT_DYNAMIC:
        case PT_INTERP:
        case PT_SHLIB:
          goto done;
        case PT_LOAD:
          if (validate_segment (&phdr, file)) 
            {
              bool writable = (phdr.p_flags & PF_W) != 0;
              uint32_t file_page = phdr.p_offset & ~PGMASK;
              uint32_t mem_page = phdr.p_vaddr & ~PGMASK;
              uint32_t page_offset = phdr.p_vaddr & PGMASK;
              uint32_t read_bytes, zero_bytes;
              if (phdr.p_filesz > 0)
                {
                  /* Normal segment.
                     Read initial part from disk and zero the rest. */
                  read_bytes = page_offset + phdr.p_filesz;
                  zero_bytes = (ROUND_UP (page_offset + phdr.p_memsz, PGSIZE)
                                - read_bytes);
                }
              else 
                {
                  /* Entirely zero.
                     Don't read anything from disk. */
                  read_bytes = 0;
                  zero_bytes = ROUND_UP (page_offset + phdr.p_memsz, PGSIZE);
                }
              if (!load_segment (file, file_page, (void *) mem_page,
                                 read_bytes, zero_bytes, writable))
                goto done;
            }
          else
            goto done;
          break;
        }
    }

  /* Set up stack. */
  if (!setup_stack (esp))
    goto done;

  /* Start address. */
  *eip = (void (*) (void)) ehdr.e_entry;

  success = true;

 done:
  /* We arrive here whether the load is successful or not. */
  release_lock_f();//释放锁
  //由于需要让可执行文件在进程运行期间一直保持打开并 deny_write，因此删除原来的file_close(file)，改为在thread_exit()中统一关闭
  return success;
}
```
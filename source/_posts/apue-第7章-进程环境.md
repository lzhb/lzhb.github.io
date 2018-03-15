---
title: apue 第7章 进程环境
date: 2017-10-28 16:06:57
tags: unix,note
---

7. 3. 进程终止

      + 内核在调用main前先调用一个特殊的启动例程(汇编)，以C代码形式表示如下:

        `exit (main(argc,argv));`
      + ```c  
        #include <stdlib.h>
        void exit(int status);  //会调用fclose
        void _Exit(int status); //不会fclose, so no flush
        int atexit(void (*func)(void));//可调用多次；exit会逆序调用funcs
        ```

   4. 命令行参数

      + 通常使用getopt_long解析参数。

   >  fork后，exec()时，如何传入argv

   5. 环境表

      ![环境表](/images/apue_环境表.png)

   6. C程序的存储空间布局

![存储空间安排](/images/apue_存储空间安排.png)

7. 7. 共享库

`gcc -static hello.c`阻止gcc使用共享库，导致可执行文件使用静态库，文件长度变长。

7. 8. 存储空间分配

+ ```c 
  #include<stdlib.h>
  void *malloc(size_t size);//size:字节数//存储区的初始值不确定//堆 heap
  void *calloc(size_t nbj,size_t size);//每一位bit初始为0；
  void *realloc(void *ptr,size_t newsize);//重新分配大小

  void free(void *ptr);

  alloca() //在栈中分配内存 //advantage: no need to free //dis: no support in some systems
  ```

7. 9. 环境变量

+ ```c 
  #include<stdlib.h>
  char *getenv(const char *name);//eg:getenv("PATH"); 得到name=value中的value
  int putenv(char *str);//增加"name=value"环境变量
  int setenv(const char *name,const char *value int rewrite);//修改；rewrite=0,不更新
  int unsetenv(const char * name);//删除name的定义，即使不存在也不出错
  ```

7. 10. 函数setjmp和longjmp

+ 跨越函数的跳转功能，类似C中的goto语句

+ 一开始，跳之前，setjmp记住环境信息到jmp_buf  env,供longjmp使用

+ jmp的任务：改PC、SP，栈指针 ； goto 使用label

+ ```c
  #include<setjmp.h>
  int setjmp(jmp_buf env);//第一次返回0；longjmp后，返回longjmp的val
  void longjmp(jmp_buf env,int val);
  ```

+ `gcc -O testjmp.c`进行全部优化的编译。

+ 全局变量(main外面 static int )，静态变量(main里面)，易失变量(volatile修饰int)不受优化的影响，jmp后变量的值不变

+ autoval(main内，普通int)，regival(main内，register修饰int)的值在Jmp后回退。

7. 11. 函数getrlimit和setrlimit


---

## 8. 进程控制

![存储空间安排](/images/apue_task_model.png)

8. 2.  进程标识
      + | PID     | Process                     |
        | ------- | --------------------------- |
        | 0       | swapper(scheduler)          |
        | 1       | init(/sbin/init) 所有孤儿进程的父进程 |
        | 2       | pagedaemon(负责虚拟存储系统的分页操作)   |
        | 3，4···· | other processes             |
      + ​
      ```c
        #include<unistd.h>
        pid_t getpid(void);//返回：进程的进程id
        pid_t getppid(void);//返回：父进程的id
        
        uid_t getuid(void);//返回：进程的实际用户id
        uid_t geteuid(void);//返回：进程的有效用户id; 判断进程的权限，用effective id
        
        gid_t getgid(void);
        gid_t getegid(void);
      ```
   3. 函数fork
      - ```c
        #include<unistd.h>
        pid_t fork(void);//子进程返回0,父进程返回子进程的id(>0),若出错，返回-1；
        ```

      - child gets a **copy** of parent's data space,heap,and stack.

      - often, read-only text segement is shared.

      - however,fork() is followed by exec() , so it's a wate of space and time to **copy**. so we use **COW**(copy-on-write).

      - > `write(STDOUT_FILENO,buf,sizeof(buf)-1)` strlen计算不包括终止null字节的字符串长度，而sizeof会计算，所以要-1; 但sizeof的好处是：当buf已初始化，长度固定时，sizeof是编译时就计算了，而strlen需要一次函数调用。
        >
        > write不带缓冲，printf是标准IO，带缓冲。回忆5.12，当标准输出连到终端设备时，它是行缓冲的，否则它是全缓冲(比如，从定位到一个文件时)。

      - 文件共享

      - + fork后，父进程的所有打开文件描述符都被复制到子进程中。
        + 父子进程共享同一个文件偏移量。
        + ![存储空间安排](/images/apue_fork_file_sharing.png)


8. 4. 函数vfork

      + creates a new process only to 'exec' a new program

      - No copy of parent's address space(no need)
      - before exec , child runs in 'address space of parent'
      - **child runs first;**
      - parent waits until child '**exec**' or 'exit'


8. 5. child termination
      + normal: exit status
      + abnormal: kernel indicates reason
      + when child terminates before parent, child returns termination status
      + + child terminates -> kernel sends **SIGCHLD** signal to parent.
      + + default action of parent for SIGCHLD signal : ignore it.
      + when parent terminates before child, parent PID(of orphaned child) = 1 (init)

   6. 函数wait和waitpid

      + ```c
        #include<sys/wait.h>
        pid_t wait(int *statloc);//bloc wait for any one child to terminate
        pid_t waitpid(pid_t pid, int *statloc, int options);
        //statloc is place for storing termination status ; Null -> no need
        //if pid=-1; waitpid与wait等效
        //pid>0 
        //pid==0; waitpid等待 组ID和父相等的子进程
        //pid<-1; 等待 组ID等于pid的绝对值的任一子进程
        ```

      + 检查status(即statloc)的宏

        ![存储空间安排](/images/apue_status_macros.png)

      + zombie process: child terminates first && parent don't wait child

      + + init的子进程不会成为zombie；because init会及时wait；

      + A process doesn't wait for the child to complete and not want child become zombie.how to do this? answer: **fork twice**

      + + ![存储空间安排](/images/apue_fork_twice_avoid_zombie.png)
        + $a.out
        + second child,parent pid=1

   7. 函数waitid(unix specification 提供的更灵活的waitpid)

   8. 函数wait3和wait4(延袭自BSD，附加参数允许内核返回终止进程及其所有子进程使用的资源情况)

   9. 竞争条件

      + Multiple processes share some data
      + outcome depends on the order of their execution
      + After fork(), we cannot predict if the parent or the child runs first.
      + polling(轮询)方法，不好wastes CPU time
      + + parent call wait,waitpid,wait3,wait4 to wait for child
        + child : while(getppid()!=1) sleep(1);
      + use signals (10.6) or other IPC (15、16)methods

   10. 函数exec

     + ```c
       #include<unistd.h>
       int execl(const char *pathname, const char *arg0, .../* (char *)0 */);//list
       int execv(const char *pathname, char *const argv[]);//vector

       int execle(const char *pathname, const char *arg0, .../* (char *)0 , char *const envp[] */);//环境变量
       int execve(const char *pathname, char *const argv[],char *const envp[] );

       int execlp(const char *filename, const char *arg0, .../* (char *)0 */);
       int execvp(const char *filename, char *const argv[]);//filename has '/',it's a path ; or PATH找filename

       int fexecve(int fd, char *const argv[], char *const envp[]);//build path from /proc/self/fd alias

       return -1 on error , no return on success.
       ```

     + ![存储空间安排](/images/apue_exec_6_relationship.png)

   11. 更改用户ID和组ID

       + ```c
         #include<unistd.h>
         int setuid(uid_t uid);
         int setgid(gid_t gid);

         return: 0 if ok , -1 on error
         ```

       + if **Superuser:**  real,effective,save set-UID := uid

       + if **real or save set-UID == uid :**  effective := uid

         else error := EPERM ; return error;

       + only effective UID or GID is changed: (often , superuser use these funcs )

         ```c
         #include<unistd.h>
         int seteuid(uid_t uid);
         int setegid(gid_t gid);

         return: 0 if ok , -1 on error.
         ```

       > -rws.. 中设置s是设置用户ID位，作用：当执行此文件时，将进程的有效用户ID设置为文件所有者的用户ID(st_uid);
       >
       > eg:  passwd系统程序是一个**设置用户ID**程序。因此普通用户运行它时，拥有超级用户权限，能够修改/etc/passwd或/etc/shadow文件。

   12. 解释器文件

   13. 函数system

       + `#include<stdlib.h>  int system(const char *cmdstring);`

       + uses fork to create a child process that executes the shell command using execl:

         `execl("/bin/sh","sh","-c",command,(char*)0);`

         + shell的-c选项告诉shell程序取下一个命令行参数(command)作为命令输入
         + shell 也会fork再执行cmd. 两次fork，avoid zombie process.

       + 使用system而不直接fork和exec的优点：system进行了所需的各种出错处理和各种信号处理

       + ![apue_8_system](/images/apue_8_system.png)

## 10. 信号

10. 1. 引言

      信号时软中断。信号提供了一种异步处理事件的方法。早期Unix实现信号的机制不可靠，之后Berkeley和AT&T做的更改又不兼容。最后，POSIX.1对可靠信号例程进行了标准化。

   2. dd

   3. 函数signal

      ```c
      #include<signal.h>
      void (*signal(int signo, void (*func)(int)))(int);//改变信号处理的动作
      //简化
      typedef void Sigfunc(int);//定义了一个 有int参数，无返回值的函数类型
      Sigfunc *signal(int,Sigfunc *);//signal是一个函数，返回一个函数指针
      return:previous disposition of signal if OK,SIG_ERR on error
      ```

      ```c
      #define SIG_ERR (void (*)())-1 //将常量强转为指向函数的指针
      #define SIG_DFL (void (*)())0
      #define SIG_IGN (void (*)())1
      //eg:
      static void sig_handler(int);
      if(signal(SIGUSR1,sig_handler)==SIG_ERR)
        err_sys("cant't catch SIGUSR1");
      ```

   4. 不可靠的信号

      + 早期不具备阻塞信号的能力。

   5. 中断的系统调用

   6. 可重入函数(Reetrant Functions)

      + 在信号处理程序中保证调用安全的函数，可重入的，异步信号安全的。
      + 不可重入函数的特征：使用静态数据结构；调用了malloc或free；标准I/O函数。
      + 通用的规则：信号处理函数中，调用可重入函数前应先保存errno，最后恢复errno。

  7. SIGCLD语义

  8. 可靠信号术语和语义(terminology and semantics)

      + three status of signal: Genertated,Pending,delivered.
      + Aprocess can **block** the delivery of a signal using **SIGNAL MASK**
      + signal mask can be changed by ***sigpromask()*** (see 10.12)
      + ***sigpending()*** function is used to determine blocked and pending signals.
      + More than one blocked signal? 
      + + (When 一种信号产生多次)Queued? No! Just once! 
        + (When 多种信号发给一个进程)SIGSEGV delivered first.(与进程状态相关优先)
      + **sigset_t**: data type to store signal mask
      + ![apue_8_system](/images/apue_10_signal.png)

  9. 函数kill和raise

      + ```c
        #include<signal.h>
        int kill(pid_t pid,int signo);
        int raise(int signo);//用处：自测；向进程自己发信号；= kill(getpid(),signo)
        ```

      + pid>0: sent to PID==pid

      + pid<0: sent to PGID==|pid|

      + pid==0: sent to all processes with PGID==PGID of sender(with perm)

      + pid==-1: broadcast signals 

  10. 函数alarm和pause

      + ```c
        #include<unistd.h>
        unsigned int alarm(unsigned int seconds);//定时产生SIGALRM；默认动作是终止进程
        return: 0 or seconds until previously set alarm
        int pause(void);//suspends a process until a signal is caught
        ```

  11. 信号集

       + 五个函数

  12. 函数sigprocmask

       + 用来查看或更改进程的信号屏蔽字

  13. 函数sigpending

       + returns the set of signals that are blocked and pending

  14. 函数sigaction

       + 可靠的修改信号处理的动作

       + ```c
         #include<signal.h>
         int sigaction(int signo, const struct sigaction *restrict act, struct sigaction *restrict oact);
         return: 0 if OK, -1 on error
         ```

       + restrict 是为了告诉编译器额外信息（两个指针不指向同一数据），从而生成更优化的机器码。

       + ```c
         struct sigaction{
           void (*sa_handler)(int);/*address of signal handler*/
           sigset_t sa_mask;//保证本action不被打断
           int sa_flags;//options
           //alternate handler
           void (*sa_sigaction)(int,siginfo_t *,void *);
         }
         ```

  15. 函数sigsestjmp和siglongjmp

       + 与setjmp、longjmp的区别：当sigsetjmp的savemask为非0时，env中会保存进程当前信号屏蔽字。

       + > 信号处理程序中，需要一种保护措施，防止信号发生时，jmpbuf尚未由sigsetjmp初始化好。eg: sigsetjmp调用完后，设置canjmp=1；在sig_usr1信号处理函数中，一开始检测,若canjmp!=1就return。
         >
         > static volatile sig_atomic_t canjmp;
         >
         > sig_atomic_t是由ISO C标准定义的变量类型。在写该变量时不会被中断。

  16. 函数sigsuspend

       + **如果希望对一个信号接触阻塞，然后pause以等待被阻塞的信号发生**；则会因为sigprocmask和pause前后不是原子操作，会发生丢失信号的情况。因此，使用sigsuspend.

       + ```c
         #include<signal.h>
         int sigsuspend(const sigset_t *sigmask);
         return:-1 with errno set to EINTR
         ```

       + 进程一直被挂起，直到捕捉到一个信号或出现了终止信号。

       + 如果捕捉到一个信号而且从该信号处理程序返回，则sigsuspend返回，信号屏蔽字设为sigsuspend之前的值。

       + suspend另一个应用：等待一个信号处理程序设置一个全局变量

       + 可以用信号实现父子进程之间的同步。

11. 线程

   1. ​
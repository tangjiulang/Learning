# LINUX

## fork

fork 创建一个新进程

```c
pid_t pidt = fork() 
/*
	pidt == 0 表示子进程
	pidt != 0 表示父进程，此时的pidt是子进程的pid
	多次fork得到的是一棵树
*/
```

## wait

wait函数的作用是父进程调用，等待子进程退出，回收子进程的资源

```c
pid_t pidt = wait(int* status);
/*
	pidt == -1 失败返回
	pidt != -1 返回子进程pid
*/
```

回收资源，避免僵尸进程

## waitpid

```c
pid_ t pidt = waitpid(pid_t pid, int *status, int options);
/*
	pidt = -1 等待任一个子进程。与wait等效。
	pidt > 0  等待其进程ID与pid相等的子进程。
	status 得到子进程返回的状态，子进程退出码
*/
```

waitpid返回的本质也就是：将该父进程从等待队列拿到运行队列中执行；
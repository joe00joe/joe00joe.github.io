本实验要求实现一个简单的unix shell程序。在shell上可以运行多个后台任务，但只能有一个前台任务。前台任务可被Ctrl+C终止或被Ctrl+D停止。当任务被停止，fg和bg命令可分别其共转化为前台任务和后台任务。

### 框架代码

csapp中提供了shell的一个不包含信号处理的框架代码，只需要把信号处理内容加上即可。框架的整体如下： 首先解析命令行输入，如果是内置命令（jobs、fg、bg、quit），则立即执行命令的逻辑；若不是内置命令（如`./myspin`)，则fork一个子进程，在子进程执行任务。

```c
void eval(char *cmdline) 
{

    sigset_t mask_all, mask_one, prev_one;

    sigfillset(&mask_all);
    sigemptyset(&mask_one);
    sigaddset(&mask_one, SIGCHLD);


    char *argv[MAXARGS]; /* Argument list execve() */
    char buf[MAXLINE];   /* Holds modified command line */
    int bg;              /* Should the job run in bg or fg? */
    pid_t pid;           /* Process id */
    
    strcpy(buf, cmdline);
    bg = parseline(buf, argv); 
    if (argv[0] == NULL)  
	return;   /* Ignore empty lines */

    if (!builtin_cmd(argv)) { 
        sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        if ((pid = fork()) == 0) {   /* Child runs user job */
            sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            setpgid(0, 0);  //must reset process group id before execve
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
            
        }
        
        /* Parent waits for foreground job to terminate */
        if (!bg) {
           //printf("bg: %d ",bg);
           sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */  
           addjob(jobs, pid, FG, cmdline);  /* Add the child to the job list */
           sigprocmask(SIG_SETMASK, &prev_one, NULL);  /* Unblock SIGCHLD */
           waitfg(pid);
        }
        else{
           sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */  
           addjob(jobs, pid, BG, cmdline);  /* Add the child to the job list */
           sigprocmask(SIG_SETMASK, &prev_one, NULL);  /* Unblock SIGCHLD */
           struct job_t * job = getjobpid(jobs, pid);
           printf("[%d] (%d) %s", job->jid, pid, cmdline);
        }
           
    }
    return;
    
}
```

### 信号处理

如果命令行输入的尾部不带'&'符号，那么执行的任务为前台任务，需要调用waitfg函数显式地等待任务终止信号（任务正常结束或Ctrl+C）或被停止信号（被Ctrl+D）。

```c 
/* 
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    

   sigset_t mask;
   sigemptyset(&mask);

   while(fgpid(jobs) == pid){
      //printf("pid: %d fgpid: %d\n",pid, fgpid(jobs));
      sigsuspend(&mask);
   }
   
}
```

在接收到任务终止信号信号时，`sigsuspend` 从处理程序`sigchld_handler`返回。因为`sigchld_handler`处理信号时将从任务列表删除前台任务，所以`sigsuspend`返回后可以退出while循环，结束前台任务，等待下一条命令。同理，用相同的机制处理停止信号（被Ctrl+D）

```c
void sigchld_handler(int sig) 
{
    int olderrno = errno;
    int status;
    sigset_t mask_all, prev_all;
    pid_t pid;
    
    struct job_t * job;

    sigfillset(&mask_all);
    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) { /* Reap a zombie child */
        sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        if(WIFEXITED(status)){    // terminated normally
            //printf("Job [%d] (%d)  terminated normally with exit status=%d\n",pid2jid(pid), pid, WEXITSTATUS(status));
            deletejob(jobs, pid);
        }
        else if (WIFSIGNALED(status)){   //Ctrl+C       
            printf ("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid), pid, WTERMSIG(status));
            deletejob(jobs, pid);
        }
        else if (WIFSTOPPED(status)){   //Ctrl+D        
            printf ("Job [%d] (%d) stoped by signal %d\n", pid2jid(pid), pid, WSTOPSIG(status));
            job = getjobpid(jobs, pid);
            job->state = ST;
        }
        sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    errno = olderrno;
    return;
}
```

最后，当任务被停止，可执行fg和bg命令发送SIGCONT信号将己停止的任务唤醒。

```c

    int cmdType = !strcmp(argv[0], "fg") ? FG : BG;
    struct job_t* job = getjobjid(jobs, jobId);
    pid_t pid = job->pid;
    kill(-pid, SIGCONT);   // send SIGCONT
    job->state = cmdType;
    if(cmdType == FG){
        waitfg(job->pid);
    }else{
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
    }
```


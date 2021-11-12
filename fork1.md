## 关于fork程序输出情况的讨论

考虑下面这段运行在rCore中的伪代码（来自chapter5简答题）


```rust
// fork_print.rs
fn main() -> i32 {
    fork() && fork() && fork() || fork() && fork() || fork() && fork();
    print!("A");
    return 0;
}
```

假设上面代码编译出可执行文件fork_print，在user_shell中输入命令fork_print并按一次回车会输出几个A呢？在实验之后发现，其结果可能是像下面这样暂时只输出1个A（具体输出几个A是不确定的，总之会少于理论答案）。

```
>> fork_print
AShell: Process 2 exited with code 0
>> 
```

现在具体分析一下这个现象。

shell中输入print并按下回车，会从user_shell程序中fork出一个进程（这个进程就是第一个运行print.c的进程），假设该进程pid为2，那么在user_shell中就会等待pid=2的进程退出。一旦退出，shell就会打印出"Shell: Process 2 exited with code 0"，然后输出">> "。参见下面改编自rCore的代码。

```rust
if pid != 0 { 
    // fork() != 0
   	waitpid(pid as usize, &mut exit_code);
    println!("Shell: Process {} exited with code {}", pid, exit_code);
}
print!(">> ");
```

pid=2的进程退出后，一定已经fork出几个子进程。pid=2退出后，它的子进程大概率没有全部运行完，因此这就解释了为什么此时输出A的数量远远少于预期。2退出后，这些子进程会成为孤儿进程，挂在initproc上。理论上来说，这些子进程应该会继续并行地执行，将A输出到各自的ConsoleBuffer中，最后在各自进程调用exit时将缓冲区中的内容flush到标准输出中。因此程序运行的结果应该像下面这样输出A，输出个数符合预期(改自Linux实际运行结果)。

```
>> print
AAAAAA>> AAA（更多A省略）
```

然而在rCore中，并不会自动输出更多的A。原因是user_shell在wait结束后会开始试图读取下一个标准输入中的字符，从标准输入读取字符的函数`getchar()`并不是立刻返回的。在user_shell调用系统调用sys_read后（标准输入），内部会调用函数`console_getchar()`，`console_getchar()`会引发一个系统调用中断，由RustSBI中的handler去处理，而这个handler在从标准输入中读到值或出错之前都不会返回。因此，user_shell在调用`getchar()`后就会被阻塞而无法返回用户态，sstatus.MIE已经被置为0，不会响应S态时钟中断，但是sip中时钟中断位被置为1，从而只要返回用户态，就会立刻再进入内核态中处理S态时钟中断。这样，只要键盘上按下任何一个键，调度器会调度fork_print程序fork出的子进程并输出A，直到下次调度到user_shell。就像下面这样，输入一个"."，接下来由于抢占调度fork_print子进程输出"A"，然后调度回user_shell输出"."，接着又在getchar时阻塞在内核中（严格来说是M态中）。

```
>> fork_print
AShell: Process 2 exited with code 0
>> AA.
```




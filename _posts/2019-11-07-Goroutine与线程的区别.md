---
layout: post
title: "Goroutines和线程的区别"
categories: 编程语言
---

- content
{:toc}





1. **Growable Stacks**

   > Each OS thread has a fixed-size block of memory (often as large as 2MB) for its stack, the work area where it saves the local variables of function calls that are in progress or temporarily suspended while another function is called. This fixed-size stack is simultaneously too much and too little. A 2MB stack would be a huge waste of memory for a little goroutine, such as one that merely waits for a WaitGroup then closes a channel. It’s not uncommon for a Go program to create hundreds of thousands of goroutines at one time, which would be impossi- ble with stacks this large. Yet despite their size, fixed-size stacks are not always big enough for the most complex and deeply recursive of functions. Changing the fixed size can improve space efficiency and allow more threads to be created, or it can enable more deeply recursive functions, but it cannot do both.

   > In contrast, a goroutine starts life with a small stack, typically 2KB. A goroutine’s stack, like the stack of an OS thread, holds the local variables of active and suspended function calls, but unlike an OS thread, a goroutine’s stack is not fixed; it grows and shrinks as needed. The size limit for a goroutine stack may be as much as 1GB, orders of magnitude larger than a typical fixed-size thread stack, though of course few goroutines use that much.

2. **Goroutine Scheduling**

   > OS threads are scheduled by the OS kernel. Every few milliseconds, a hardware timer inter- rupts the processor, which causes a kernel function called the scheduler to be invoked. This function suspends the currently executing thread and saves its registers in memory, looks over the list of threads and decides which one should run next, restores that thread’s registers from memory, then resumes the execution of that thread. Because OS threads are scheduled by the kernel, passing control from one thread to another requires a full context switch, that is, saving the state of one user thread to memory, restoring the state of another, and updating the scheduler’s data structures. This operation is slow, due to its poor locality and the number of memory accesses required, and has historically only gotten worse as the number of CPU cycles required to access memory has increased.

   > The Go runtime contains its own scheduler that uses a technique known as m:n scheduling, because it multiplexes (or schedules) m goroutines on n OS threads. The job of the Go scheduler is analogous to that of the kernel scheduler, but it is concerned only with the goroutines of a single Go program.

   > Unlike the operating system’s thread scheduler, the Go scheduler is not invoked periodically by a hardware timer, but implicitly by certain Go language constructs. For example, when a goroutine calls time.Sleep or blocks in a channel or mutex operation, the scheduler puts it to sleep and runs another goroutine until it is time to wake the first one up. Because it doesn’t need a switch to kernel context, rescheduling a goroutine is much cheaper than rescheduling a thread.

3. **GOMAXPROCS**

   > The Go scheduler uses a parameter called GOMAXPROCS to determine how many OS threads may be actively executing Go code simultaneously. Its default value is the number of CPUs on the machine, so on a machine with 8 CPUs, the scheduler will schedule Go code on up to 8 OS threads at once. (GOMAXPROCS is the n in m:n scheduling.) Goroutines that are sleeping or blocked in a communication do not need a thread at all. Goroutines that are blocked in I/O or other system calls or are calling non-Go functions, do need an OS thread, but GOMAXPROCS need not account for them.

   > You can explicitly control this parameter using the GOMAXPROCS environment variable or the runtime.GOMAXPROCS function. We can see the effect of GOMAXPROCS on this little program, which prints an endless stream of zeros and ones:

   ```go
    for {
        go fmt.Print(0)
        fmt.Print(1)
    }
    $ GOMAXPROCS=1 go run hacker-cliché.go 111111111111111111110000000000000000000011111...
    $ GOMAXPROCS=2 go run hacker-cliché.go 010101010101010101011001100101011010010100110...
   ```

   > In the first run, at most one goroutine was executed at a time. Initially, it was the main goroutine, which prints ones. After a period of time, the Go scheduler put it to sleep and woke up the goroutine that prints zeros, giving it a turn to run on the OS thread. In the second run, there were two OS threads available, so both goroutines ran simultaneously, printing digits at about the same rate. We must stress that many factors are involved in goroutine scheduling, and the runtime is constantly evolving, so your results may differ from the ones above.

4. **Goroutines Have No Identity**

   > In most operating systems and programming languages that support multithreading, the cur- rent thread has a distinct identity that can be easily obtained as an ordinary value, typically an integer or pointer. This makes it easy to build an abstraction called thread-local storage, which is essentially a global map keyed by thread identity, so that each thread can store and retrieve values independent of other threads.

   > Goroutines have no notion of identity that is accessible to the programmer. This is by design, since thread-local storage tends to be abused. For example, in a web server implemented in a language with thread-local storage, it’s common for many functions to find information about the HTTP request on whose behalf they are currently working by looking in that storage. However, just as with programs that rely excessively on global variables, this can lead to an unhealthy ‘‘action at a distance’’ in which the behavior of a function is not determined by its arguments alone, but by the identity of the thread in which it runs. Consequently, if the iden- tity of the thread should change—some worker threads are enlisted to help, say—the function misbehaves mysteriously.

   > Go encourages a simpler style of programming in which parameters that affect the behavior of a function are explicit. Not only does this make programs easier to read, but it lets us freely assign subtasks of a given function to many different goroutines without worrying about their identity.

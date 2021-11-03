# COMPSCI 377 LAB: Xv6 Filesystem

## Purpose

This lab is designed to familiarize students with the Xv6 learning operating system and more specifically familiarize students with the Xv6 filesytem. Please make sure that all of your answers to questions in these labs come from work done on the Edlab environment – otherwise, they may be inconsistent results and will not receive points.

Please submit your answers to this lab on Gradescope in the assignment marked “Lab #3’. All answers are due by the time specified on Gradescope. The TA present in your lab will do a brief explanation of the various parts of this lab, but you are expected to answer all questions by yourself. Please raise your hand if you have any questions during the lab section – TAs will be notified you are asking a question. Questions and Parts have a number of points marked next to them to signify their weight in this lab’s final grade. Labs are weighted equally, regardless of their total points.

Once you have logged in to Edlab, you can clone this lab repo using

```bash
git clone https://github.com/jbettencourt10/377-lab-filesystem_xv6.git
```

Then, clone the Xv6 repo into the 377-lab-gdb_xv6 directory using

```bash
git clone https://github.com/mit-pdos/xv6-public.git
```

Finally, you can use `cd` to open the directory you just cloned:

```bash
cd 377-lab-gdb_xv6
```

The xv6 OS includes a Makefile that allow you to locally compile the OS run all the sample code listed in this tutorial. You can compile them by running `make`. Feel free to modify the source files yourself, after making changes you can run `make` again to build new binaries from your modified files. You can also use `make clean` to remove all the built files, this command is usually used when something went wrong during the compilation so that you can start fresh.

## Part 1: Xv6 (5 Points)

Xv6 is a small operating system designed by MIT for students to learn how operating systems work. It includes several features we have or will learn about, like a scheduler, processes, file system, and system calls. For the sake of this lab, we will be investigating how Xv6 implements an ordinary filesystem by looking through the source code available to us.

To run xv6, first use the 'make' command in the directory. Then, after completion, use the 'make qemu-nox' command in the same directory. A shell will appear, and you will be using the Xv6 OS.

proc.c
```c++
//PAGEBREAK: 42
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run
//  - swtch to start running that process
//  - eventually that process transfers control
//      via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;

  for(;;){
    // Enable interrupts on this processor.
    sti();

    // Loop over process table looking for process to run.
    acquire(&ptable.lock);
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      // Switch to chosen process.  It is the process's job
      // to release ptable.lock and then reacquire it
      // before jumping back to us.
      c->proc = p;
      switchuvm(p);
      p->state = RUNNING;

      swtch(&(c->scheduler), p->context);
      switchkvm();

      // Process is done running for now.
      // It should have changed its p->state before coming back.
      c->proc = 0;
    }
    release(&ptable.lock);

  }
}
```

trap.c
```c++
  // Force process exit if it has been killed and is in user space.
  // (If it is still executing in the kernel, let it keep running
  // until it gets to the regular system call return.)
  if(myproc() && myproc()->killed && (tf->cs&3) == DPL_USER)
    exit();

  // Force process to give up CPU on clock tick.
  // If interrupts were on while locks held, would need to check nlock.
  if(myproc() && myproc()->state == RUNNING &&
     tf->trapno == T_IRQ0+IRQ_TIMER)
    yield();

  // Check if the process has been killed since we yielded
  if(myproc() && myproc()->killed && (tf->cs&3) == DPL_USER)
    exit();
```

From the above files, and the files listed on gradescope, we believe that looking through the source code should be a sufficient amount of information to answer the questions on Gradescope. However, there exists a wide range of documentation about Xv6 on the internet, so feel free to look for that if you find it necessary. A useful page is https://github.com/YehudaShapira/xv6-explained/blob/master/Explanations.md.

## Part 2: GDB (5 Points)

GDB stands for "GNU Project Debugger" and serves as the primary debugger for C and C++ programs. Typical debugging features like breakpoints, stepping, and analytics exist that allow you to pinpoint where issues exist in your code. In industrial applications, it is essential to be able to debug, so we hope this section of the lab will at least serve to introduce you to GDB and how useful it can be.

GDB is run with "gdb ____" where ___ is the name of the binary you wish to debug. Then, you can type "run" to run the program in the debugger. If some runtime error occurs, GDB will be able to provide you with information on what the issue could be.

Note: For debugging in your applications, you must include a "-g" flag when compiling to use GDB at its full potential. In this lab, however, the Makefile handles this flag already, so you do not need to worry about it.

 broken.cpp
```c++
#include <iostream>
using namespace std;

int factorial(int n) {
    return factorial(n-1) * n;
}

int main(){

    cout << "Beginning execution." << "\n";
    cout << factorial(10) << "\n";

}
```

With a very simple program like this, it could potentially be more useful to just look through the code and find the issue. Still, it is essential to be able to use GDB for large-scale programs where pinpointing issues could take hours or days. GDB provides a way to find issues automatically through execution, and can save a lot of time in the hands of a competent debugger.

## Please submit your answers to these questions on gradescope!

**This is not an officially supported Google product.**

# hawk

## 1. Overview

In this short tutorial you will learn how to write and compile a simple **LSM BPF program** that performs **bpf_trace_printk**. The program will be loaded into the kernel and will print a string after the execution of each process. Moreover, the program uses a BPF map of type PERCPU_ARRAY to keep track of the number of processes for each CPU, allowing userpace to print the total count each second.

You need two C files, which can be found in */src*:

**exec.c** (the file that contains the BPF program)
- SEC(str): shows what LSM hook to use to attach to the kernel
- int BPF_PROG(): the actual code for the BPF program that is executed

**user.c** (the file that contains the userspace program that loads the BPF program into the kernel using libbpf)
- the main function in which the BPF program is opened, loaded and attached to the kernel

## 2. Setup

Start by cloning this repo:
```
git clone https://github.com/Poppy22/hawk.git
```

**Linux kernel**

Get the latest version of the Linux kernel:
```
cd  # in your home directory
git clone https://github.com/torvalds/linux.git
```

**libbpf**

Install libbpf with the following commands:
```
cd linux/tools/lib/bpf
make
```

**[bpftool](https://www.mankier.com/8/bpftool)**

This is a tool for inspection and simple manipulation of BPF programs.
```
cd linux/tools/bpf/bpftool
make
sudo make install
```

## 3. Aliases

In your home directory, open .bashrc:
```
cd
vim .bashrc
```

Introduce aliases for the following commands:
```
__kcc()
{
  clang -g -D__TARGET_ARCH_x86 -mlittle-endian -Wno-compare-distinct-pointer-types -O2 -target bpf -emit-llvm -c $1 -o - | llc -march=bpf -mcpu=v2 -filetype=obj -o "$(basename  $1 .c).o";
}
alias kcc=__kcc

__ucc ()
{
  gcc -g $1 -o "$(basename $1 .c)" -I$HOME/linux/tools/lib/bpf  $HOME/linux/tools/lib/bpf/libbpf.a -lelf -lz
}
alias ucc=__ucc
```
Restart your terminal now.

## 4. Compile the BPF program

Go to src/kern and generate `vmlinux.h` with the following command:
```
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```
This header contains useful data structures and it should be included in `exec.c`.

Now compile exec.c:
```
kcc exec.c
```

Now, still from src/kern,  generate `exec.skel.h` with the following command:
```
bpftool gen skeleton exec.o > ../user/exec.skel.h
```
This skeleton header generated by bpftool simplifies the process of working with BPF programs, as it contains functions such as `exec__open_and_load` and `exec__attach`, which are used by the userspace program in order to open, load and attach the BPF program to the kernel. Needless to say, `user.c` should include this header file.

Go to src/user, where there should be `user.c` and `exec.skel.h`, and compile user.c
```
ucc user.c
```

## 5. Run the BPF program
In src/user, run the following command:
```
sudo ./user
```

Open a new terminal and type:
```
sudo cat /sys/kernel/debug/tracing/trace
```
The expected output is:
```
# tracer: nop
#
# entries-in-buffer/entries-written: 3840/3840   #P:8
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
           <...>-1677604 [004] .... 693150.730117: 0: hello
           <...>-1677607 [005] .... 693152.316666: 0: hello
           <...>-1677614 [005] .... 693152.731883: 0: hello
           <...>-1677620 [003] .... 693152.761767: 0: hello
           <...>-1677621 [006] .... 693152.763810: 0: hello
           <...>-1677621 [000] .... 693152.764590: 0: hello
 ```

 Notice how some processes printed "hello", which is what the BPF program was meant to do.

 **Bonus**:
 Go to `script/` and run:
 ```
 ./compile_run.sh
 ```
This will have the same effect as following the steps 4 and 5.

 **Congrats**, you compiled your first BPF program! 🎉

# new_syscall_5.18  
Creating a new vm.     
Boot the new vm with a new Linux kernel (version 5.18).  
Add to the Linux kernel a new "dummy" system system call before compiling it.  
Brief talk about the anatomy of a system call.

## Steps to follow
* `mkdir manjaro_vm`
* `cd manjaro_vm`
* Download a manjaro iso. And save it to this directory (let's say it is manjaro-xfce-21.3.7-minimal-220816-linux515.iso)
* `qemu-img create -f qcow2 manjaro_disk.img 10G`
> Note: the -f option specifies the format of the virtual hard disk, in this case, Qcow2. With the Qcow2 format, only enough disk space is allocated upfront. As you add more data to the guest operating system more disk space will be allocated dynamically.
* `qemu-system-x86_64 -m 2G -cdrom manjaro-xfce-21.3.7-minimal-220816-linux515.iso -boot d manjaro_disk.img -enable-kvm`
* Press `launch installer` when the `Welcome to Manjaro!` window will pop up
* Complete the configurations
* Check the installation `qemu-system-x86_64 -m 2G manjaro_disk.img -enable-kvm`
* Download the 5.18 Linux kernel `wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.18.tar.xz`
* `tar xf linux-5.18.tar.xz`
* `rm -rf linux-5.18.tar.xz`
* `cd linux-5.18`

## Add the new system call.
* `mkdir hello && cd hello`
* `vim hello.c`
* ```c
  #include <linux/kernel.h>
  #include <linux/syscalls.h>
  
  SYSCALL_DEFINE1(hello, unsigned long, i) {
    printk(KERN_ALERT "Hello World, i'm sys_hello with arg: %ld\n", i);
    return 0;
  }
  ```
* `vim Makefile`  
   Add `obj-y := hello.o`
* `cd ..`
* `vim Makefile`  
   Find `core-y   += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/`  
   And add `hello/`. So the line is now:  
   `core-y   += kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ hello/`  
* `vim arch/x86/entry/syscalls/syscall_64.tbl`
* Add in the end `548 64 hello     sys_hello`
> Note: the last syscall on my implementation had number 547.
* `vim include/linux/syscalls.h`
* Add in the end `asmlinkage long sys_hello(unsigned long i);`
* Now we are ready to compile the new kernel
* `make defconfig`
* `make -j4`
* `make modules`
* `make modules_install`
* `make install`
* Run VM with the new kernel:  
`qemu-system-x86_64 -m 2G -kernel "linux-5.18/arch/x86/boot/bzImage"  
 -append "root=/dev/sda1 console=ttyS0,115200n8 serial" -nographic    
 -hda manjaro_disk.img -enable-kvm -smp 2`
 * Test new syscall.  
 Inside the vm write a simple program like
 ```c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
        long int check = syscall(548, 42);
        printf("System call sys_hello returned %ld\n", check);
        return 0;
}
 ```
* If `printk()` is not showing in the screen just `dmesg | grep World` to see the line our syscall displayed 


## The anatomy of a system call
### What is a system call?  
System call is a function (more specifically a **service routine**) that lives inside the kernel space. A user issues a system call when she wants a paritcular functionality (usually to interact with a device) and the kernel takes the responsibility of the request because **users are not allowed to interact with devices without the permission of the kernel**.  
So inside the system call the kernel runs in favour of the process that made the call (in **process context**).  

One way to issue a system call from user-space (in `x86_64` architecture) is the `syscall` instruction.  
The `syscall` instruction jumps to the address stored in the `MSR_LSTAR` Model specific register* (Long system target address register).  
The kernel is responsible for providing its own custom function for handling syscalls as well as writing the address of this handler function to the `MSR_LSTAR` register upon system startup. The custom function is `entry_SYSCALL_64()`.
> *A model-specific register (MSR) is any of various control registers in the x86 instruction set used for debugging, program execution tracing, computer performance monitoring, and toggling certain CPU features.  
Reading and writing to these registers is handled by the `rdmsr` and `wrmsr` instructions, respectively. As these are privileged instructions, they can be executed only by the operating system.  
  
So during startup kernel fills the register with the address of the `entry_SYSCALL_64()`  
`wrmsrl(MSR_LSTAR, entry_SYSCALL_64);`.  
  
Ok, till now we know how to reach the kernel when issuing a system call from user space. But two questions arose.  
1. When and how **exactly** is happening the switch from user mode to kernel mode?  
2. How does the kernel knows which service routine (function) to execute in order to fullfill the user's request?  
  
We will try to answer these questions.  
1. From Intel's manual we can see that  
> SYSCALL invokes an OS system-call handler at privilege level 0. It does so by loading RIP from the IA32_LSTAR
MSR (after saving the address of the instruction following SYSCALL into RCX).  
SYSCALL loads the CS and SS selectors with values derived from  IA32_STAR MSR.  

So we need to inform `MSR_LSTAR` with the system call entry address, and `MSR_STAR` with the kernel CS selector.  
Linux kernel calls the `trap_init()` function during the initialization process. Between various initializations, this function calls the `cpu_init()`, which in turn calls the `syscall_init()` function.  
Inside `syscall_init()` function implementation we can see the following (kernel version 5.4):
```c
wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```
So bits 16-31 stores the user's CS and SS. These bits will be loaded to the CS and SS segment registers for the `sysret` instruction which provides functionality to return from a system call to user code.  
And bits 0-15 contains bits from the kernel code that will be used as the base selector for CS and SS registers when user space applications execute a system call.  
And this gives us all we need to join the dots from user space to the kernel code.  
A user invokes a system call. The `RIP` register loads the `entry_SYSCALL_64` address and the Code Segment register is being equal to the `__KERNEL_CS`.  
Kernel took control.  

2. Linux associates each system call with a natural number, in order to facilitate a user to issue the call. The user "tells" to the kernel "Hey i want to implement the 5th system call". The kernel reads the request and then goes to a table it keeps where each number is associated with a function pointer. A pointer to the coresponding service routine.  
This table is called system call table and is represented by `sys_call_table` array. Practically it is an array of the form:
```clike
sys_call_table[] = {
  [0] = sys_read,
  [1] = sys_write,
  [2] = sys_open,
  [3] = sys_close,
  ...
}; 
```

More specifically the user before the call to `syscall`, loads the contents of `%rax` register with the desired number of the system call.  
If the system call takes arguments they are loaded in `%rdi, %rsi, %rdx, %rcx, %r8, %r9` respectively. If it takes more than 6 arguments then the remaining arguments are stored in the stack.  






# new_syscall_5.18
Creating a new vm. 
Boot the new vm with a new Linux kernel (version 5.18)
Add to the Linux kernel a new "dummy" system system call before compiling it.

### Steps to follow
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

Add the new system call.
* `mkdir hello && cd hello`
* `vim hello.c`
* ```c
  #include <linux/kernel.h>
  #include <linux/syscalls.h>
  
  DEFINE_SYSCALL1(hello, unsigned long, i) {
    printk(KERN_ALERT "Hello World, i'm sys_hello with arg: %ld\n", i);
    return 0;
  }
  ```
* `vim Makefile`
* Add `obj-y := hello.o`
* `cd ..`
* `vim arch/x86/entry/syscalls/syscall_64.tbl`
* Add in the end `548 64 hello     sys_hello`
> Note: the last syscall on my implementation had number 547.
* `vim include/linux/syscalls.h`
* Add in the end `asmlinkage long sys_hello(unsigned long i);`
* Now we are ready to compile the new kernel
* `make defconfig`
* `make -j4`
* `make modules`
* `make moduels_install`
* `make install`
* Run VM with the new kernel:  
`qemu-system-x86_64 -m 2G -kernel "linux-5.18/arch/x86/boot/bzImage"  
 -append "root=/dev/sda1 console=ttyS0,115200n8 serial" -nographic    
 -hda manjaro_disk.img -enable-kvm -smp 2`
 * Test new syscall.  
 Inside the vm write a simple program like
 ```c
#include <stdio.h>
#include <linux/kernel.h>
#include <sys/syscall.h>
#include <unistd.h>

int main(int argc, char **argv) {
        long int helloCheck = syscall(548, 5);
        printf("System call sys_hello returned %ld\n", helloCheck);
        return 0;
}
 ```
* If `printk()` is not showing in the screen just `dmesg | grep World` to see the line our syscall displayed 

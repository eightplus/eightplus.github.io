---
title:      "为Android系统编写Linux内核驱动程序"
date:       2018-08-02 09:04:36
author:     "lixiang"
categories: 安卓开发
tags:
    - Android
---

> 这里使用一个虚拟的硬件设备，这个设备只有一个4字节的寄存器，它可读可写。这里把这个虚拟的设备命名为“mydev”，而这个内核驱动程序也命名为mydev驱动程序。<br><br>
> 这里通过cat命令直接访问/proc/mydev和/sys/class/mydev/mydev/val文件验证驱动程序的正确性。<br><br>
---

- 驱动程序目录结构如下：
```
~/Android/kernel/goldfish
----drivers
    ----mydev
        ----mydev.h
        ----mydev.c
        ----Kconfig
        ----Makefile
```

- 新建mydev目录
* ~/Android$ cd kernel/goldfish/drivers
* ~/Android/kernel/goldfish/drivers$ mkdir mydev

- 在mydev目录中增加mydev.h文件

``` c
#ifndef _MYDEV_ANDROID_H_
#define _MYDEV_ANDROID_Hpp
#include <linux/cdev.h>
#include <linux/semaphore.h>

#define MYDEV_DEVICE_NODE_NAME  "helmydevlo"
#define MYDEV_DEVICE_FILE_NAME  "mydev"
#define MYDEV_DEVICE_PROC_NAME  "mydev"
#define MYDEV_DEVICE_CLASS_NAME "mydev"

//字符设备结构体
struct mydev_android_dev {
    int val;//寄存器
    struct semaphore sem;//信号量
    struct cdev dev;//内嵌的字符设备
};

#endif
```

- 在mydev目录中增加mydev.c文件，这是驱动程序的实现部分。
驱动程序的功能主要是向上层提供访问设备的寄存器的值，包括读和写。这里，提供了三种访问设备寄存器的方法，一是通过proc文件系统来访问，二是通过传统的设备文件的方法来访问，三是通过devfs文件系统来访问。下面分段描述该驱动程序的实现。

 * 首先是包含必要的头文件和定义三种访问设备的方法：

``` cpp
#include <linux/init.h>
#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/proc_fs.h>
#include <linux/device.h>
#include <asm/uaccess.h>

#include "mydev.h"

/*主设备和从设备号变量*/
static int mydev_major = 0;
static int mydev_minor = 0;

/*设备类别和设备变量*/
static struct class* mydev_class = NULL;
static struct mydev_android_dev* mydev_dev = NULL;

/*传统的设备文件操作方法*/
static int mydev_open(struct inode* inode, struct file* filp);
static int mydev_release(struct inode* inode, struct file* filp);
static ssize_t mydev_read(struct file* filp, char __user *buf, size_t count, loff_t* f_pos);
static ssize_t mydev_write(struct file* filp, const char __user *buf, size_t count, loff_t* f_pos);

/**传统的设备文件操作方法表*/
static struct file_operations mydev_fops = {
    .owner = THIS_MODULE,
    .open = mydev_open,
    .release = mydev_release,
    .read = mydev_read,
    .write = mydev_write,
};

/*devfs文件系统的设备属性操作方法*/
static ssize_t mydev_val_show(struct device* dev, struct device_attribute* attr,  char* buf);
static ssize_t mydev_val_store(struct device* dev, struct device_attribute* attr, const char* buf, size_t count);

/*devfs文件系统的设备属性*/
static DEVICE_ATTR(val, S_IRUGO | S_IWUSR, mydev_val_show, mydev_val_store);

//定义传统的设备文件访问方法，主要是定义mydev_open、mydev_release、mydev_read和mydev_write这四个打开、释放、读和写设备文件的方法：

/*打开设备方法*/
static int mydev_open(struct inode* inode, struct file* filp) {
    struct mydev_android_dev* dev;

    /*将自定义设备结构体保存在文件指针的私有数据域中，以便访问设备时拿来用*/
    dev = container_of(inode->i_cdev, struct mydev_android_dev, dev);
    filp->private_data = dev;

    return 0;
}

/*设备文件释放时调用，空实现*/
static int mydev_release(struct inode* inode, struct file* filp) {
    return 0;
}

/*读取设备的寄存器val的值*/
static ssize_t mydev_read(struct file* filp, char __user *buf, size_t count, loff_t* f_pos) {
    ssize_t err = 0;
    struct mydev_android_dev* dev = filp->private_data;

    /*同步访问*/
    if(down_interruptible(&(dev->sem))) {
        return -ERESTARTSYS;
    }

    if(count < sizeof(dev->val)) {
        goto out;
    }

    /*将寄存器val的值拷贝到用户提供的缓冲区*/
    if(copy_to_user(buf, &(dev->val), sizeof(dev->val))) {
        err = -EFAULT;
        goto out;
    }

    err = sizeof(dev->val);

out:
    up(&(dev->sem));
    return err;
}

/*写设备的寄存器值val*/
static ssize_t mydev_write(struct file* filp, const char __user *buf, size_t count, loff_t* f_pos) {
    struct mydev_android_dev* dev = filp->private_data;
    ssize_t err = 0;

    /*同步访问*/
    if(down_interruptible(&(dev->sem))) {
        return -ERESTARTSYS;
    }

    if(count != sizeof(dev->val)) {
        goto out;
    }

    /*将用户提供的缓冲区的值写到设备寄存器去*/
    if(copy_from_user(&(dev->val), buf, count)) {
        err = -EFAULT;
        goto out;
    }

    err = sizeof(dev->val);

out:
    up(&(dev->sem));
    return err;
}

//定义通过devfs文件系统访问方法，这里把设备的寄存器val看成是设备的一个属性，通过读写这个属性来对设备进行访问，主要是实现mydev_val_show和mydev_val_store两个方法，同时定义了两个内部使用的访问val值的方法__mydev_get_val和__mydev_set_val：

/*读取寄存器val的值到缓冲区buf中，内部使用*/
static ssize_t __mydev_get_val(struct mydev_android_dev* dev, char* buf) {
    int val = 0;

    /*同步访问*/
    if(down_interruptible(&(dev->sem))) {
        return -ERESTARTSYS;
    }

    val = dev->val;
    up(&(dev->sem));

    return snprintf(buf, PAGE_SIZE, "%d\n", val);
}

/*把缓冲区buf的值写到设备寄存器val中去，内部使用*/
static ssize_t __mydev_set_val(struct mydev_android_dev* dev, const char* buf, size_t count) {
    int val = 0;

    /*将字符串转换成数字*/
    val = simple_strtol(buf, NULL, 10);

    /*同步访问*/
    if(down_interruptible(&(dev->sem))) {
        return -ERESTARTSYS;
    }

    dev->val = val;
    up(&(dev->sem));

    return count;
}

/*读取设备属性val*/
static ssize_t mydev_val_show(struct device* dev, struct device_attribute* attr, char* buf) {
    struct mydev_android_dev* hdev = (struct mydev_android_dev*)dev_get_drvdata(dev);

    return __mydev_get_val(hdev, buf);
}

/*写设备属性val*/
static ssize_t mydev_val_store(struct device* dev, struct device_attribute* attr, const char* buf, size_t count) {
    struct mydev_android_dev* hdev = (struct mydev_android_dev*)dev_get_drvdata(dev);

    return __mydev_set_val(hdev, buf, count);
}

//定义通过proc文件系统访问方法，主要实现了mydev_proc_read和mydev_proc_write两个方法，同时定义了在proc文件系统创建和删除文件的方法mydev_create_proc和mydev_remove_proc：

/*读取设备寄存器val的值，保存在page缓冲区中*/
static ssize_t mydev_proc_read(char* page, char** start, off_t off, int count, int* eof, void* data) {
    if(off > 0) {
        *eof = 1;
        return 0;
    }

    return __mydev_get_val(mydev_dev, page);
}

/*把缓冲区的值buff保存到设备寄存器val中去*/
static ssize_t mydev_proc_write(struct file* filp, const char __user *buff, unsigned long len, void* data) {
    int err = 0;
    char* page = NULL;

    if(len > PAGE_SIZE) {
        printk(KERN_ALERT"The buff is too large: %lu.\n", len);
        return -EFAULT;
    }

    page = (char*)__get_free_page(GFP_KERNEL);
    if(!page) {
            printk(KERN_ALERT"Failed to alloc page.\n");
            return -ENOMEM;
    }

    /*先把用户提供的缓冲区值拷贝到内核缓冲区中去*/
    if(copy_from_user(page, buff, len)) {
        printk(KERN_ALERT"Failed to copy buff from user.\n");
        err = -EFAULT;
        goto out;
    }

    err = __mydev_set_val(mydev_dev, page, len);

out:
    free_page((unsigned long)page);
    return err;
}

/*创建/proc/mydev文件*/
static void mydev_create_proc(void) {
    struct proc_dir_entry* entry;

    entry = create_proc_entry(MYDEV_DEVICE_PROC_NAME, 0, NULL);
    if(entry) {
        entry->owner = THIS_MODULE;
        entry->read_proc = mydev_proc_read;
        entry->write_proc = mydev_proc_write;
    }
}

/*删除/proc/mydev文件*/
static void mydev_remove_proc(void) {
    remove_proc_entry(MYDEV_DEVICE_PROC_NAME, NULL);
}

//最后，定义模块加载和卸载方法，这里只要是执行设备注册和初始化操作：

/*初始化设备*/
static int  __mydev_setup_dev(struct mydev_android_dev* dev) {
    int err;
    dev_t devno = MKDEV(mydev_major, mydev_minor);

    memset(dev, 0, sizeof(struct mydev_android_dev));

    cdev_init(&(dev->dev), &mydev_fops);
    dev->dev.owner = THIS_MODULE;
    dev->dev.ops = &mydev_fops;

    /*注册字符设备*/
    err = cdev_add(&(dev->dev),devno, 1);
    if(err) {
        return err;
    }

    /*初始化信号量和寄存器val的值*/
    init_MUTEX(&(dev->sem));
    dev->val = 0;

    return 0;
}

/*模块加载方法*/
static int __init mydev_init(void){
    int err = -1;
    dev_t dev = 0;
    struct device* temp = NULL;

    printk(KERN_ALERT"Initializing mydev device.\n");

    /*动态分配主设备和从设备号*/
    err = alloc_chrdev_region(&dev, 0, 1, MYDEV_DEVICE_NODE_NAME);
    if(err < 0) {
        printk(KERN_ALERT"Failed to alloc char dev region.\n");
        goto fail;
    }

    mydev_major = MAJOR(dev);
    mydev_minor = MINOR(dev);

    /*分配helo设备结构体变量*/
    mydev_dev = kmalloc(sizeof(struct mydev_android_dev), GFP_KERNEL);
    if(!mydev_dev) {
        err = -ENOMEM;
        printk(KERN_ALERT"Failed to alloc mydev_dev.\n");
        goto unregister;
    }

    /*初始化设备*/
    err = __mydev_setup_dev(mydev_dev);
    if(err) {
        printk(KERN_ALERT"Failed to setup dev: %d.\n", err);
        goto cleanup;
    }

    /*在/sys/class/目录下创建设备类别目录mydev*/
    mydev_class = class_create(THIS_MODULE, MYDEV_DEVICE_CLASS_NAME);
    if(IS_ERR(mydev_class)) {
        err = PTR_ERR(mydev_class);
        printk(KERN_ALERT"Failed to create mydev class.\n");
        goto destroy_cdev;
    }

    /*在/dev/目录和/sys/class/mydev目录下分别创建设备文件mydev*/
    temp = device_create(mydev_class, NULL, dev, "%s", MYDEV_DEVICE_FILE_NAME);
    if(IS_ERR(temp)) {
        err = PTR_ERR(temp);
        printk(KERN_ALERT"Failed to create mydev device.");
        goto destroy_class;
    }

    /*在/sys/class/mydev/mydev目录下创建属性文件val*/
    err = device_create_file(temp, &dev_attr_val);
    if(err < 0) {
        printk(KERN_ALERT"Failed to create attribute val.");
        goto destroy_device;
    }

    dev_set_drvdata(temp, mydev_dev);

    /*创建/proc/mydev文件*/
    mydev_create_proc();

    printk(KERN_ALERT"Succedded to initialize mydev device.\n");
    return 0;

destroy_device:
    device_destroy(mydev_class, dev);

destroy_class:
    class_destroy(mydev_class);

destroy_cdev:
    cdev_del(&(mydev_dev->dev));

cleanup:
    kfree(mydev_dev);

unregister:
    unregister_chrdev_region(MKDEV(mydev_major, mydev_minor), 1);

fail:
    return err;
}

/*模块卸载方法*/
static void __exit mydev_exit(void) {
    dev_t devno = MKDEV(mydev_major, mydev_minor);

    printk(KERN_ALERT"Destroy mydev device.\n");

    /*删除/proc/mydev文件*/
    mydev_remove_proc();

    /*销毁设备类别和设备*/
    if(mydev_class) {
        device_destroy(mydev_class, MKDEV(mydev_major, mydev_minor));
        class_destroy(mydev_class);
    }

    /*删除字符设备和释放设备内存*/
    if(mydev_dev) {
        cdev_del(&(mydev_dev->dev));
        kfree(mydev_dev);
    }

    /*释放设备号*/
    unregister_chrdev_region(devno, 1);
}

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Fake Register Android Driver");

module_init(mydev_init);
module_exit(mydev_exit);
```

- 在mydev目录中新增Kconfig和Makefile两个文件，其中Kconfig是在编译前执行配置命令make menuconfig时用到的，而Makefile是执行编译命令make是用到的
> 在Kconfig文件中，tristate表示编译选项MYDEV支持在编译内核时，mydev模块支持以模块、内建和不编译三种编译方法，默认是不编译，因此，在编译内核前，我们还需要执行make menuconfig命令来配置编译选项，使得mydev可以以模块或者内建的方法进行编译。
> 在Makefile文件中，根据选项MYDEV的值，执行不同的编译方法。
    - Kconfig文件的内容(默认的编译方式为n，即不编译到内核中，故在编译驱动程序之前，需要执行make menuconfig命令来修改编译选项)
```
config MYDEV
	tristate "Fake Register Android Driver"
	default n
	help
	This is the mydev driver for android system.
```

    - Makefile
``` makefile
obj-$(CONFIG_MYDEV) += mydev.o
```

- 修改内核Kconfig文件
    - 打开arch/arm/Kconfig文件，找到以下两行内容
```
menu "Device Drivers"
......
endmenu
```
在这两行内容之间添加下面一行内容，将驱动程序mydev和Kconfig文件包含尽量。
```
menu "Device Drivers"
source "drivers/mydev/Kconfig"
......
endmenu
```
这样，执行make menuconfig时，就可以配置mydev模块的编译选项了。
    - 打开drivers/Kconfig，和arch/arm/Kconfig一样，增加一行内容：source "drivers/mydev/Kconfig"

- 修改drivers/Makefile文件，添加一行：
```
obj-$(CONFIG_MYDEV) += mydev/
```

- 配置编译选项：
* ~/Android/kernel/goldfish$ make menuconfig
找到"Device Drivers" => "Fake Register Android Drivers"选项，设置为y。
> 注意，如果内核不支持动态加载模块，这里不能选择m，虽然我们在Kconfig文件中配置了MYDEV选项为tristate。要支持动态加载模块选项，必须要在配置菜单中选择Enable loadable module support选项；在支持动态卸载模块选项，必须要在Enable loadable module support菜单项中，选择Module unloading选项。

- 编译：
* ~/Android/kernel/goldfish$ make
编译成功后，就可以在mydev目录下看到mydev.o文件了，这时候编译出来的zImage已经包含了mydev驱动。

运行新编译的内核文件，验证mydev驱动程序是否已经正常安装
* ~/Android$ emulator -kernel ./kernel/goldfish/arch/arm/boot/zImage &
* ~/Android$ adb shell

进入到dev目录，可以看到mydev设备文件
* root@android:/ # cd dev
* root@android:/dev # ls

进入到proc目录，可以看到mydev文件：
* root@android:/ # cd proc
* root@android:/proc # ls

访问mydev文件的值
* root@android:/proc # cat mydev
0
* root@android:/proc # echo '5' > mydev
* root@android:/proc # cat mydev
5

进入到sys/class目录，可以看到mydev目录：
* root@android:/ # cd sys/class
* root@android:/sys/class # ls

进入到mydev目录，可以看到mydev目录：
* root@android:/sys/class # cd mydev
* root@android:/sys/class/mydev # ls

进入到下一层mydev目录，可以看到val文件
* root@android:/sys/class/mydev # cd mydev
* root@android:/sys/class/mydev/mydev # ls

访问属性文件val的值
* root@android:/sys/class/mydev/mydev # cat val
5
* root@android:/sys/class/mydev/mydev # echo '0'  > val
* root@android:/sys/class/mydev/mydev # cat val
0

至此，mydev内核驱动程序就完成了。这里采用的是系统提供的方法和驱动程序进行交互，也就是通过proc文件系统和devfs文件系统的方法。

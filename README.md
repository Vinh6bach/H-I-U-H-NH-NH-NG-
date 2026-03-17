# H-I-U-H-NH-NH-NG-


# 📘 Bài tập HDH Nhúng – Giao tiếp Device Driver (BBB)

## 🎯 Mục tiêu

* Xây dựng **Device Driver LED** cho BeagleBone Black (BBB)
* Tạo file thiết bị `/dev/led`
* Giao tiếp từ user space bằng:

  * `echo`
  * `cat`
* Điều khiển LED:

  * ON
  * OFF

---

## 🧱 Cấu trúc thư mục

```
buildroot/
└── package/
    └── led_driver/
        ├── Config.in
        ├── led_driver.mk
        └── src/
            ├── led_driver.c
            └── Makefile
```

---

## ⚙️ 1. Tạo package trong Buildroot

```bash
cd buildroot/package
mkdir led_driver
cd led_driver
mkdir src
```

---

## 📄 2. File `Config.in`

```bash
config BR2_PACKAGE_LED_DRIVER
    bool "LED driver"
    help
      Simple LED driver for BBB
```

---

## 📄 3. File `led_driver.mk`

```make
LED_DRIVER_VERSION = 1.0
LED_DRIVER_SITE = package/led_driver/src
LED_DRIVER_SITE_METHOD = local

define LED_DRIVER_BUILD_CMDS
	$(MAKE) ARCH=arm CROSS_COMPILE=$(TARGET_CROSS) -C $(LINUX_DIR) M=$(@D) modules
endef

define LED_DRIVER_INSTALL_TARGET_CMDS
	mkdir -p $(TARGET_DIR)/lib/modules
	cp $(@D)/led_driver.ko $(TARGET_DIR)/lib/modules/
endef

$(eval $(generic-package))
```

---

## 📄 4. File `Makefile` (trong src)

```make
obj-m += led_driver.o
```

---

## 📄 5. File `led_driver.c`

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/gpio.h>

#define DEVICE_NAME "led"
#define GPIO_LED 60   // P9_12

static int major;
static char msg[2];

static ssize_t led_write(struct file *file, const char *buf, size_t len, loff_t *offset)
{
    copy_from_user(msg, buf, 1);

    if (msg[0] == '1') {
        gpio_set_value(GPIO_LED, 0);
        printk("LED ON\n");
    } 
    else if (msg[0] == '0') {
        gpio_set_value(GPIO_LED, 1);
        printk("LED OFF\n");
    }

    return len;
}

static struct file_operations fops = {
    .write = led_write,
};

static int __init led_init(void)
{
    major = register_chrdev(0, DEVICE_NAME, &fops);

    gpio_request(GPIO_LED, "led");
    gpio_direction_output(GPIO_LED, 1);

    printk("LED driver loaded\n");
    return 0;
}

static void __exit led_exit(void)
{
    gpio_set_value(GPIO_LED, 1);
    gpio_free(GPIO_LED);
    unregister_chrdev(major, DEVICE_NAME);

    printk("LED driver removed\n");
}

module_init(led_init);
module_exit(led_exit);

MODULE_LICENSE("GPL");
```

---

## 🔗 6. Thêm package vào Buildroot

Mở file:

```bash
nano buildroot/package/Config.in
```

Thêm dòng:

```bash
source "package/led_driver/Config.in"
```

---

## ⚙️ 7. Enable driver

```bash
make menuconfig
```

Chọn:

```
Target packages → LED driver → [*]
```

---

## 🔨 8. Build hệ thống

```bash
make
```

---

## 💿 9. Flash vào SD Card

```bash
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M status=progress
```

---

## 🔌 10. Trên BeagleBone Black

### Load driver

```bash
insmod /lib/modules/led_driver.ko
```

---

## 🔧 11. Tạo device `/dev/led`

```bash
cat /proc/devices | grep led
```

→ Lấy **major number**

```bash
mknod /dev/led c <major> 0
```

---

## 🎮 12. Điều khiển LED

### Bật LED

```bash
echo 1 > /dev/led
```

### Tắt LED

```bash
echo 0 > /dev/led
```

---

## 📌 13. Giải thích

* `echo` → gửi dữ liệu từ user space xuống driver
* Driver nhận trong hàm `write()`
* GPIO được điều khiển qua:

  ```c
  gpio_set_value()
  ```

---

## ✅ Kết quả đạt được

* ✔ Tạo thành công device `/dev/led`
* ✔ Giao tiếp qua `echo`
* ✔ Điều khiển LED ON/OFF
* ✔ Driver hoạt động ổn định trên BBB

---

## 🚀 Ghi chú

* Cần `insmod` lại sau mỗi lần reboot
* Có thể dùng `rm /dev/led` để tạo lại device
* GPIO có thể thay đổi tùy chân sử dụng



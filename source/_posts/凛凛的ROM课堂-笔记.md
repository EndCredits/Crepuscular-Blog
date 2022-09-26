---
title: 凛凛的ROM课堂 -- 笔记
date: 2022-09-26 12:23:24
cover: https://avatars.githubusercontent.com/u/30337499?v=4
---

前一段 b 站直播凛凛没开麦... 斗胆加上一点点自己的理解（

## 基本结构

最基础的文件有以下几个 

```
cce87ff thyme: Initial tree from lisa
```

ROOT

```
|-Android.bp
|-Android.mk
|-AndroidProducts.mk
|-BoardConfig.mk
|-device.mk
|-lineage_thyme.mk
|-recovery
| |-recovery.fstab
|-rootdir
| |-Android.mk
| |-etc
| | |-fstab.qcom
| | |-init.recovery.qcom.rc
```

因为只是初始化了一个设备树的框架，所以比起 Github 上的成品 dt 会显得少很多东西，不要担心，之后我们会根据需要分别加入需要的部分

## 它们有什么用?

- Android.bp
 
    它定义了一个 soong namespace ，指向你的 dt ，至于什么是 soong namespace ... 说来话长，可以理解为让 soong 编译系统能够发现你的 dt 以及里面配置的编译选项，它通常也会用于导入其他的 soong namespace，这样我们就可以选择编译导入的那个 namespace 里定义的模块，例如下面的这个例子，定义了自己的命名空间，以及导入了一个命名空间 ```hardware/xiaomi```

    ```blueprint
    soong_namespace {
        imports: [
            "hardware/xiaomi",
        ],
    }
    ```

- Android.mk
    
    主要用于 blueprint 还不成熟时，定义编译模块。它还会与 ```make``` 直接打交道，它通常至少有以下内容 ( 以 ```picasso``` 为例 )

    ```Makefile
    LOCAL_PATH := $(call my-dir)

    ifeq ($(TARGET_DEVICE),picasso)

    include $(call all-makefiles-under,$(LOCAL_PATH))

    include $(CLEAR_VARS)

    endif
    ```

    上面的内容告诉 ```make``` ，如果定义了 ```picasso```，这个设备，那么调用当前目录下所有的 ```Makefile``` 来控制这个产品的编译。 ```ifeq``` 等等关键字是 Makefile 语法自带的关键字， ```my-dir```, ```all-makefiles-under```, 是 AOSP 为你预先写好的工具函数，简化你的工作，详细信息可以查阅 AOSP 的文档以及编译系统源代码，这些函数在整个编译系统以及我们设备的定义中会很常见

    Android.mk 还有一个常见的用处就是可以创建一些 ```symlink```，允许我们在编译时就把一些系统必要的挂载点创建好，这对一些服务的启动是必需的，例如 ( 以 ```picasso``` 为例 )

    ```Makefile
    # A/B builds require us to create the mount points at compile time.
    # Just creating it for all cases since it does not hurt.
    FIRMWARE_MOUNT_POINT := $(TARGET_OUT_VENDOR)/firmware_mnt
    $(FIRMWARE_MOUNT_POINT): $(LOCAL_INSTALLED_MODULE)
	    @echo "Creating $(FIRMWARE_MOUNT_POINT)"
	    @mkdir -p $(TARGET_OUT_VENDOR)/firmware_mnt

    BT_FIRMWARE_MOUNT_POINT := $(TARGET_OUT_VENDOR)/bt_firmware
    $(BT_FIRMWARE_MOUNT_POINT): $(LOCAL_INSTALLED_MODULE)
	    @echo "Creating $(BT_FIRMWARE_MOUNT_POINT)"
	    @mkdir -p $(TARGET_OUT_VENDOR)/bt_firmware

    DSP_MOUNT_POINT := $(TARGET_OUT_VENDOR)/dsp
    $(DSP_MOUNT_POINT): $(LOCAL_INSTALLED_MODULE)
	    @echo "Creating $(DSP_MOUNT_POINT)"
	    @mkdir -p $(TARGET_OUT_VENDOR)/dsp

    ALL_DEFAULT_INSTALLED_MODULES += $(FIRMWARE_MOUNT_POINT) $(BT_FIRMWARE_MOUNT_POINT) $(DSP_MOUNT_POINT)

    ```
    这里定义了 firmware 的挂载点，在编译初期你会看到

    ```
    Creating xxxx mount point ...
    ```

    就是它们在起作用

    对于高通设备，这些内容通常都可以在 CLO 对应的 Soc 仓库中找到

- AndroidProducts.mk

    它的作用是让你的设备能够被 AOSP 编译系统识别，也就是定义了你在使用 ```lunch``` 命令的时候，后面应该传递的参数，内容通常比较简单。

    ```Makefile
    PRODUCT_MAKEFILES := \
        $(LOCAL_DIR)/aosp_picasso.mk

    COMMON_LUNCH_CHOICES := \
        aosp_picasso-userdebug \
        aosp_picasso-eng
    ```

    以前的编译系统还会用到 ```add_lunch_combo``` 这个函数，不过现在已经被弃用了，改成了定义 ```COMMON_LUNCH_CHOICES``` (如上代码框所示)

- lineage_thyme.mk

    这里定义了跟你设备有关的信息，比如它叫什么名字，是什么牌子的，编译它时应该继承哪些配置文件，特点是什么，GMS Client Base 等等，同样地，以编译我的 AOSP 的 ```picasso``` 为例

    ```aosp_picasso.mk```

    可以看到，这个文件的命名是有规律的，下划线前面的部分通常是你要编译的 rom 的代号，下划线后面的部分通常是你的设备代号，这个文件的名字应该与上面定义在 ```AndroidProducts.mk``` 里 ```PRODUCT_MAKEFILES``` 的名字一致

    ```Makefile
    # Inherit from those products. Most specific first.
    $(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
    $(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk)

    # Inherit from picasso device
    $(call inherit-product, device/xiaomi/picasso/device.mk)

    # Inherit some common AOSP stuff.
    $(call inherit-product, vendor/aosp/config/common.mk)

    # Add some Crepuscular AOSP Feature
    TARGET_SUPPORTS_QUICK_TAP := true
    TARGET_FACE_UNLOCK_SUPPORTED := true
    TARGET_INCLUDE_PIXEL_CHARGER := true

    # Device identifier. This must come after all inclusions.
    PRODUCT_NAME := aosp_picasso
    PRODUCT_DEVICE := picasso
    PRODUCT_MODEL := Redmi K30 5G
    PRODUCT_BRAND := Redmi
    PRODUCT_MANUFACTURER := Xiaomi

    TARGET_BOOT_ANIMATION_RES := 1080

    PRODUCT_GMS_CLIENTID_BASE := android-xiaomi
    ```
    英语好的同学应该已经可以明白上面的东西定义的是什么了，它们的名字比较直球的表示了它们的作用

    首先是继承 AOSP 产品级配置文件

    ```$ANDROID_ROOT/build/make/target/product/core_64_bit.mk```
    ```$ANDROID_ROOT/build/make/target/product/full_base_telephony.mk```

    有兴趣的同学可以去查阅以下源代码它们到底定义了什么，总结一下就是集合了 Android 整个系统的绝大部分代码

    然后就是继承你的设备配置文件

    ```$ANDROID_ROOT/device/xiaomi/picasso/device.mk```

    这个文件待会还会讲到，里面定义了什么 HAL 以及其他杂七杂八的必要组件应该被编译进你的系统

    然后就是继承你的 rom 的配置文件，这里面通常包含了 ROM 特有的一些附加功能，以及一些对 AOSP 编译系统必要的补充，比如支持内核的编译等等

    ```$ANDROID_ROOT/vendor/aosp/config/common.mk```

    下面的就顾名思义就可以了，意义比较直球

- BoardConfig.mk

    比较重要的文件，定义了你的设备的所有的板级细节，这里也是凛凛着重讲解的一部分

    代码比较多，我们分块来读 ( Dyncmic A-only 设备以 picasso 为例，AB 设备以 thyme 为例)

    首先是架构体系相关的信息（ 同一个 platform 的设备这里可以直接抄，以 ```picasso``` 为例 ）

    ```Makefile
    # Architecture
    TARGET_ARCH := arm64
    TARGET_ARCH_VARIANT := armv8-a
    TARGET_CPU_ABI := arm64-v8a
    TARGET_CPU_ABI2 :=
    TARGET_CPU_VARIANT := cortex-a76

    TARGET_2ND_ARCH := arm
    TARGET_2ND_ARCH_VARIANT := armv8-a
    TARGET_2ND_CPU_ABI := armeabi-v7a
    TARGET_2ND_CPU_ABI2 := armeabi
    TARGET_2ND_CPU_VARIANT := cortex-a76
    ```

    这里定义了你的 Soc 是什么体系结构以及它所使用的变体，比如 ```picasso``` 是 ```lito``` 也就是 ```sm7250``` 平台，那么它的定义就是上面那个样子，如果你相关的平台还没有设备有设备树的话，那么这些信息去 Google 以下具体的 CPU 型号也都可以查的到

    然后是 ```bootloader``` 的名字

    ```Makefile
    # Bootloader
    TARGET_BOOTLOADER_BOARD_NAME := lito
    ```

    它定义了你的 Soc 的代号，可以通过这个命令查到

    ```bash
    adb shell getprop ro.board.platform
    ```
    或者也可以像凛凛直播中讲的那样，去把它的信息 dump 出来然后翻 prop，来定义它，这里定义它的作用主要是为了编译一些跟你 Soc 相关的 HAL ，比如 ```audio | display | media``` 都会用到这个

    下面是 ```A/B``` 机型特别所需的内容

    ```Makefile
    # A/B
    AB_OTA_UPDATER := true

    AB_OTA_PARTITIONS += \
        boot \
        dtbo \
        odm \
        product \
        system \
        system_ext \
        vbmeta \
        vbmeta_system \
        vendor_boot \
        vendor
    ```


    上面的 ```AB_OTA_UPDATER``` 标志你的设备支持 AB 无缝系统更新， Legacy AB |  Dynamic AB | Virtual AB 都会需要这个，下面的 ```AB_OTA_PARTITIONS``` 定义了你的设备具有 AB 槽位的分区，这些分区会以 xxx_a 和 xxx_b 的形式出现在你的分区表里，你可以通过 ```fastboot``` 返回的信息确认你有哪些分区采用了 ab 架构

    ```bash
    fastboot getvar all
    ```
    下面是内核的编译选项 (以 ```picasso``` 为例)

    ```Makefile
    # Kernel
    BOARD_KERNEL_CMDLINE := console=ttyMSM0,115200n8 androidboot.hardware=qcom androidboot.console=ttyMSM0 androidboot.memcg=1 lpm_levels.sleep_disabled=1 msm_rtb.filter=0x237 service_locator.enable=1 androidboot.usbcontroller=a600000.dwc3 swiotlb=2048 cgroup.memory=nokmem,nosocket loop.max_part=7 reboot=panic_warm
    BOARD_KERNEL_CMDLINE += androidboot.init_fatal_reboot_target=recovery
    # BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive
    BOARD_KERNEL_IMAGE_NAME := Image
    BOARD_BOOTIMG_HEADER_VERSION := 2
    BOARD_KERNEL_BASE := 0x00000000
    BOARD_KERNEL_PAGESIZE := 4096
    BOARD_KERNEL_SEPARATED_DTBO := true
    BOARD_MKBOOTIMG_ARGS += --header_version $(BOARD_BOOTIMG_HEADER_VERSION)
    TARGET_KERNEL_ADDITIONAL_FLAGS := DTC_EXT=$(shell pwd)/prebuilts/misc/$(HOST_OS)-x86/dtc/dtc
    TARGET_KERNEL_VERSION := 4.19
    TARGET_KERNEL_CLANG_COMPILE := true
    TARGET_KERNEL_SOURCE := kernel/xiaomi/sm7250
    TARGET_KERNEL_CONFIG := vendor/picasso_user_defconfig

    ```

    内核编译是相对重要的一段，这里给出的实例是 oss kernel 的编译配置，prebuilt kernel 将会在下面给出

    其中的 ```BOARD_KERNEL_CMDLINE``` 定义了传递给内核的可以理解为启动参数的命令行，这些当然不可能是我们默写出来的，它可以通过解包厂商 rom 里的 boot.img 得到，或者也可以通过 adb shell 查询
    ```bash
    adb shell cat /proc/cmdline
    ```
    如果想要解包 boot 的话可以使用 unpackbootimg，使用方法可以查阅 [anestisb/android-unpackbootimg](https://github.com/anestisb/android-unpackbootimg)

    或者提供一个更好解包工具集 [ShivamKumarJha/android_tools](https://github.com/ShivamKumarJha/android_tools)

    然后是你的 ```BOARD_KERNEL_IMAGE_NAME``` ，这里定义了你的内核编译产物是什么，对于 4.19 平台，如果没有开启压缩的话，默认都是 ```Image```，开启压缩可能是 ```Image.gz```，对于 4.9 及以下的平台 ( 使用 bootheader v1 的设备 )，编译产物可能是 ```Image.gz-dtb``` 或者 ```Image-dtb``` ，它们的 dtb 附加在内核文件的后面，这一点可以查看你的内核 ```defconfig``` 是否开启了```CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE```，如果开启，那么就使用连接 dtb 的形式，如果没有开启这个而是开启了 ```CONFIG_BUILD_ARM64_DT_OVERLAY```，说明你的 bootheader version 至少在 v2 及以上，那么就使用单独的 ```Image``` 形式

    bootheader 版本也可以通过解包 boot.img 得到

    关于 bootheader 的更多详细信息，请查阅: [Boot Image Header - Android Open Source Project](https://source.android.com/docs/core/architecture/bootloader/boot-image-header)，这里只做简单讲解，注意，以下直到讲解结束的 Android 版本均指设备出厂时的 Android 版本

    Boot Image Header 定义了一系列 Android 引导加载程序所使用的标准，在 Android 9 之前，它属于 bootheader v0

    ```C
    struct boot_img_hdr
    {
        uint8_t magic[BOOT_MAGIC_SIZE];
        uint32_t kernel_size;                /* size in bytes */
        uint32_t kernel_addr;                /* physical load addr */

        uint32_t ramdisk_size;               /* size in bytes */
        uint32_t ramdisk_addr;               /* physical load addr */

        uint32_t second_size;                /* size in bytes */
        uint32_t second_addr;                /* physical load addr */

        uint32_t tags_addr;                  /* physical addr for kernel tags */
        uint32_t page_size;                  /* flash page size we assume */
        uint32_t unused;
        uint32_t os_version;
        uint8_t name[BOOT_NAME_SIZE];        /* asciiz product name */
        uint8_t cmdline[BOOT_ARGS_SIZE];
        uint32_t id[8];                      /* timestamp / checksum / sha1 / etc */
        uint8_t extra_cmdline[BOOT_EXTRA_ARGS_SIZE];
    };
    ```
    它有一个 ```uint32_t``` 类型的 ```unused``` 的保留字段

    Android 9 正式支持了 bootheader 这一特性，其将保留字段更新为 ```header_version``` ，并新增了以下内容，该特性是 VTS 的强制要求

    ```C
    struct boot_img_hdr
    {
        ...
        uint32_t page_size;                  /* flash page size we assume */
        uint32_t header_version;
        uint32_t os_version;
        ...
        uint32_t recovery_[dtbo|acpio]_size;    /* size of recovery image */
        uint64_t recovery_[dtbo|acpio]_offset;  /* offset in boot image */
        uint32_t header_size;               /* size of boot image header in bytes */
    }
    ```

    这时候设备就可以添加用于 recovery 的 dtbo | acpio 的内容了

    在 Android 10 ， bootheader 更新到了 v2，添加了 dtb 相关信息的字段

    ```C
    struct boot_img_hdr
    {
        ...
        uint32_t header_size;               /* size of boot image header in bytes */
        uint32_t dtb_size;                  /* size of dtb image */
        uint64_t dtb_addr;                  /* physical load address */
    };
    ```
    可以看到，这个时候 dtb 从内核上分离了出来，有了单独的存储区域，这也是为什么不再使用在内核后连接 dtb 的编译形式，```CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE``` 也就不会被打开了

    而 Android 11 ，bootheader 更新至 v3，改动较大
    
    1. 移除了第二阶段引导加载程序的相关信息
    2. 移除了 recovery 分区的相关信息
    3. 移除了 dtb 的相关信息


    ```C
    struct boot_img_hdr
    {
    #define BOOT_MAGIC_SIZE 8
        uint8_t magic[BOOT_MAGIC_SIZE];

        uint32_t kernel_size;    /* size in bytes */
        uint32_t ramdisk_size;   /* size in bytes */

        uint32_t os_version;

        uint32_t header_size;    /* size of boot image header in bytes */
        uint32_t reserved[4];
        uint32_t header_version; /* offset remains constant for version check */

    #define BOOT_ARGS_SIZE 512
    #define BOOT_EXTRA_ARGS_SIZE 1024
        uint8_t cmdline[BOOT_ARGS_SIZE + BOOT_EXTRA_ARGS_SIZE];
    };
    ```

    这个时候，AB 设备中的 recovery 分区被合并进了 boot 分区，也就没有必要再指定 recovery 所使用的 dtbo | acpio，原本属于 bootheader 中的内容的 dtb 被放置到了 vendor_boot 分区中，所以也没有必要再放置有关 dtb 的信息

    至于 vendor_boot ，是在 Android 11 中为兼容 GKI 而引入的新分区，所有 OEM 的修改都被分离到 vendor_boot 中，剩余部分使用 AOSP 通用内核的编译产物

    而 A only 设备则只能止步于此，因为使用单独的 recovery 分区和 boot 分区，所以它们只能使用到 bootheader v2

    到 Android 12 ， bootheader 更新至了 v4

    ```C
    struct boot_img_hdr
    {
    #define BOOT_MAGIC_SIZE 8
        uint8_t magic[BOOT_MAGIC_SIZE];

        uint32_t kernel_size;    /* size in bytes */
        uint32_t ramdisk_size;   /* size in bytes */

        uint32_t os_version;

        uint32_t header_size;    /* size of boot image header in bytes */
        uint32_t reserved[4];
        uint32_t header_version; /* offset remains constant for version check */

    #define BOOT_ARGS_SIZE 512
    #define BOOT_EXTRA_ARGS_SIZE 1024
        uint8_t cmdline[BOOT_ARGS_SIZE + BOOT_EXTRA_ARGS_SIZE];

        uint32_t signature_size; /* size in bytes */
    };
    ```

    新增了一个 ```uint32_t``` 类型的 ```boot_signature``` 字段，用于验证内核和 ramdisk 的完整性，但须注意的是，该字段并不参与 AVB 的启动过程而只由 VTS 测试使用

    bootheader v4 同时还支持了分段的 vendor_boot 分区

    关于 bootheader 的讨论到此结束，剩余部分请查阅 AOSP 项目文档

    ---
    ## 未完待续 | To be continued
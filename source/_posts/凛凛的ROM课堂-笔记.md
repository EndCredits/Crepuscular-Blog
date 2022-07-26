---
title: 凛凛的ROM课堂 -- 笔记
date: 2022-09-26 12:23:24
cover: https://avatars.githubusercontent.com/u/30337499?v=4
---

前一段 b 站直播凛凛没开麦... 斗胆加上一点点自己的理解，也记录一下自己做 rom 的一点点经验，算是为开源社区贡献一份自己微薄的力量

## 基本结构

最基础的文件有以下几个 

```vcs
cce87ff thyme: Initial tree from lisa
```

```text
ROOT
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

    同理，上面中的 ```BOARD_KERNEL_BASE``` 和 ```BOARD_KERNEL_PAGESIZE``` 也可以从 ```unpackbootimg``` 中的输出信息中得到

    事实上，像凛凛佬直播中使用的 ```magiskboot``` 程序会有更加良好的格式化以及更佳的可读性，同样也推荐使用

    对于 ```BOARD_KERNEL_SEPARATED_DTBO```，这个 flag 在有独立的 dtbo 分区的设备上是必须打开的，它控制编译系统是否支持以及编译单独的 dtbo 分区

    而 ```BOARD_MKBOOTIMG_ARGS``` 则是在生成 ```boot.img``` 时，传递给 ```mkbootimg``` 程序的参数，比如上述参数

    ```Makefile
    BOARD_MKBOOTIMG_ARGS += --header_version $(BOARD_BOOTIMG_HEADER_VERSION)
    ```
    意思就是给 ```mkbootimg``` 传入一个 --header_version 的参数，后面的是你的 bootheader version，已经在之前定义过了

    下面的 ```TARGET_KERNEL_ADDITIONAL_FLAGS``` 则是在编译内核 ```Image``` 的时候，要传递给 ```make``` 程序的额外参数，比如上文中给出的

    ```Makefile
    TARGET_KERNEL_ADDITIONAL_FLAGS := DTC_EXT=$(shell pwd)/prebuilts/misc/$(HOST_OS)-x86/dtc/dtc
    ```
    其中 ```pwd``` 指令是自己当前所在的目录，这样展开其实就是 ```$ANDROID_ROOT/prebuilts/misc/linux-x86/dtc/dtc``` 传入这一句的意思是向 ```make``` 程序说明，我们不希望使用内核源码中自带的 dtc ( device tree compiler | 内核设备树编译器 ) 编译内核的 dts，而是使用这个外部的预编译的 dtc ，这样做一方面是可以避免因为 dtc 版本不同而导致语法的不兼容，避免编译失败，也可以节约使用 HOSTCC 编译 dtc 的时间 ( 虽然说这个时间成本跟编译整个系统比起来要低很多就是了... )

    有的时候我们还需要用这个 flag 来传递一些信息，比如我们可以传入

    ```Makefile
    TARGET_KERNEL_ADDITIONAL_FLAGS += OBJDUMP=llvm-objdump
    ```
    这一句的意思是我们想要使用 LLVM 的 ```objdump``` 程序而不是用 ```gcc``` 的

    而最后面的三个 flag 就比较易懂了，它们的作用已经写在了它们的名字里

    内核部分结束了，然后是分区的定义

    ```Makefile
    # Metadata
    BOARD_USES_METADATA_PARTITION := true
    ```
    这个 flag 指定了你需不需要用 ```metadata``` 分区进行加密，现在采用 FBEv1/v2 的机器基本上都会使用这个分区，同样地 ```fastboot``` 程序返回的数据里也会有关于这个分区的信息，视情况打开它就好

    下面的是 oss vendor 所使用的分区信息
    ```Makefile
    # Partitions
    BOARD_BOOTIMAGE_PARTITION_SIZE := 134217728
    BOARD_CACHEIMAGE_PARTITION_SIZE := 402653184
    BOARD_DTBOIMG_PARTITION_SIZE := 33554432
    BOARD_RECOVERYIMAGE_PARTITION_SIZE := 134217728
    BOARD_USERDATAIMAGE_PARTITION_SIZE := 114919714816

    BOARD_SUPER_PARTITION_SIZE := 9126805504
    BOARD_SUPER_PARTITION_GROUPS := qti_dynamic_partitions
    BOARD_QTI_DYNAMIC_PARTITIONS_PARTITION_LIST := system system_ext product odm vendor
    BOARD_QTI_DYNAMIC_PARTITIONS_SIZE := 9122611200

    BOARD_FLASH_BLOCK_SIZE := 262144

    BOARD_CACHEIMAGE_FILE_SYSTEM_TYPE := ext4
    BOARD_ODMIMAGE_FILE_SYSTEM_TYPE := erofs
    BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE := erofs
    BOARD_SYSTEM_EXTIMAGE_FILE_SYSTEM_TYPE := erofs
    BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE := erofs
    BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE := erofs

    BOARD_EROFS_PCLUSTER_SIZE := 65536
    BOARD_EROFS_USE_ZTAILPACKING := true

    TARGET_COPY_OUT_ODM := odm
    TARGET_COPY_OUT_PRODUCT := product
    TARGET_COPY_OUT_SYSTEM_EXT := system_ext
    TARGET_COPY_OUT_VENDOR := vendor
    ```
    首先是第一组数据，定义了 ```boot, cache, dtbo, recovery, userdata``` 这几个分区的大小，这些属性同样也都会在你执行 ```fastboot getvar all``` 的时候悉数返回给你，只不过返回的值是 16 进制的，你需要用一些工具把它转换成 10 进制

    第二组中，```BOARD_SUPER_PARTITION_SIZE``` 定义了你的 super 分区的大小，获取方式同上，需要注意的是，只有在使用动态分区的设备上，第二块内容的定义才是必要的，否则如果没有使用动态分区，那么不需要定义 ```DYNAMIC_PARTITIONS``` 的相关属性

    特别地，如果你的设备使用了 bootheader v3/v4 并且拥有 ```vendor_boot``` 分区，那么你还需要定义 ```BOARD_VENDOR_BOOT_PARTITIONS_SIZE``` 这个 flag 来告诉编译系统 ```vendor_boot``` 分区的大小

    其中还有一个 flag ，```BOARD_QTI_DYNAMIC_PARTITIONS_PARTITION_LIST``` 定义了你的 super 分区中有哪些子分区，比如对于我们的 oss vendor 的编译需求，我们会需要去尽可能多的减少底包对我们的稳定性，所以我们需要将 super 分区里的全部内容都修改成我们需要的，也就是说我们编译我们需要的分区并且放进去，比如我们下面最后一组数据定义了我们会自行编译哪些分区的镜像，可以看到，有 ``` vendor product system_ext odm``` 当然还有最重要的 ```system``` ，所以我们把它们写入动态分区的列表中，告诉编译系统，生成 super 分区镜像的时候需要把哪些子分区的镜像一起打包进去

    而对于 prebuit vendor ，不需要编译那么多东西，通常只需要 ```system product system_ext``` 就可以了

    至于名字为什么是 ```QTI_DYNAMIC_PARTITIONS``` ，这都不要紧，可以自己定义一个，比如上面 ```BOARD_SUPER_PARTITION_GROUPS``` flag 定义的是 ```qti_dynamic_partitions``` ，我可以改成 ```picasso_dynamic_partitions```，只不过把所有的 ```QTI_DYNAMIC_PARTITIONS``` 都换成 ```PICASSO_DYNAMIC_PARTITIONS``` 就可以了

    而对于 ```BOARD_QTI_DYNAMIC_PARTITIONS_SIZE``` 的计算方法

    - Virtual AB:  ```BOARD_QTI_DYNAMIC_PARTITIONS_SIZE``` = ```BOARD_SUPER_PARTITION_SIZE``` - ```overhead```
    - AB: ```BOARD_QTI_DYNAMIC_PARTITIONS_SIZE``` = ```BOARD_SUPER_PARTITION_SIZE``` / 2  - ```overhead```
    - non-AB: ```BOARD_QTI_DYNAMIC_PARTITIONS_SIZE``` = ```BOARD_SUPER_PARTITION_SIZE``` - ```overhead```

    其中 overhead 通常等于 4MB ，注意减去该值时需要将 MB 转化为 byte 才可以相减

    下面的 ```BOARD_FLASH_BLOCK_SIZE``` 会定义在你的 boot.img 里，使用 ```unpackbootimg``` 工具可以看到这个值，它用于告诉 ```fastboot``` 它最大能一次刷入多少大小的东西，如果分区镜像文件的总大小超过了这个值，那就需要分步刷入，比如分成 7 次，每次刷入一点，这个值可以比 boot.img 中提取得到的值小，但绝对不能大，如果设置过大的话会导致无法刷入

    第四组 flags 则是定义了你需要用什么样的分区格式，与 fstab 中的保持一致就好，需要注意的是，使用 ```erofs``` 文件系统需要内核支持，如若不支持请不要用 ```erofs```

    最后一组就是定义了这些分区最终的编译产物的输出目录，copy 就好

    然后是一些杂项

    ```Makefile
    # Platform
    BOARD_USE_QCOM_HARDWARE := true
    TARGET_BOARD_PLATFORM := lito
    ```
    前一个比较明显，高通设备打开就好了，后面的那个跟你的 ```TARGET_BOOTLOADER_BOARD_NAME``` 保持一致就好了

    然后是 Recovery 相关的配置

    ```Makefile
    >>>>>>> picasso: Recovery Section
    # Recovery
    BOARD_INCLUDE_DTB_IN_BOOTIMG := true
    BOARD_INCLUDE_RECOVERY_DTBO := true
    TARGET_RECOVERY_FSTAB := $(DEVICE_PATH)/rootdir/etc/fstab.qcom
    TARGET_RECOVERY_PIXEL_FORMAT := "BGRA_8888"
    TARGET_USERIMAGES_USE_EXT4 := true
    TARGET_USERIMAGES_USE_F2FS := true
    ========
    # Recovery
    BOARD_INCLUDE_RECOVERY_DTBO := true
    BOARD_USES_RECOVERY_AS_BOOT := true
    TARGET_NO_RECOVERY := true
    TARGET_RECOVERY_FSTAB := $(DEVICE_PATH)/recovery/recovery.fstab
    TARGET_RECOVERY_PIXEL_FORMAT := "RGBX_8888"
    TARGET_USERIMAGES_USE_EXT4 := true
    TARGET_USERIMAGES_USE_F2FS := true
    <<<<<<< thyme: Recovery Section
    ```

    可以看到差别不是很大，而且 flag 的名字也比较易懂，就只介绍 non-AB 和 AB 有什么异同点

    对于 bootheader v2 及以上的具有独立 dtbo 分区的设备而言，```BOARD_INCLUDE_RECOVERY_DTBO``` 需要打开，这一点 ```picasso``` 是 v2，```thyme``` 是 v3，所以它们都启用了这个 flag，```BOARD_INCLUDE_DTB_IN_BOOTIMG``` 在 ```picasso``` 的配置中打开了，但是 ```thyme``` 没有打开，原因可以参照前文的 bootheader v3 发生的变化，它的 dtb 不再包含在 boot 分区中，而是移动到了 ```vendor_boot``` 分区里，所以 ```boot.img``` 里是不含 dtb 的，然后就是 ```BOARD_USES_RECOVERY_AS_BOOT``` 和 ```TARGET_NO_RECOVERY``` ，同样地，这个时候 AB 分区的手机不再有独立的 recovery 分区，而是合并进了 boot 分区，所以 ```thyme``` 打开了这两个 flag 而 ```picasso``` 没有

    下面介绍它们共有的 flags

    首先是 ```TARGET_RECOVERY_FSTAB``` ，它指定了你的 recovery 在挂载分区的时候应该使用哪个 fstab ，可以看到这里 ```picasso``` 直接复用了编译进系统的 fstab ，而 ```thyme``` 则是单独摘了一个出来，都可以

    然后是 ```TARGET_RECOVERY_PIXEL_FORMAT```，它描述了你的 recovery 使用什么样的像素格式，像凛凛视频里那样介绍的拿取这个 flag 的值就好，或者也可以直接 copy 别的，影响不大

    然后两个 flags 指定了你的 userdata 分区的格式，顾名思义即可

    Android Verified Boot 的话... 直接去看 AOSP 的文档就好了，看看每一项都有什么作用

    比较奇怪的一点是，在我翻源码的时候并没有找到传递给 ```avbtool make_vbmeta_image``` 的 ```--flags``` 标志位是什么含义，如果有人知道为什么要加上这个 ```--flags 3``` 的话请告诉我，十分感谢

    BoardConfig.mk 的内容肯定不止有这一点点，之后我们还会再加入，以完善这个设备树

- device.mk

    这个文件里主要定义了你需要编译进系统里的各种软件包，库文件，配置文件等等，你的大部分精力基本上都是花在这些东西上的，下面来分解一下 prebuilt vendor 的内容

    首先是 include 外部的 Makefile ，它们一般是这个格式

    ```Makefile
    # Installs gsi keys into ramdisk, to boot a GSI with verified boot.
    $(call inherit-product, $(SRC_TARGET_DIR)/product/developer_gsi_keys.mk)
    ```
    需要注意的是凛凛直播中说到的 Updatable APEX ( 不是 APEX Legend !!! ) 我们使用 oss vendor 一般都不会打开，因为 ROM 优化的 jemalloc 或者 mimalloc 都在这里面， update 了之后就没了，而且在一些设备上还会造成 ```bootloop```，所以这里不多提了，不过它是很好的虚拟化和容器化的例子，感兴趣的话可以学习一下

    当然说 prebuilt 必须要加上这个，毕竟它 vendor 自带的 apex 多半是我们不能用的，所以要 update 成我们自己的

    下面是 thyme 等 VAB 机型必须加入的内容

    ```Makefile
    # Virtual A/B
    ENABLE_VIRTUAL_AB := ture
    $(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota.mk)
    ```
    意思比较一目了然，表明你的设备支持并使用了 Virtual A/B 分区布局，如果不是 VAB 的话这里不需要加

    然后是 Dalvik VM 堆内存的配置
    
    ```
    # Setup dalvik vm configs
    $(call inherit-product, frameworks/native/build/phone-xhdpi-6144-dalvik-heap.mk)

    ```

    现代手机基本都是 6GB+ 了，所以直接用这个就好，然后的话对于老机型就去这个目录下找你对应的内存容量，加上就好了

    然后是这个标记设备出厂时的 Android 版本的 flag

    ```Makefile
    # Shipping API level
    PRODUCT_SHIPPING_API_LEVEL := 29
    ```

    是几就写几，特别重要不能写错，Android 版本与 API 版本的对应关系可以在 AOSP 文档里查到

    然后就是 AB 系统更新的内容

    ```Makefile
    # A/B
    AB_OTA_POSTINSTALL_CONFIG += \
        RUN_POSTINSTALL_system=true \
        POSTINSTALL_PATH_system=system/bin/otapreopt_script \
        FILESYSTEM_TYPE_system=ext4 \
        POSTINSTALL_OPITIONAL_system=true

    ```

    直接 copy 就好啦，不过要注意的是有的厂商会使用 erofs 或者别的文件系统 ，```FILESYSTEM_TYPE_system```要跟着一起变（ 最好 ) ，总之跟 fstab 中保持一致即可

    然后设定启动动画的分辨率
    ```Makefile
    # Boot animation
    TARGET_SCREEN_HEIGHT := 2400
    TARGET_SCREEN_WIDTH := 1080
    ```

    这就看你设备具体长宽是多少了，单位是像素

    下面定义了你的 init 脚本配置
    
    ```Makefile
    # Common init scripts
    PRODUCT_PACKAGES += \
        init.recovery.qcom.rc
    ```

    见凛凛 commit 里的内容，这是我们目前为止遇到的第一个 product package，也就是编译进你的设备的 "包" ，顺带提一嘴如何用 Android.mk 定义一个简单的模块，也就是所谓的 "包"

    >  File: $DEVICE_PATH/rootdir/Android.mk

    ```Makefile
    include $(CLEAR_VARS)
    LOCAL_MODULE       := init.recovery.qcom.rc
    LOCAL_MODULE_TAGS  := optional
    LOCAL_MODULE_CLASS := ETC
    LOCAL_SRC_FILES    := etc/init.recovery.qcom.rc
    LOCAL_MODULE_PATH  := $(TARGET_ROOT_OUT)
    include $(BUILD_PREBUILT)
    ```

    上面的字段中，首先是 include 了 ```$(CLEAR_VARS)```，用于清除之前一个模块的配置信息 ( 比如前一个模块定义的 ```LOCAL_MODULE``` )，以免影响到我们这个模块的编译，然后是 ```LOCAL_MODULE_CLASS``` 定义了你这个模块所属的类，比如这个模块属于 ETC 类，后面的 ```LOCAL_SRC_FILES``` 则是定义了这个文件的具体位置 (相对于 Android.mk 的路径)，后面的 ```LOCAL_MODULE_PATH``` 定义了最后你的这个文件会被拷贝到哪里去，比如上文的这个模块，```init.recovery.qcom.rc``` 就会被放置到 ```$(TARGET_ROOT_OUT)``` 这个位置，它是 AOSP 编译系统预先定义好的一组环境变量，用来告知编译系统你想把这个文件放到哪里，这里的意思就是放在 ```root``` 目录，其实对于 system-as-root 而言，也就是放进 system 里，以前的不使用 system-as-root 的机型可能会有 ramdisk，或者使用 2 Stage init 的设备也会有一个 ramdisk，最后就是 ```include $(BUILD_PREBUILT)``` ，它指明了你定义的这是个什么类型的文件，比如这个就是一个 prebuilt 文件，它决定了编译系统最后会怎么处置这个模块定义的源文件，它们也是 AOSP 编译系统预先定义的一组环境变量，可以去 build/make 里查看

    好的继续

    上面 Common init scripts 字段中， ```PRODUCT_PACKAGES``` 定义了你需要将哪些包编译进系统，需要注意的是，这时候为这个变量赋值的时候必须使用 ```+=``` 运算符而不能使用 ```:=```，使用后者会导致你原本的 ```PRODUCT_PACKAGES``` 被 override，这样很多软件包就丢了，毕竟这么大一个系统，不能只编译咱们定义的这几个 packages，所以上面那一句赋值的意思就是你想要多编译一个叫 ```init.recovery.qcom.rc``` 的包进入系统，加上这一句之后编译系统就会在它能访问的命名空间里找这个模块并且帮你编译进去，如果它找不到，它就会向你报错，告诉你这个模块它找不到，请君明鉴

    然后是 fastbootd 的内容，fastbootd 是用户空间内的 fastboot 程序，用于操作 super 分区内的子分区，比如 system, vendor, system_ext 等等

    ```Makefile
    # Fastbootd
    PRODUCT_PACKAGES += \
        fastbootd \
        android.hardware.fastboot@1.0-impl-mock
    ```

    之后的内容就比较简单了，只记录下 initial commit 的内容

    ```Makefile
    # F2FS utilities
    PRODUCT_PACKAGES += \
        sg_write_buffer \
        f2fs_io \
        check_f2fs \
    
    # Partitions
    PRODUCT_BUILD_SUPER_PARTITION := false
    PRODUCT_USE_DYNAMIC_PARTITIONS := true

    # Soong namespace
    PRODUCT_SOONG_NAMESPACE := \
        $(LOCAL_PATH)

    # Update engine
    PRODUCT_PACKAGES += \
        otapreopt_script \
        update_engine \
        update_engine_sideload \
        update_verifier
    
    PRODUCT_PACKAGES_DEBUG += \
        update_engine_client

    PRODUCT_HOST_PACKAGES += \
        brillo_update_payload
    ```

    需要注意的是 Update engine 只有 A/B 设备才会使用，Aonly 的设备不需要添加这些

    然后是为 ramdisk 添加 fstab.qcom

    ```Makefile
    # Vendor Boot
    PRODUCT_COPY_FILES += \
        $(LOCAL_PATH)/rootdir/etc/fstab.qcom:$(TARGET_COPY_OUT_VENDOR_RAMDISK)/first_stage_ramdisk/fstab.qcom
    ```

    其中有些分区需要在第一阶段 init 的时候挂载，不然的话后续的系统加载流程找不到这些分区，系统就无法启动，对于 bootheader v3/v4 使用 Virtual A/B 分区布局的设备来说，我们需要把它拷贝到 ```vendor_boot``` 里的第一阶段 init 读取的目录，对于 A/B 或者 non-A/B 设备，我们只需要把它拷贝到第一阶段 ramdisk 里就好，比如你可以定义一下下面的模块

    > $(DEVICE_PATH)/rootdir/Android.mk

    ```
    include $(CLEAR_VARS)
    LOCAL_MODULE       := fstab.qcom.ramdisk
    LOCAL_MODULE_STEM  := fstab.qcom
    LOCAL_MODULE_TAGS  := optional
    LOCAL_MODULE_CLASS := ETC
    LOCAL_SRC_FILES    := etc/fstab.qcom
    LOCAL_MODULE_PATH  := $(TARGET_RAMDISK_OUT)
    include $(BUILD_PREBUILT)
    ```

    然后在 device.mk 里选择编译它

    > $(DEVICE_PATH)/device.mk

    ```Makefile
    # Fstab
    PRODUCT_PACKAGES += \
        fstab.qcom.ramdisk
    ```

    后面凛凛直播的内容就是找文件了 (主要是 fstab) 和一个 init rc

    基本的目录结构到这里就介绍完了

## 后续工作

其实凛凛在直播里说的很正确，前人已经踩过的坑就没有必要自己去踩一遍了，对于 bring up 新设备而言也是如此，找一个硬件跟你的手机差异不大的已经有 device tree 的机型做参考无疑是最佳的选择，这样可以少走很多弯路，使用它们的 commits ，然后自己去试着编译，哪里出问题了修哪里，修到最后，你的 device tree 也就大成了，在这个修的过程中你也会收获很多知识，得到很多经验，干这一块知识储备固然重要，但是经验也是很重要的，很多问题大佬看一眼就明白是哪里出了问题，是因为这个问题他遇到过了，他知道怎么去修复，或者他的知识储备很丰富，推断出这个问题可以怎么解决，对于高通设备而言，Soc 型号不一样甚至都没什么大问题，用同一个 tag ，需要的流程就基本一致，拿 lito 和 kona 举例子吧，它们都使用 SMxx50 的 tag，甚至 lito 的设备 inherit kona 的 common tree 做一点点然后编译就能开机

## 私有 blobs

也就是凛凛直播中提到的第二个 commit

> ```3a51ade thyme: Import extract utils```

这个 commit 里提交了三个文件

```text
extract-files.sh
setup-makefiles.sh

proprietary-files.txt
```
这三个文件非常重要，它们协同工作，从厂商的 rom 里提取 blobs，生成了你的 vendor tree ，今后如果使用 oss vendor ，这三个文件更是重中之重

上面的 ```extract-files.sh``` 会读取 ```proprietary-files.txt``` 中的文件，然后从你厂商的 rom dump 里一个一个的把它们揪出来，放进 ```vendor/<manufacturer>/<devicecodename>``` ，为什么要从厂商的 dump 里提取呢，因为它们是私有的二进制文件，不能从 AOSP/CLO 源码里编译出来，所以我们必须使用它们来让一些 (实际上是几乎全部的) 硬件跑起来

对于 prebuilt vendor， ```proprietary-files.txt``` 里的内容比较好搞，就直接 copy 那些内容，至于 oss vendor，放在之后补充

凛凛的直播里说的很对的，不再赘述

## vbmeta 与 dtbo

接着就是下面一条 commit 的内容

> 61c5e31 thyme: releasetools: Ship and update vbmeta and dtbo images
> 76bfff7 thyme: releasetools: Add vbmeta_system to output zip

所谓 releasetools ，顾名思义，就是用来往你最终的那个 zip 里添加文件的一个 python 脚本，AOSP 编译系统默认不会把 dtbo 和 vbmeta 分区镜像添加进我们最终的 zip，所以我们需要自定义一下它来让它识别我们编译得到的这些镜像并把它们加入

同样地，如果想要在编译 rom 的时候顺便把 firmware 更新一并做进去的话，靠的也是这个文件

## SELinux

接下来你要面对的是整个过程里最让你头疼没有之一的东西，SELinux

Security-Enhanced Linux ，是 Linux MAC 权限控制框架下的一个基于标签的强制权限控制实现，由美国国家安全局 (NSA) 开发并开源，目的是为了增强 Linux 的安全性，限制程序对系统的修改，该组件自 Linux 2.6 被合并入 Linux 主线并于 Android 4.4 开始为 Android 提供全面保护

上面说的那么好，为什么令人头疼呢，那就是因为它的权限控制策略配置，真的超级超级麻烦...

> 2048f0e thyme: include QCOM Sepolicy

```Makefile
# Sepolicy
include device/qcom/sepolicy/SEPolicy.mk
```

Android 编译系统依赖 SELinux 上下文来确定每个文件的权限标签，不 include 它的话甚至编译都过不去，然后就是将 SELinux 设置为宽容模式

```Makefile
BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive
```
SELinux 会阻止所有未经策略配置允许的对系统资源的访问，如果让 SELinux 处于强制模式而你还没有配置好它的策略的话，它会阻止 Android 系统的正常启动，你是绝对开不了机的

开机之后就需要配置 sepolicy，以便于日后的 enforcing，所有未经允许的访问 SELinux 都会通过 ```logcat``` 或者 ```dmesg``` 日志来告诉你，它的格式一般是

```log
09-23 20:44:39.878 19355:19355 W android.hardware.fingerprint@2.1-service_picasso: type=1400 audit(0.0:410972): avc: denied { create } for name="gf_data" scontext=u:r:hal_fingerprint_default:s0 tcontext=u:object_r:system_data_root_file:s0 tclass=dir permissive=1
```
其中的 ```{}``` 里面的内容告诉了你是哪个操作被阻止了， ```name``` 的内容告诉你是什么东西发起了这个操作，```scontext``` 的内容告诉你发起这个操作的程序的 selinux 上下文标签是什么，比如上面的例子里它的标签就叫 ```hal_fingerprint_default```，然后 ```tcontext``` 的内容告诉你，被访问的目标的 selinux 上下文标签是什么，比如上面的例子里它是 ```system_data_root_file```，后面的 ```tclass``` 标识了被访问的内容是什么东西，在这个例子里它是 ```dir``` ，也就是个目录，后面的 ```permissive```标识当前 selinux 运行在什么模式下，比如这个例子里它运行在宽容模式下，那么 ```permissive``` 就是 1，如果它运行在 enforcing ( 强制 ) 模式下，那么它的值会变成 0

你会得到一大堆像这种形式一样的报错，你的任务就是一个一个的去把它们解决掉然后添加策略... 是不是有点太麻烦了？？？

不错，NSA 为我们提供了修复它的工具，```audit2allow```，详情可以直接查看 AOSP 文档，它会手把手的教你怎么用这个工具

建议从导入一些别的机型的基本的 sepolicy 开始，比如 commit:

> ```fb4d487 thyme: Initial sepolicy```

## VINTF

Vendor 接口对象，描述了你的设备中所有 HAL 的信息，以供编译时对 HAL 接口的检查以及运行时 ```getTransport``` 来找到各个 HAL 的接口入口 ( 当然如果它找不到的话就会报错你就开不了机XD )，同时它也是 CTS 测试的重要一部分

对了.. 凛凛直播的时候可能是太紧张口误说错了一个地方，就是 HAL 它是硬件抽象层 ( Hardware Abstraction Layer ) 而不是硬件叠加层 ( Hardware Overlay Layer )，这两个概念是不同的，感兴趣的同学可以自己去查一查维基

> ```2055f62 thyme: Import FCM from stock```

整个 VINTF 体现在你 oss device tree 里其实有三个文件

```
manifests.xml
compatibility_matrixes.xml
framework_compatibility_matrixes.xml
```

oss tree 稍后会再讲，嘛... 对于 prebuilt vendor 来说没那么麻烦，你就只需要一个 ```framework_compatibility_matrixes.xml``` 就足够了，这个文件是跟你 framework 里定义的 HAL 相关的，而对于你 device 自定义的 HAL 才需要上面的两个文件来描述它们，而这些 prebuilt vendor 都已经包含了，所以不需要

从扩展名就可以看出， VINTF 配置文件采用 xml 语法，一般对于 ```manifest.xml``` 中的项来说，它们一般都像这样

```
<hal format="hidl">
    <name>android.hardware.biometrics.fingerprint</name>
    <transport>hwbinder</transport>
    <version>2.1</version>
    <interface>
        <name>IBiometricsFingerprint</name>
        <instance>default</instance>
    </interface>
</hal>
```

- ```name``` 定义了一个 HAL 的名字
- ```transport``` 定义了该 HAL 使用什么方式进行通信，对于 Android 9 之后 binderlized 的 HAL 而言一般都是 ```hwbinder```，对于一些老的 HAL 它们通常是 ```passthrough```
- ```version``` 标记了这个 HAL 的版本
- ```interface``` 定义了这个 HAL 的接口，也就是 ```getTransport``` 需要的东西，这个接口可以有多个，比如下面 ```vendor fingerprint hal``` 的例子
- ```interface``` -> ```name``` 标记了一个接口的名称
- ```interface``` -> ```instance``` 标记了这个接口的实现，实现可以有多个

这里```name``` 和 ```instance``` 可以唯一确定一个接口，它们绝不会有重复

这是一个 ```Vendor Fingerprint HAL``` 的例子

```
<hal format="hidl">
    <name>vendor.goodix.hardware.cap.biometrics.fingerprint</name>
    <transport>hwbinder</transport>
    <version>2.1</version>
    <interface>
        <name>IGoodixFingerprintDaemon</name>
        <instance>default</instance>
    </interface>
    <interface>
        <name>IGoodixFingerprintDaemonExt</name>
        <instance>default</instance>
    </interface>
</hal>
```

可以看到 ```interface```可以有多个，它也可以这么写

```
<hal format="hidl">
    <name>vendor.goodix.hardware.cap.biometrics.fingerprint</name>
    <transport>hwbinder</transport>
    <fqname>@2.1::IGoodixFingerprintDaemon/default</fqname>
    <fqname>@2.1::IGoodixFingerprintDaemonExt/default</fqname>
</hal>
```
用 ```fqname``` 可以简化写法，甚至可以定义不同接口多个版本，像下面 ```Radio HAL``` 的写法

```
<hal format="hidl">
    <name>android.hardware.radio</name>
    <transport>hwbinder</transport>
    <fqname>@1.5::IRadio/slot1</fqname>
    <fqname>@1.5::IRadio/slot2</fqname>
    <fqname>@1.2::ISap/slot1</fqname>
    <fqname>@1.2::ISap/slot2</fqname>
</hal>
```

当然说具体应该写什么还得看系统给你报什么错，这个错误一般是由 ```check_vintf_list``` 在编译时报错以及 ```hwservicemanager``` 的  ```getTransport``` 在运行时报错，错一个修一个就可以

不过一般也不用你来写，可以从 CLO 直接拿或者从 dump 里提取，用 find + grep 找它就好了

## BootControl HAL

> ```77e9181 thyme: Import and Build bootctl from CAF```

Boot Control HAL 是使用 A/B 无缝系统更新的设备必须实现的 HAL ，它会接收来自系统 ```update_engine``` 的消息，选择启动时应该使用哪一个槽位，它实现了启动过程中的一个状态机，根据不同的情况进行不同的槽位选择

使用 A/B 无缝系统更新的设备 OEM 和 SoC 厂商必须确保 bootloader 已经按照要求实现了 bootcontrol hal 所需要的接口，这也是为什么 Google 的 boot control hal 我们没有办法直接用，因为这个跟 OEM 和 SoC 都有一定的关系，所以我们从 CAF ( 现在改名叫 CLO 了 ) 导入这个包含了高通特有修改的 bootctl 和 gpt-utils

这一条 commit 对于使用 A/B 系统更新的设备来说直接 pick 就好

## Build Flags

这些 Build 过程中起作用标志位主要是为了打开一些编译系统默认已经弃用的功能，不想那么麻烦的话一股脑打开就好

```Makefile
# Build
BUILD_BROKEN_DUP_RULES := true
BUILD_BROKEN_ELF_PREBUILT_PRODUCT_COPY_FILES := true
BUILD_BROKEN_VENDOR_PROPERTY_NAMESPACE := true
BUILD_BROKEN_MISSING_REQUIRED_MODULES := true
BUILD_BROKEN_ENFORCE_SYSPROP_OWNER := true
```
对于凛凛的那个 commit ，不加的话像这样的东西就会是 error 了

```log
build/make/core/Makefile:72: warning: overriding commands for target `/home/crepuscular/Working/home/crepuscular/development/aosp/out/target/product/picasso/vendor/lib/hw/audio.primary.lito.so'
build/make/core/base_rules.mk:533: warning: ignoring old commands for target `/home/crepuscular/Working/home/crepuscular/development/aosp/out/target/product/picasso/vendor/lib/hw/audio.primary.lito.so'
```

## IMS && Telephony

> ```e6d415f thyme: Compile IMS & Telephony packages```

这一条 commit 里的内容包含了编译 ```IMS``` 和 ```Telephony``` 功能的包，```IMS``` 通常跟 VOLTE 有关，Telephony 嘛，顾名思义，接打电话用的，高通设备 pick 这条 commit 就好，对于 MTK 设备的话，还需要 phh 的 IMS 补丁集

## Prebuilt odm

> ```adc57bd thyme: Use prebuilt odm from stock```

对于 prebuilt vendor 的设备需要原厂的 odm.img ，因为 vendor 里的东西会需要 odm 分区中的一些内容，如果我们编译的东西把它们覆盖掉了那就不好了，所以需要 prebuilt

像这条 commit 里的 ```THYME_PREBUILT``` flag 是你自己定义的变量，不用这个也没事，你也可以在 dt 里放一个 prebuilt 文件夹，把 ```odm.img``` 放进去，然后 ```BOARD_PREBUILT_ODMIMAGE``` 对应的改成你的路径就好了，比如

```Makefile
BOARD_PREBUILT_ODMIMAGE := $(LOCAL_PATH)/prebuilt/odm.img
```

对于上面的例子，之后就是记得在 ```AB_OTA_PARTITION``` 和 ```BOARD_THYME_DYNAMIC_PARTITIONS_PARTITION_LIST``` 里加上 odm 分区，让它一并被刷入就可以了

## Overlay

> ```d7f7f31 thyme: overlay: Import WifiRes overlay```

严格意义上讲这里所说的 ```overlay``` 应该是特指 ```Runtime Resource Overlay```，运行时资源叠加层，它的作用是在运行时改变某一个目标软件包的资源值，它可以覆盖原本源码里写死的值，替换成你想要的值，这对我们来说是非常有用的，因为一些功能的开启和关闭或者系统 UI 一些属性的定义 ( 比如挖孔样式 ) 都可以通过它来定义而没有必要每个设备都去修改一遍源码

至于 ```WifiResOverlay``` 顾名思义，就是应用到 WIFI 相关软件包的资源叠加层，它的作用就是改变默认编译的 WIFI 软件包里某一些资源的值，来看这个例子

> $(DEVICE_PATH)/overlay/WifiResOverlayThyme/AndroidManifest.xml

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.wifi.resources.thyme"
    android:verisonCode="1"
    android:versionName="1.0">
    <application android:hasCode="false" />
    <overlay android:targetPackage="com.android.wifi.resources"
                   android:targetName="WifiCustomization"
                   android:isStatic="true"
                   android:priority="0" />
</manifest>
```

接触过 Android 应用开发的同学应该对上面的那个文件很熟悉，它定义了一个 Android 软件包

- ```manifest->package``` 属性描述了这个 overlay 包的名字
- ```manifest->android:versionCode``` 属性描述了这个 overlay 包的版本
- ```manifest->android:versionName``` 属性描述了这个 overlay 包的版本名字

- ```overlay->android:targetPackage=``` 属性标示了这个 overlay 包要去覆盖哪个软件包 (包名) 的资源
- ```overlay->android:targetName``` 属性描述了你这个 overlay 的 target 是谁

重要的就是这些，其他的保持默认就好

然后就是写入你想要覆盖的资源标签和内容，这些可以通过逆向厂商的 overlay 来获取，或者如果你自己有什么想法也可以往这里面加，同样遵循 xml 语法，厂商的 overlay 通常放在

> ```/product/overlay/```

去那里面把它们揪出来，然后用 ```apktool d <包>``` 或者 jadx 解包都可以，它们的资源文件就会全部暴露出来，放到你的 overlay 对应的目录中，在上面的例子里，它是

> ```$(DEVICE_PATH)/overlay/WifiResOverlayThyme/res/values/```

最后就是声明这个包是一个 overlay 并且在 device.mk 里编译它

> ```$(DEVICE_PATH)/overlay/WifiResOverlayThyme/Android.bp```

```blueprint
runtime_resource_overlay {
    name: "WifiResOverlayThyme",
    theme: "WifiResOverlayThyme",
    certificate: "platform",
    sdk_version: "current",
    product_specific: true,
}
```

> ```$(DEVICE_PATH)/device.mk```

```Makefile
# Overlays
PRODUCT_PACKAGES += \
    WifiResOverlayThyme
```

加入其他的 overlay 方式也是一样的，有的时候一些 overlay 在你的厂商 overlay 里找不到的话可以去搜一下相近设备的 commit，如果有的话也可以直接采用

另一种一些人比较熟悉的写法是 ```Static Resource Overlay```，就是写在这种文件里的 overlay

> ```overlay/frameworks/base/core/res/res/values/config.xml```

这种静态资源叠加层不像 RRO 将会作为一个独立的 APK 编译进系统，而是会在编译时生效，去叠加源码中对应路径的资源值，最后你编译出来的这个源码组件就是你想要的样子，这有一个缺点就是对于那种大型组件比如 fwb 来说的话，整个重新来编译它耗时很长 ( 真的很长 )，而对于 RRO 来说，它只需要编译 RRO 的那个 APK 就可以了，好处是相对于 RRO 它的性能稍微好那么一点 ( 但也只有一点 ) ，相比之下如果性能不是那种非常非常吃紧的情况下用 RRO 会更加合算，而且这样也是 Google 推荐的做法

## Props

这些属性定义了 Android 中的一些特性是否开启或者关闭，对于 Prebuilt vendor 来说没有太麻烦，去 stock 的包里拿 build.prop 里的内容就好，而对于 oss vendor，因为自己魔改的自由度更大嘛，你就可以支持一些原本不支持的特性，关于每一条 prop 都有什么作用，可以直接去 Android Code Search 上搜索，源码中都有详细的注释

## Init Scripts 

有一点点 Linux 基础的人都会知道这是干什么用的，不再赘述

## Configs

> ```b6f9728: thyme: Import audio policy configuration from CAF```

这些配置文件是 ```AudioFlinger``` 或者 ```MediaProvider``` 或者 ```soundtrigger``` 等系统服务需要的，比如 picasso 有这些配置文件

> $(DEVICE_PATH)/audio

```text
audio_effects.xml
audio_io_policy.conf
audio_platform_info.xml
audio_platform_info_intcodec.xml
audio_platform_info_lagoon_qrd.xml
audio_platform_info_qrd.xml
audio_policy_configuration.xml
audio_policy_volumes.xml
audio_tuning_mixer.txt
mixer_paths.xml
mixer_paths_cdp.xml
mixer_paths_lagoonmtp.xml
mixer_paths_lagoonqrd.xml
mixer_paths_mtp.xml
mixer_paths_overlay_dynamic.xml
mixer_paths_overlay_static.xml
mixer_paths_qrd.xml
sound_trigger_mixer_paths.xml
sound_trigger_mixer_paths_cdp.xml
sound_trigger_mixer_paths_lagoonmtp.xml
sound_trigger_mixer_paths_lagoonqrd.xml
sound_trigger_mixer_paths_qrd.xml
sound_trigger_platform_info.xml
```

Audio 的配置文件通常用于配置 Audio 路由转发以及系统的各项音频相关策略，Media 通常包含了对于系统软硬件解码器以及其他媒体的细节的描述，这些都可以直接从 stock ROM 里拿，它们一般会被放置在

> ```/vendor/etc```

至于需要拿取哪些文件，对于 prebuilt vendor 而言其实一个 ```audio_policy_configuration.xml``` 一般就足够，剩下的使用 prebuilt vendor 里的，对于 oss vendor 而言，不如一股脑全放进去，至少绝对不会少不是嘛（）

## vendor overlay

> ```231cae0 thyme: Start using vendor overlay```

prebuilt vendor，顾名思义，预编译的 vendor，我们不能直接修改 vendor 里的内容，我来多提两嘴这个 commit 有什么作用

而 ```vendor overlay``` 允许你在设备启动时将变更叠加到 vendor 分区里，vendor overlay 是一系列放置在 ```product``` 分区的 vendor 模块 ( 比如各种运行库，可执行文件等 )，它们将在设备启动时替换或者添加 vendor 对应路径的文件

当设备启动时，```init``` 进程会完成第一阶段挂载并且读取默认的 props，然后 ```init``` 进程会读取 ```/product/vendor_overlay/<target_vendor_version>```，并且把该目录的子目录挂载到 ```vendor``` 分区对应的路径下

要使用 ```vendor overlay``` 时，需要满足下面的条件

- ```vendor``` 分区中存在要被叠加的目标路径
- ```/product/vendor_overlay/<target_vendor_version>``` 下要叠加到 ```/vendor``` 里的子目录 ( 或文件 ) 与 ```/vendor``` 对应的目录具有相同的 SELinux 文件上下文 ( file context )
- ```/vendor``` 下目标目录的 selinux 文件上下文 ( file context ) 允许 ```init``` 挂载

>! 注意，Vendor Overlay 可以替换，添加，但不可以删除 vendor 下的文件

>! 使用 Vendor Overlay 同时需要你的内核支持 overlayfs ，并且内核版本至少在 4.4 以上

如果要使用 vendor overlay 的话， pick 上面的那条更改并且往 

> ```/product/vendor_overlay/<target_vendor_version>``` 

下放置你想要 overlay 的文件就可以了，具体这条 commit 时怎么写出来的就是上面的三个条件，详情可以查阅 AOSP 的文档查看添加所用的示例

而 oss vendor 不需要 vendor overlay ，因为 oss vendor 里面一部分的文件是我们用 AOSP/CLO 源码编译进去的

## Fingerprint

可以考虑直接使用 prebuilt fingerprint hal ，或者对于屏下指纹可以使用 ```hardware/xiaomi``` 中的 common udsfps handler ，也就是修改过的 fingerprint hal 2.3，当然你可以自己来改，而侧边指纹通常是使用 fingerprint hal 2.1 ，导入之后自己修改即可，对于 sidefps 而言，通常是

```log
865a4e2 picasso: fingerprint: Add Xiaomi fingerprintextension support
f0086ab picasso: fingerprint: Don't set ro.boot.fpsensor
48bb127 picasso: Make fingerprint HIDL fully treble compliant
157da0c picasso: Import fingerprint HIDL
```
对于 udsfp 的话，我没有做过，不敢说多，但是关于这方面的例子很多，都可以参考

## Debug 调试

对于 AOSP Bring up 过程中的调试，通常我们会用到两个东西

### 1. logcat

这是 AOSP 为我们提供的日志查看工具，可以输出 Android 各层输出的日志，有不同的日志输出级别，方便筛选信息，它通常是这个样子的

```log
08-19 23:03:43.941  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.wm.ActivityTaskManagerService$Lifecycle took to complete: 2ms
08-19 23:03:43.941  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.am.ActivityManagerService$Lifecycle
08-19 23:03:43.941  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.am.ActivityManagerService$Lifecycle took to complete: 0ms
08-19 23:03:43.941  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.DataLoaderManagerService
08-19 23:03:43.941  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.DataLoaderManagerService took to complete: 0ms
08-19 23:03:43.941  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.PowerManagerService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.PowerManagerService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.ThermalManagerService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.ThermalManagerService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.hint.HintManagerService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.power.hint.HintManagerService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.recoverysystem.RecoverySystemService$Lifecycle
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.recoverysystem.RecoverySystemService$Lifecycle took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.lights.LightsService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.lights.LightsService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.display.DisplayManagerService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.display.DisplayManagerService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.verify.domain.DomainVerificationService
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.verify.domain.DomainVerificationService took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.PackageInstallerService$Lifecycle
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.PackageInstallerService$Lifecycle took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.UserManagerService$LifeCycle
08-19 23:03:43.942  1393  1730 V SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.pm.UserManagerService$LifeCycle took to complete: 0ms
08-19 23:03:43.942  1393  1730 D SystemServerTimingAsync: ssm.onUnlockedUser-0_com.android.server.om.OverlayManagerService
08-19 23:03:43.942  1393  1718 I AppWidgetServiceImpl: Processing of handleUserUnlocked u0 took 5 ms
```
它会提供给你很多信息

如果你少了什么库，它通常会以 linker 或者 init 报错告诉你，格式是

linker:
```
CANNOT LINK EXECTUABLE xxxx....
```

init:
```
dlopen failed
```

遇到这种情况你就可以去 Github 上搜索这个库，然后决定是从源码编译出这个库来，还是去 stock 的 ROM 里提取，还是从别的机型 kang

如果 HAL 因为 vintf 原因挂掉了，```hwservicemanager``` 会向你报告错误，格式是

```log
hwservicemanager: getTransport: Cannot find entry <hal name>@<version number>::<interface name>/<interface instance> in either framework or device VINTF manifest.
```

如果这个 HAL 压根都没有， ```hwservicemanager``` , ```libc``` , ```HidlServiceManagement``` , ```init``` 都会向你报错，格式是
```
09-29 08:47:05.673  3866  5249 W HidlServiceManagement: Waited one second for vendor.qti.data.factory@2.0::IFactory/default
09-29 08:47:05.673   608   608 I hwservicemanager: Since vendor.qti.data.factory@2.0::IFactory/default is not registered, trying to start it as a lazy HAL.
09-29 08:47:05.674  3866  5249 I HidlServiceManagement: getService: Trying again for vendor.qti.data.factory@2.0::IFactory/default...
09-29 08:47:05.676   608 20661 W libc    : Unable to set property "ctl.interface_start" to "vendor.qti.data.factory@2.0::IFactory/default": error code: 0x20
09-29 08:47:05.676   608 20661 I hwservicemanager: Tried to start vendor.qti.data.factory@2.0::IFactory/default as a lazy service, but was unable to. Usually this happens when a service is not installed, but if the service is intended to be used as a lazy service, then it may be configured incorrectly.
07-23 02:03:50.894     0     0 E init    : Control message: Could not find 'vendor.qti.data.factory@2.0::IFactory/default' for ctl.interface_start from pid: 608 (/system/bin/hwservicemanager)
```

如果系统里什么东西 crash 了，它会这样告诉你

```
09-14 12:03:02.155  7417  7417 F DEBUG   : *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
09-14 12:03:02.155  7417  7417 F DEBUG   : crDroid Version: '9.0'
09-14 12:03:02.155  7417  7417 F DEBUG   : Build fingerprint: 'Xiaomi/dipper/dipper:8.1.0/OPM1.171019.011/V9.5.5.0.OEAMIFA:user/release-keys'
09-14 12:03:02.155  7417  7417 F DEBUG   : Revision: '0'
09-14 12:03:02.155  7417  7417 F DEBUG   : ABI: 'arm64'
09-14 12:03:02.155  7417  7417 F DEBUG   : Timestamp: 2022-09-14 12:03:02.019211249+0800
09-14 12:03:02.155  7417  7417 F DEBUG   : Process uptime: 268s
09-14 12:03:02.155  7417  7417 F DEBUG   : Cmdline: com.coolapk.market
09-14 12:03:02.155  7417  7417 F DEBUG   : pid: 6936, tid: 6936, name: .coolapk.market  >>> com.coolapk.market <<<
09-14 12:03:02.155  7417  7417 F DEBUG   : uid: 10257
09-14 12:03:02.155  7417  7417 F DEBUG   : signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0000000000000028
09-14 12:03:02.155  7417  7417 F DEBUG   : Cause: null pointer dereference
09-14 12:03:02.155  7417  7417 F DEBUG   :     x0  0000000000000000  x1  000000784208a5e0  x2  0000000000000200  x3  4c4d410100000000
09-14 12:03:02.155  7417  7417 F DEBUG   :     x4  0000007840f56390  x5  0000000001414d4c  x6  0000000001414d4c  x7  0000007840f56004
09-14 12:03:02.156  7417  7417 F DEBUG   :     x8  0000000000000001  x9  0000000000001b18  x10 0000000000001b18  x11 0000000000000000
09-14 12:03:02.156  7417  7417 F DEBUG   :     x12 0037b99731b7b0b1  x13 0000000000000023  x14 0000007841f756d6  x15 000000000000000a
09-14 12:03:02.156  7417  7417 F DEBUG   :     x16 0000000000000001  x17 000000784205ad40  x18 00000075ac0f3c0c  x19 0000000000000000
09-14 12:03:02.156  7417  7417 F DEBUG   :     x20 0000007feb456204  x21 000000765c0a95d0  x22 0000007500516f73  x23 00000075ac0f89f0
09-14 12:03:02.156  7417  7417 F DEBUG   :     x24 0000007840c7f000  x25 0000007500559000  x26 0000007840c7f000  x27 0000007500516f73
09-14 12:03:02.156  7417  7417 F DEBUG   :     x28 0000000000000000  x29 000000762c0e3898
09-14 12:03:02.156  7417  7417 F DEBUG   :     lr  000000750048c9a0  sp  0000007feb456180  pc  000000750048c9b0  pst 0000000000000000
09-14 12:03:02.156  7417  7417 F DEBUG   : backtrace:
09-14 12:03:02.156  7417  7417 F DEBUG   :       #00 pc 00000000000af9b0  [anon:.bss]
```
有时候就需要你自己判断该怎么去修了

2. pstore

pstore 是内核中用于实现调试的一个子系统，ramoops 会把控制台日志转储到一段特定的内存区域里，这部分内存不会被修改，不会断电丢失数据，在正常内核启动时，这部分数据会被挂载到 ```/sys/fs/pstore``` 下，这样你就可以知道 crash 的时候到底发生了什么，比如下面这个很经典的例子

```log
[    0.543742] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000002
[    0.543746] Mem abort info:
[    0.543748]   ESR = 0x96000005
[    0.543750]   Exception class = DABT (current EL), IL = 32 bits
[    0.543752]   SET = 0, FnV = 0
[    0.543754]   EA = 0, S1PTW = 0
[    0.543756] Data abort info:
[    0.543758]   ISV = 0, ISS = 0x00000005
[    0.543761]   CM = 0, WnR = 0
[    0.543763] [0000000000000002] user address but active_mm is swapper
[    0.543767] Internal error: Oops: 96000005 [#1] PREEMPT SMP
[    0.543770] Modules linked in:
[    0.543774] Process swapper/0 (pid: 1, stack limit = 0x00000000a6b62718)
[    0.543778] CPU: 7 PID: 1 Comm: swapper/0 Tainted: G S                4.19.259-Driftwood #5
[    0.543780] Hardware name: Qualcomm Technologies, Inc. 7250 picasso (DT)
[    0.543783] pstate: 20c00005 (nzCv daif +PAN +UAO)
[    0.543795] pc : inode_permission2+0x20/0x180
[    0.543798] lr : lookup_one_len_common+0x10c/0x140
[    0.543801] sp : ffffff800805b750
[    0.543802] x29: ffffff800805b750 x28: 0000000000000078 
[    0.543806] x27: ffffff98f2c5b1b8 x26: dead000000000200 
[    0.543808] x25: fffffff7b687cc10 x24: ffffff98f36f2f10 
[    0.543810] x23: fffffff7b6adf420 x22: ffffff98f36f2f10 
[    0.543812] x21: fffffff7b6adf490 x20: 0000000000000000 
[    0.543815] x19: 0000000000000001 x18: 00000000000001c2 
[    0.543817] x17: 000000000000ba7e x16: 00000000000000e4 
[    0.543819] x15: 000000000000ffff x14: 000074756f307065 
[    0.543822] x13: 00000000fffffffe x12: ffffffffffffffff 
[    0.543824] x11: fffffff79949a270 x10: 000074756f307065 
[    0.543827] x9 : df6191354b2a79c7 x8 : 0000000000000000 
[    0.543830] x7 : 6e2f6f64fefefefe x6 : 0000808080808080 
[    0.543832] x5 : 0000000000000000 x4 : ffffff800805b7a8 
[    0.543835] x3 : 0000000000000000 x2 : 0000000000000001 
[    0.543837] x1 : 0000000000000000 x0 : 0000000000000000 
[    0.543840] Call trace:
[    0.543843]  inode_permission2+0x20/0x180
[    0.543846]  lookup_one_len_common+0x10c/0x140
[    0.543849]  lookup_one_len_unlocked+0x38/0x100
[    0.543854]  debugfs_lookup+0x44/0x70
[    0.543865]  dwc3_gadget_free_endpoints+0x2c/0xd0
[    0.543868]  dwc3_gadget_exit+0x28/0x140
[    0.543872]  dwc3_remove+0x24/0x100
[    0.543876]  platform_drv_remove+0x24/0x50
[    0.543879]  device_release_driver_internal+0x16c/0x230
[    0.543881]  device_release_driver+0x14/0x20
[    0.543885]  bus_remove_device+0xd0/0x100
[    0.543888]  device_del+0x288/0x580
[    0.543890]  platform_device_unregister+0x2c/0xb0
[    0.543895]  of_platform_device_destroy+0xb4/0xd0
[    0.543897]  device_for_each_child+0x54/0xb0
[    0.543900]  of_platform_depopulate+0x30/0x60
[    0.543902]  dwc3_msm_probe+0xe10/0xf10
[    0.543905]  platform_drv_probe+0x7c/0xc0
[    0.543907]  really_probe+0x274/0x300
[    0.543909]  driver_probe_device+0x60/0x100
[    0.543911]  __driver_attach+0xcc/0x110
[    0.543915]  bus_for_each_dev+0x78/0xc0
[    0.543917]  driver_attach+0x20/0x30
[    0.543920]  bus_add_driver+0x11c/0x200
[    0.543923]  driver_register+0x74/0x110
[    0.543926]  __platform_driver_register+0x40/0x50
[    0.543928]  dwc3_msm_init+0x18/0x20
[    0.543932]  do_one_initcall+0x118/0x280
[    0.543937]  do_initcall_level+0x144/0x16c
[    0.543939]  do_basic_setup+0x30/0x48
[    0.543941]  kernel_init_freeable+0xc4/0x144
[    0.543945]  kernel_init+0x14/0x290
[    0.543947]  ret_from_fork+0x10/0x20
[    0.543951] Code: 2a0203f3 aa0103f4 aa0003e8 37080202 (79400689) 
[    0.543954] ---[ end trace f9c0eeb6e53bd0e4 ]---
[    0.543971] Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
[    0.543971] 
[    0.543976] SMP: stopping secondary CPUs
[    0.543986] devfreq_panic_callback: L3-DOMAIN
[    0.543991]        PERF_STATE_DESIRED: 0x0000000b
[    0.543994]             PSTATE_STATUS: 0xc44f03c4
[    1.144083] ipa ipa3_active_clients_panic_notifier:305 
[    1.144083] ---- Active Clients Table ----
[    1.144083] 
[    1.144083] Total active clients count: 0
[    1.144083] 
[    1.144090] Kernel Offset: 0x18e8800000 from 0xffffff8008000000
[    1.144093] CPU features: 0x0000000c,a2802218
[    1.144095] Memory Limit: none
[    1.144099] Rebooting in 5 seconds..
[    6.144528] SMP: stopping secondary CPUs
[    6.144533] Going down for restart now
```

不过 prebuilt kernel 通常不会有这个，有的话基本就别想适配了（，所以这个主要是给 oss kernel 用的调试手段，在知道了是哪个函数 crash 之后，可以通过编译时生成的 ```System.map``` 来查询函数的具体地址，然后通过 ```addr2line``` 来查看到底是内核的哪一行出了问题

寄存器信息也是很有用的，在 arm64 架构下，比较重要的寄存器有三个

- pc 寄存器: 即程序计数 ( Program Counter ) 寄存器，这个寄存器保存着 CPU 正在执行的指令的地址，addr2line 可以来找出这个地址对应了哪个函数，用于定位问题所在
- sp 寄存器: 即栈指针 ( Stack Pointer ) 寄存器，这个寄存器始终指向当前函数调用栈的栈顶，在 arm64 中, x0-x30 为通用寄存器，比如上面的日志，当前 x29 寄存器指向函数调用栈的栈顶，sp 寄存器的指向跟它一样
- lr 寄存器: 即连接 ( Linker ) 寄存器，存储了当前程序跳转后，原程序的下一条指令的地址

## oss vendor

说了这么久终于到了 oss vendor 了，oss vendor 相较于 prebuilt vendor 的区别就是它里面的内容有一些是厂商私有的 blobs，还有一部分是我们自己从 AOSP/CLO 编译的，因为 oss vendor 刷入时不像prebuilt vendor 不会修改原始 stock ，而是替换了整个 vendor 成我们自己的，这样做的好处是增加了我们的灵活性，可以灵活调整里面的 blobs，可以更新 HAL，可以更新二进制文件，可以适配新的 HAL，缺点就是十分麻烦，因为里面终究还是有私有的组件来确保你的设备能够正常运行，这些组件什么该加，什么不该加，HAL 需要编译什么，不需要编译什么，都由你来掌控，比较难以把握

### 再谈 proprietary-files.txt

前文中我们说到了这个文件的作用，我们发现，一些设备在 prebuilt vendor 下，这个文件不过几百行最多，但是一旦切换到了 oss vendor，这个文件马上就会有 1500 多行，这是为什么呢

因为我们编译的 HAL 和二进制文件需要它们的支持，AOSP 源码编译出来的东西并不能满足我们设备的需要，这就需要厂商 blobs 出马，提供对我们设备的兼容性

这里可以提供一点小小的经验关于这里到底应该添加什么东西以及从哪里找

记得善用 ```find``` 和 ```grep``` 命令

1. ACDB Data

通常被放置在 

> ```vendor/etc/acdbdata/``` 

下，用于承载 ADSP 服务，通常需要全部加入，有啥要啥，这一块主要管通话以及音频路由

2. ACDB Loader

ACDB 加载器，用于承载 ADSP 数据文件，它们通常有以下内容，以 ```libacdb``` 开头

```text
vendor/lib/libacdb-fts.so
vendor/lib/libacdbloader.so
vendor/lib/libacdbrtac.so
vendor/lib64/libacdb-fts.so
vendor/lib64/libacdbloader.so
vendor/lib64/libacdbrtac.so
```

3. ADSP 

高通 ADSP ( Audio DSP ) 服务，我这里就将它与 CDSP ( Compute DSP ) 分开了，它们通常具有以下文件名和路径特征

```
vendor/bin -> { adsprpc | dspservice | loadalgo}
vendor/etc/init -> { adscrpc-service.rc | hardware.dsp }
vendor/etc/seccomp_policy -> { hardware.dsp }
vendor/lib -> hardware.dsp
vendor/lib64 -> { hardware.dsp | libadsp | libloadalgo }
```
用 find 找到它们，加入 proprietary-files 中就好了

4. ADSP Modules

高通 ADSP 服务模块，全部位于

```
vendor/lib/rfsa/adsp
vendor/lib64/rfsa/adsp
```
这两个文件夹里所有的库全部加入

5. Alarm

关机 Alarm 驱动及其配置文件，位于

```
/product/app/PowerOffAlarm
/system_ext/framework/
/vendor/bin
/vendor/bin/hw
/vendor/etc/init
/vendor/lib64
/vendor/lib64/hw
```

通常对应的 apk 叫做 PowerOffAlarm.apk，对应的二进制可执行文件叫做 power_off_alarm，其余的 Blobs 名字里都带 ```hardware.alarm``` 字段，find + grep 找它们就好

6. ANT+

ANT+ Wireless Service 驱动，
# Introduction

The “Piano Tutor” is an embedded educational system that pairs a 25‑note piano keyboard with a visual and tactile interface to guide the learner in real time. Key components include:

Base board: LCPI‑PC‑F133‑T113‑D1S‑V1.3 (Allwinner T113‑S3, dual‑core Cortex‑A7).

5×5 key and LED matrices: shift registers (74HC595 and 74HC165) reduce required GPIO from 50 down to just 12 pins.

OLED display (SSD1306) driven via I²C bit‑banging to show menus, lessons, and feedback.

Physical buttons for switching modes (sequential lessons, free play, guided tutorial).

Firmware: boots with Buildroot + Debian Bookworm, scans the key matrix, lights LEDs to indicate the next note, and updates the on‑screen UI.

# Potential Applications

- Music Education
  	- Interactive piano lessons without requiring an in‑person teacher.

	- Sight‑reading exercises and ear‑training drills.

- Entertainment & Art Installations

	- Interactive exhibits where visitors “play” guided musical pieces.

- DIY & Maker Projects

	- A platform for extensions: integrate with modular synthesizers, MIDI control, or IoT.

# Embedded System
This part of the piano project follows the instructions provided by Carlos Iván Camargo Bareño, our instructor for the semester. He shared a book and a guide detailing how to set up the embedded system on our board. The text is available in the `Resources` folder for further reference.

We chose the LCPI-PC-F133-T113-D1S-V1.3 development board, available at AliExpress.

This board features a dual-core ARM Cortex-A7 CPU, powered by the Allwinner T113-S3 SOC.

1. [Buildroot](#buildroot)
    1. [Clone the repository](#1-clone-the-repository)
    2. [Kernel configuration](#2-kernel-configuration)
    3. [Toolchain](#3-toolchain)
    4. [U-boot setup](#4-u-boot-setup)
    5. [Creating the SD](#5-creating-the-sd)
    6. [Creating image](#6-creating-image)
    7. [Debian](#7-debian)
    8. [Enjoy](#8-enjoy)
2. [Pins and Peripherals](#pins-and-peripherals)
3. [Device Tree](#device-tree)
    1. [The .dtsi File](#the-dtsi-file)
    2. [The .dts File](#the-dts-file)
4. [Rebuilding](#rebuilding)


## Buildroot
Buildroot is a tool that helps in the construction of embedded systems, similar to Yocto, OpenWRT, and other build systems. We are using Buildroot to create the system image. Below is the step-by-step guide.

### 1. Clone the repository
First, you will need to run this commands:

```
git clone https://github.com/yuzukihd/Buildroot-YuzukiSBC
cd Buildroot-YuzukiSBC
source envsetup.sh
lunch
make mangopi_mq_dual_defconfig
make -j10 # Adjust according your machine, running the nproc command on the terminal helps
```
You will encounter three errors during this process. While running `make`, it will attempt to download some files that are no longer available at the specified URLs. Here's how to resolve them:

1. Within the `Resources` folder you have `v1.0.1.tar.gz`. You should move this file to `buildroot/dl/uboot/`.
2. Within the `Resources` folder you have `1.0.0.tar.gz`. You should move this file to `buildroot/dl/linux`.
3. Within the `Resources` folder you have `mkbootimg`. You should move this file to the folder of binaries of your computer, you can use `cd /bin/` and copy the file.
4. Other errors may advise you to install different packages, they are usually available with `apt-get install`.

### 2. Kernel configuration
The last step was executed using the default configuration. Now, we need to modify the configuration file to add support for the specific devices we want to use. Run the following command to open the menu configuration:
```
make linux-menuconfig
```
Now, make sure to enable (or disable) the following nodes:
```
Device Drivers
    GPIO Support --->
        <*> /sys/class/gpio/
        <*> Debug GPIO calls

Device Drivers
    Network device support --->
        PHY Device support and infrastructure --->
            < > RealTek RTL-8363NB Ethernet Adapter support

Device Drivers
    I2C support --->
        I2C Hardware Bus Support --->
            <*> GPIO-based bitbanging I2C

Device Drivers
    Graphics support --->
        Frame buffer Devices --->
            <*> Support for frame buffer devices --->
                [*] Simple framebuffer support
                <*> Solomon SSD1307 framebuffer support

File systems
    <*> Old Kconfig name for Kernel automounter support
    <*> Kernel automounter support (supports v3, v4 and v5)
    < > SquashFS 4.0 - Squashed file system support

General setup
    <*> Control Group support
    [*] Support for paging of anonymous memory (swap)
```
You can now rebuild your image with the following commands
```
make -j12 linux-rebuild
make -j12
```

### 3. Toolchain 
These tools will help us in the compilation proccess, the `TOOLCHAIN` folder is available at `Resources`.
Just make sure to add the this folder to the path with `export PATH=$HOME/TOOLCHAIN/gcc-linaro-7.2.1-2017.11-x86_64_arm-linux-gnueabi/bin:$PATH`

### 4. U-boot setup
Within the `Resources` folder you also have the `v1.0.1.tar.gz` folder. You need to unpack that and patch the instructions with some files available at `patch_uboot`, also available in `Resources`. This way:

```
tar zxvf v1.0.1.tar.gz
cd u-boot-2018-1.0.1

patch -p1 < ../patch_uboot/0001-add-support-for-buildroot.patch
patch -p1 < ../patch_uboot/0002-fix-uboot-disable-dtc-selfbuilt.patch
patch -p1 < ../patch_uboot/0003-fix-uboot-support-for-buildroot-dts-file.patch
patch -p1 < ../patch_uboot/0004-fix-No-rule-to-make-target-sunxi_challenge.patch
patch -p1 < ../patch_uboot/0005-fix-yylloc.patch

cp ../patch_uboot/uboot_mangopi_mq_dual_defconfig configs/
cp ../patch_uboot/sun8i-mangopi-mq-dual-uboot.dts ./arch/arm/dts/
cp ../patch_uboot/sun8iw20p1-soc-system.dtsi arch/arm/dts/
```
You’ve just copied a `.dts` and a `.dtsi` file. These are **not** the device tree files you will modify for future tests involving pins in your project. They are basic device tree files used to initialize essential hardware and boot the root filesystem.

For pin configuration and other hardware customizations, you’ll need to locate and edit the appropriate board-specific device tree files later in the process. We will check that in a second.

Now, you need open the menu configuration:

```
make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm uboot_mangopi_mq_dual_defconfig
make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm menuconfig
```
and make sure to enable the following options
```
Command line interface --->
    Boot commands --->
        [*] bootz
    [*] go

Device Tree Control --->
    Provider of DTB for DT control --->
        [*] Provided by the board at runtime
```
And make the instructions:
```
make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm
```
At the end, you will have a file named `u-boot-sun8iw20p1.bin`. Please copy this file to `your-path/Buildroot-YuzukiSBC/buildroot/output/images`. We will need that file in the future;)

### 5. Creating the SD
Finally, we are going to create the partitions and add the necessary files to our SD card to make everything work.

According to the image:
![buildroot_table](/embedded-systems-development/Images/buildroot_table.png)

* `boot_package.fex` includes `u-boot-sun8iw20p1.bin` and `sun8i-mangopi-mq-dual-linux.dtb`. These are essential for the initial boot stage of the system.
* `boot.vfat` includes `boot.img`, `zImage`, and `sun8i-mangopi-mq-dual-linux.dtb`. These files contain the kernel, device tree, and bootloader required to start the operating system.

First, we have to copy the `boot_package.fex` on the requiered location, please run the following commands:
```
cd your-path/Buildroot-YuzukiSBC/buildroot/output/images
./dragonsecboot -pack boot_package.cfg
sudo dd if=boot_package.fex of=/dev/sdX(CHANGE) bs=1k seek=16400
```
We have just completed the initial boot. Now, we need to ensure that the partitions are correctly created on the SD card and have the appropriate sizes.
```
cd your-path/Buildroot-YuzukiSBC/buildroot/output/images
sudo dd if=sdcard.img of=/dev/sdX(CHANGE)

sudo fdisk /dev/sdX(CHANGE)

Command (m for help): p

Device          Start   End         Sectors     Size    Type
/dev/sda1       35360   36383       1024        512K    Linux filesystem
/dev/sda2       36384   36639       256         128K    Linux filesystem
/dev/sda3       36640   36895       256         128K    Linux filesystem
/dev/sda4       36896   102431      65536       32M     Linux filesystem
/dev/sda5       104448  62333918    62229471    29,7G   Linux filesystem
    
Borrar partición: d
Crear nueva partición n
Gardar cambios: w
```
These steps resize the fifth partition. Now, we just need to format it with the correct filesystem.

```
sudo mkfs.ext4 /dev/sdX5(CHANGE)
```
**PLEASE UNPLUG AND PLUG ONCE AGAIN THE SD CARD TO MAKE SURE THE CHANGES ARE SAVED**

### 6. Creating image
Now, we just need to add the image to our SD Card with the following commands:
```
cd your-path/Buildroot-YuzukiSBC/buildroot/output/images
mkbootimg --kernel zImage --output boot.img
sudo cp zImage boot.img sun8i-mangopi-mq-dual-linux.dtb /media/USER/FOURTH-PARTITION/ # That fourth partition is usually something with 8 characters like 12B7-6A23 in my case
```

### 7. Debian
The last step is to load an OS to the SD Card.
Within the `Resources` folder you have `debian_bookworm`. You just need to copy the files into the fifth partition, the one we just resized.
```
cd your-path/debian_bookworm
sudo cp -avr * /media/USER/FIFTH-PARTITION/  # That fifth partition is usually something with a lot of characters like d4ab1e98-2104-4b8f-8526-55a987f8b082 in my case
```
For further details on this process, the guidebook contains a step-by-step guide on how to download and set up the OS.

### 8. Enjoy
Now you're all set! You can communicate with your board from your computer over the serial connection using Minicom. :D

## Pins and Peripherals
As mentioned at the beginning, this project aims to be a tutor for piano learners. To accomplish that, we need to have three main things:

* A piano keyboard that sounds
* A system to guide the user on which key to press next
* A menu interface for the user to interact with the device

We proposed the construction of a daughter board, a shield-like board, with the necessary elements to achieve our three specifications. More details on this process are provided in another section of this repo, `electronic-system-design`, which includes files, key points of the design, the methodology used, and the physical construction of the piano.

However, it is important to mention a couple of things here regarding the pinout.

This board has 36 pins, some of which are part of the PB, PE, and PG banks of pins. More information on that can be found in the documentation included in  [this documentation](https://drive.google.com/drive/folders/1lrqDsxtGl8WvU7o547lT9IkHwGyAHXFU?spm=a2g0o.detail.1000023.1.119bNxvmNxvmuI)

We used MIDI controllers like the MINILAB3, AKAI MPK Mini, and OP-1 from Teenage Engineering as references for our piano. They have 25 keys from C4 to C6.

For the tutor, we proposed one LED per key to instruct the player on "the next key to be pressed."

That is, 25 keys with 25 LEDs. The pins are limited on the board, so we have to reserve some for serial communication, the I2C screen, and some buttons.

The solution is to set up a 5x5 matrix for the keys and a 5x5 matrix for the LEDs, controlled by shift registers. This way, we reduce the pin count from 10 pins for each matrix (20 pins total for keys and LEDs) to just 12 pins, like this:

* Keys: the **74HC595** will activate the columns of the matrix, while the **74HC165** will read the values.
* LEDs: one **74HC595** will activate the columns of the matrix, and the other **74HC595** will activate the rows.

Each shift register uses three pins, and we have four integrated circuits, so that’s 12 pins for the matrices.

The screen, initially proposed to be SPI, was changed to I2C with bit-banging, and 3 buttons are used for the user interface.

Those are the pins used for this project. Below, you will find some images for further reference.

### Keys Matrix
![keys_matrix](/embedded-systems-development/Images/keys_matrix.png)
![keys_shifts](/embedded-systems-development/Images/keys_shifts.png)

### LEDs Matrix
![leds_matrix](/embedded-systems-development/Images/leds_matrix.png)
![leds_shifts](/embedded-systems-development/Images/leds_shifts.png)

### Screen and Buttons
![screen_menu](/embedded-systems-development/Images/screen_menu.png)
![buttons_menu](/embedded-systems-development/Images/buttons_menu.png)

### Dev Board Pinout
![dev_board_pinout](/embedded-systems-development/Images/dev_board_pinout.png)

## Device Tree
This is how we set up the pins to interact with our kernel and the OS. Here, we define what each pin is used for and which drivers they should include. This is essential when customizing an image to ensure the pins are used correctly.

Keeping the pinout in mind, we configure the pins connected to a shift register or to a button for GPIO. The 3 pins for the screen are also GPIO but are used with I2C bit-banging.

### The .dtsi File
This is a general configuration. Here, we define the pins for the matrices, screen, buttons, and UART. You can see that all of them have `status = "disabled";`. This is for convention; they will be enabled in the next important file, the `.dts`.

You can find this file in `your-path/Buildroot-YuzukiSBC/buildroot/board/allwinner-generic/sun8i-t113/dts/linux/sun8iw20p1-linux.dtsi`

These are the nodes:

#### UART
For serial communication.
```
/ {
    soc: soc@3000000 {
        uart0: uart@2500000 {
            compatible = "allwinner,sun8i-uart";
            device_type = "uart0";
            reg = <0x0 0x02500000 0x0 0x400>;
            interrupts = <GIC_SPI 2 IRQ_TYPE_LEVEL_HIGH>;
            sunxi,uart-fifosize = <64>;
            clocks = <&ccu CLK_BUS_UART0>;
            clock-names = "uart0";
            resets = <&ccu RST_BUS_UART0>;
            uart0_port = <0>;
            uart0_type = <2>;
            status = "disabled";
        };
    }
}
```

#### Keys Matrix, LEDs Matrix, and Buttons
Here, we set the pins to `gpio_in`, making each pin a GPIO port that we can configure via software to send or receive data.
```
/ {
    soc: soc@3000000 {
        pio: pinctrl@2000000 {
            gpio_piano_pins: gpio_piano_pins@0 {
                pins = "PE0", "PE1", "PE4", "PE5", "PE6", "PB7",
                    "PG0", "PG1", "PG2", "PG3", "PG4", "PG5",
                    "PB4", "PB5", "PB6";
                function = "gpio_in";
                drive-strength = <10>;
                bias-disable;
            };
        }
    }
}
```

#### I2C Screen
We define the clock and data pins for the screen. Additionally, we configure some settings of our OLED display, such as width, height, and the driver `ssd1306fb-i2c`. This driver was included in the kernel configuration.
```
/ {
    i2c_gpio: i2c-gpio {
        compatible = "i2c-gpio";
        gpios = <&pio 4 10 GPIO_ACTIVE_HIGH /* SDA = PE10 */
                    &pio 4 11 GPIO_ACTIVE_HIGH>; /* SCL = PE11 */
        i2c-gpio,delay-us = <5>;
        #address-cells = <1>;
        #size-cells = <0>;
        status = "disabled";

            oled_display: oled@3c {
                compatible = "solomon,ssd1306fb-i2c";
                reg = <0x3c>;
                solomon,width = <128>;
                solomon,height = <64>;
                solomon,page-offset = <0>;
                solomon,com-offset = <0>;
                solomon,segment-offset = <0>;
                status = "okay";
            };
    };
}
```

### The .dts File
This is a more specific file with the header `#include "sun8iw20p1-linux.dtsi"`, meaning that includes our previous nodes configuration. Here we enable our devices along with the pins.

You can find this file in `your-path/Buildroot-YuzukiSBC/buildroot/board/mangopi/mq-dual/dts/linux/sun8i-mangopi-mq-dual-linux.dts`

#### UART
```
&uart0 {
	pinctrl-names = "default", "sleep";
	pinctrl-0 = <&uart0_pins_a>;
	pinctrl-1 = <&uart0_pins_b>;
	status = "okay";
};
```

#### Keys Matrix, LEDs Matrix, and Buttons
```
&pio {
    gpio_piano_pins: gpio_piano_pins@0 {
        allwinner,pins = "PE0", "PE1", "PE4", "PE5", "PE6", "PB7",
                "PG0", "PG1", "PG2", "PG3", "PG4", "PG5",
                "PB4", "PB5", "PB6";
        allwinner,function = "gpio_in";
        allwinner,muxsel = <0>;
        allwinner,drive = <1>;
        allwinner,pull = <0>;
    };
}
```

#### I2C Screen
```
&i2c_gpio {
    status = "okay";
};
```

## Rebuilding 
After making any changes within your device tree, you should:

1. Rebuild your image
2. Copy `boot_package.fex` to the appropriate location
3. Copy `boot.img` and `sun8i-mangopi-mq-dual-linux.dtb` to the appropriate location

```
## 1.
make -j10 linux-rebuild
make -j10

## 2.
cd your-path/Buildroot-YuzukiSBC/buildroot/output/images
./dragonsecboot -pack boot_package.cfg
sudo dd if=boot_package.fex of=/dev/sdX(CHANGE) bs=1k seek=16400

## 3.
cd your-path/Buildroot-YuzukiSBC/buildroot/output/images
mkbootimg --kernel zImage --output boot.img
sudo cp zImage boot.img sun8i-mangopi-mq-dual-linux.dtb /media/USER/FOURTH-PARTITION/ # That fourth partition is usually something with 8 characters like 12B7-6A23 in my case
```

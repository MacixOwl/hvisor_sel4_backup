
## Compilation & Run Steps
### 🚀【准备如下材料】🚀：
> 
> 1. linux开发环境 ，读卡器   
> 2. 开发板 imx-boot_4G.bin 启动文件（此处下载 [imx-boot_4G.bin](imx-boot_4G.bin) ）  
> 3. 根文件系统镜像 rootfs，文件较大（此处下载 [rootfs (提取码7mq8)](https://pan.sjtu.edu.cn/web/share/0c8436dbb208a66f73ca4d4ca772915c) ）
> 4. 开发板 Linux 5.4.70 SDK - 其中的内核及源码，文件较大（编译 hvisor-tool / hyperamp 都需要用，放在Server51上了，路径：/home/gongty/hvisor-files/OKMX8MP-C_Linux5.4.70+Qt5.15.0/Linux/sources/OK8MP-linux-sdk/OK8MP-linux-kernel）（不需要重复编译内核了，编译产物在OK8MP-linux-kernel/arch/arm64/boot/Image）  
> 5. （或者直接拷贝/home/gongty/hvisor-files整个目录，已包含上述所有文件了）

### SD卡内文件说明（如要删改，必须先备份）：
> Image：运行在 zone 0，必须是厂商提供的板载内核镜像，否则无法启动系统；  
> Image-5.4：运行在 zone 1，调试用；  
> hvisor.bin：hvisor启动镜像；
> hvisor、hvisor.ko：hvisor-tool内核模块；  
> 
> linux1.dtb、linux1.dts：启动 zone 0的设备树文件；  
> linux2.dtb、linux2.dts：启动 zone 1的设备树文件； 
>  
> OK8MP-C.dtb：没啥用，uboot启动时需要的板载设备树文件；  
> 
> linux2.json 或 sel4.json：启动 zone1 sel4 的配置文件；  
> con_virtio.json：启动 virtio 的配置文件；  
> hyperamp_backend：hyperamp 后端程序；  

------

### 1. 启动盘构建（可选）
- 一般来说，tf卡不需要重新构建，除非拿到手的是一张新卡（需要刷卡），或者频繁调试导致文件系统被写坏（只用重新拷贝镜像），或者启动盘损坏（等于变成新卡），需要用到上述2、3文件，已拷入SD启动盘，也可以在本备份仓库同级目录中下载。

按如下步骤刷卡，然后拷贝上述的 rootfs.tar 镜像并解压到卡。
1. 将SD卡插入读卡器，连接至主机；
2. 执行以下命令，进行分区：
```bash
sudo fdisk <$DRIVE> # <$DRIVE> 比如/dev/sdX，X按实际驱动号替换，注意不带数字，注意别选错盘不然电脑变砖头
d  # 删除所有分区
n  # 创建新分区
p  # 选择主分区
1  # 分区编号为1
16384  # 起始扇区（结束扇区回车默认即可）
t  # 更改分区类型
83  # 选择Linux文件系统（ext4）
w  # 保存并退出
```
3. 切换至/home/xxx/OKMX8MP-C_Linux5.4.70+Qt5.15.0/Linux/Images目录；
4. 将启动文件写入SD卡启动盘
```bash
dd if=imx-boot_4G.bin of=<$DRIVE> bs=1K seek=32 conv=fsync
```
5. 格式化SD卡启动盘第一个分区为ext4格式
```bash
sudo mkfs.ext4 <$DRIVE>
```
6. 将SD卡拔出，重新连接，将提前下载好的rootfs.ext4.tar解压到SD卡1号分区
```bash
sudo  tar -xvf rootfs1-backup.tar -C <path/to/mounted/SD/card/partition> # 比如/dev/sdX1
```
7. 完成后，即可弹出SD卡

------

### 2. 编译 hvisor （可选，使用[gitee仓库](https://gitee.com/open-microkernel/hvisor)）
```bash
make BID=aarch64/imx8mp LOG=info
```
编译产物为`/path/to/your/hvisor/target/aarch64-unknown-none/debug`下的`hvisor.bin`。
### 3. 编译 hvisor-tool（可选，但如果修改HyperAMP的共享内存地址，就要重新编译，使用[gitee仓库](https://gitee.com/open-microkernel/hvisor-tool)）
⚠️注意：hvisor-tool 必须使用和 hvisor 相对应的版本（简单来说gitee和github上的两个仓库版本可能有区别，不能混合使用，不知道现在有没有更新。当然，如果没遇到这个问题就不用管）
```bash 
# 必须使用板卡厂商提供的、未修改任何源码的 Linux 仓库 (版本 5.4.70)，否则无法正常运行
make all ARCH=arm64 LOG=LOG_WARN KDIR=/path/to/your/OKMX8MP-C_Linux5.4.70+Qt5.15.0/Linux/sources/OK8MP-linux-sdk/OK8MP-linux-kernel
```
编译产物为`/path/to/your/hvisor-tool/output`下的`hvisor.ko、hvisor`，以及`/path/to/hvisor-tool/tools/shm`下的`hyperamp_backend`。

------

### 4. 编译 seL4（拉取 HighSpeedCProxy 分支，编译 HyperAMP，使用[gitee仓库](https://gitee.com/open-microkernel/sel4test)）
```bash
../init-build.sh -DPLATFORM=imx8mp-evk -DAARCH64=1 -DSel4testApp=hyperamp-server
ninja
```
编译结果在`/path/to/your/seL4test/build-imx8mp/images`下。

✅ 编译结束后，把上述所有文件都拷入SD卡启动盘的`/home/arm64`路径中，并修改 sel4.json 中相应的镜像名称。(可以`lsblk`查看sd卡挂载点，然后直接cp进去；或者 vscode 中 ctrl + 单击打开这张卡，往里拖文件。⚠️拷完后，一定先`sudo umount /dev/sdX1`解除挂载，坚决不要热插拔)

------

### 5. 启动 hvisor + root Linux
1. 在 linux 主机安装`gtkterm` 
```bash
sudo apt update
sudo apt install gtkterm
```
2. 将开发板USB-C接口连接至主机，等待驱动加载完毕，在`gtkterm`中连接串口 /dev/ttyACM0（首次连接时需要`sudo chmod 666 /dev/ttyACM0`开放端口权限，重启后生效）
3. 将SD卡插入开发板，确认拨码开关状态为`{1, 2, 3, 4} = {off, on, off, off}`，拨动开关启动系统后，立即按住空格直至进入`uboot`命令行，按下`1`选项进入shell，用以下命令启动系统：
```shell
setenv loadaddr 0x40400000; setenv fdt_addr 0x40000000; setenv zone0_kernel_addr 0xa0400000; setenv zone0_fdt_addr 0xa0000000; ext4load mmc 1:1 ${loadaddr} /home/arm64/hvisor.bin; ext4load mmc 1:1 ${fdt_addr} /home/arm64/OK8MP-C.dtb; ext4load mmc 1:1 ${zone0_kernel_addr} /home/arm64/Image; ext4load mmc 1:1 ${zone0_fdt_addr} /home/arm64/linux1.dtb; bootm ${loadaddr} - ${fdt_addr};
```

- Tips: NXP 裸机启动 sel4（用于调试）
```shell
ext4load mmc 1:1 0x40400000 /home/arm64/Image-default.elf;  bootelf 0x40400000 - 0x5000000;
```

------

### 6. 进入 root Linux 后，启动系统
```bash
bash
cd home/arm64
insmod hvisor.ko
mount -t proc proc /proc
mount -t sysfs sysfs /sys
rm nohup.out
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

./hyperamp_backend > log.txt 2>&1 &
sleep 2
cat log.txt

nohup ./hvisor virtio start con_virtio.json &
nohup ./hvisor zone start sel4.json 

# 查看虚拟机运行状态
./hvisor zone list
```



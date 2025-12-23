
## Compilation & Run Steps
### 0. 启动盘构建（可选）
- Step 1-3 编译后的文件已拷入SD启动盘，也可以在本备份仓库同级目录中下载。
- 如需重新构建SD卡启动盘，可以按如下步骤刷卡，然后直接从 [云盘 (提取码7mq8)](https://pan.sjtu.edu.cn/web/share/0c8436dbb208a66f73ca4d4ca772915c) 拷贝tar包解压：
1. 将SD卡插入读卡器，连接至主机
2. 切换至/home/xxx/OKMX8MP-C_Linux5.4.70+Qt5.15.0/Linux/Images目录（或者另外下载 [imx启动文件](imx-boot_4G.bin) 到本地）
3. 执行以下命令，进行分区：
```bash
sudo fdisk <$DRIVE>
d  # 删除所有分区
n  # 创建新分区
p  # 选择主分区
1  # 分区编号为1
16384  # 起始扇区
t  # 更改分区类型
83  # 选择Linux文件系统（ext4）
w  # 保存并退出
```
4. 将启动文件写入SD卡启动盘
```bash
dd if=imx-boot_4G.bin of=<$DRIVE> bs=1K seek=32 conv=fsync
```
5. 格式化SD卡启动盘第一个分区为ext4格式
```bash
sudo mkfs.ext4 <$DRIVE>1
```
6. 将SD卡拔出，重新连接，将rootfs.ext4.tar解压到SD卡1号分区
```bash
sudo  tar -xvf rootfs1-backup.tar -C <path/to/mounted/SD/card/partition> 
```
7. 完成后，即可弹出SD卡

### 1. 编译 hvisor （可选，使用[gitee仓库](https://gitee.com/open-microkernel/hvisor)）
```bash
make BID=aarch64/imx8mp LOG=info
```
编译结果为`/path/to/your/hvisor/target/aarch64-unknown-none/debug`下的`hvisor.bin`。
### 2. 编译 hvisor-tool（可选，但如果修改HyperAMP的共享内存地址，就要重新编译，使用[gitee仓库](https://gitee.com/open-microkernel/hvisor-tool)）
```bash 
# 必须使用板卡厂商提供的、未修改任何源码的 Linux 仓库 (版本 5.4.70)
make all ARCH=arm64 LOG=LOG_WARN KDIR=/path/to/your/OKMX8MP-C_Linux5.4.70+Qt5.15.0/Linux/sources/OK8MP-linux-sdk/OK8MP-linux-kernel
```
编译结果为`/path/to/your/hvisor-tool/output`下的`hvisor.ko、hvisor`，以及`/path/to/hvisor-tool/tools/shm`下的`hyperamp_backend`。
### 3. 编译 seL4（拉取HighSpeedCProxy分支，编译HyperAMP，使用[gitee仓库](https://gitee.com/open-microkernel/sel4test)）
```bash
../init-build.sh -DPLATFORM=imx8mp-evk -DAARCH64=1 -DSel4testApp=hyperamp-server
ninja
```
编译结果在`/path/to/your/seL4test/build-imx8mp/images`下。
- 编译结束后，把上述所有文件都拷入SD卡启动盘的`/home/arm64`路径中。
### 4. 启动 hvisor + root Linux
1. 安装`gtkterm` 
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

### 5. 进入 root Linux 后，启动系统
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



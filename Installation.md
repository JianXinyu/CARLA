# CARLA Installation
## Machine Set-up 
### vGPU 云服务器
综合比较各家vGPU后，没有找到一个经济的选择。

Note: 
- Carla要求最少4GB显存，130G 硬盘
- 国内的平台基本都要实名认证

1. 滴滴云
560元/月，并给出了安装Carla的[教程](https://help.didiyun.com/hc/kb/article/1434810/)。但目前在维护，无法进行实名认证，也就无法购买服务。

2. 华为云
价格600元/月，但注册时无法选择国家为中国，后面很可能出问题，遂放弃。

3. 腾讯云
没有vGPU服务，只能租赁整块显卡，刚好符合Carla要求的也要900元/月

4. 阿里云
500元/月，但注册时需要支付宝扫码，始终显示服务器繁忙而无法绑定。

5. Google Cloud
200 USD/month 

### office台式机
DELL Precision 3640
与安装Carla相关的info:
- 仅有win10
- 500G SSD + 2T 固态， 系统盘在SSD上。*双硬盘会导致BIOS->SATA Operation Mode出问题，详见下文。*
- Nvidia Geforce GTX 2060 Super 8GB

硬件符合要求，但为以后方便起见，需要安装Ubuntu Dual System

Ubuntu version: 18.04.5 LTS

#### Install Ubuntu Dual System
[安装流程](https://www.cnblogs.com/masbay/p/10745170.html)
电脑类型为 UEFI新式bios+双硬盘（SSD固态硬盘+机械硬盘）+ 独显，因此参考教程中的:
- 情况C UEFI新式bios+单硬盘
- 情况D UEFI新式bios+双硬盘
- E  以上任意一种情况+电脑有特殊独立显(该部分没用)
- 教程比较落后，无需严格遵守，只要注意分区就问题不大

另外还可参考[Dell教程](https://www.dell.com/support/kbdoc/en-ca/000131655/how-to-install-ubuntu-linux-on-your-dell-pc)，我未参考。

- 尽量多分配点空间给Ubuntu, e.g., 300G，不然到后面需要扩容

- 用UltraISO + U盘制作系统盘，格式化时无需设置U盘File System，默认FAT即可。

- Dell 进入 BIOS: 出现Dell图标时按F2。改变BIOS Sequence。并且手动关闭了secure boot，不知道不关闭是否会影响。

- 安装Ubuntu时出现找不到分区的情况。[Solution](https://www.dell.com/support/kbdoc/en-ca/000131901/loading-ubuntu-on-systems-using-pcie-m2-drives) 在BIOS里把SATA setting 由 RAID 改为 AHCI。记住这个，双硬盘导致了这个会出问题，修改后导致Windows无法正常启动。

- 分区设置要格外小心！不然容易把原来的Win10搞崩

- 安装完之后直接重启即可，不用遵守 E 以上任意一种情况+电脑有特殊独立显

#### Boot set-up
此时Boot Sequence中应该是Ubuntu第一个。启动时应该会出现GRUB Menu，可以选择Windows Boot Manager。

由于之前设置了SATA Mode为AHCI，导致Windows无法启动，具体表现为卡在Dell图标处不断重启。这是因为Windows先前设置为RAID，但Ubuntu默认不支持RAID，只能用AHCI。[Solution](https://askubuntu.com/questions/1053589/how-can-i-boot-in-raid-mode)把Win10改为AHCI即可。

#### Install Nvidia Driver
- open terminal
- `ubuntu-drivers devices`查看显卡硬件型号
- `sudo ubuntu-drivers autoinstall` 安装默认推荐的驱动
- `lshw -c video` 查看显卡驱动
	- Nvidia driver = nouveau，需要修改
- [禁用nouveau](https://www.huaweicloud.com/articles/8deab91fc4a3a45bb3285433793ac501.html)
	- Ubuntu安装SSH
		- `sudo apt-get install openssh-server`
		- `ps -e|grep ssh` 确认ssh server是否启动. 如果只有ssh-agent那ssh-server还没有启动，需要`sudo /etc/init.d/ssh start`，如果看到sshd那说明ssh-server已经启动了
	- Win10安装使用Xshell [教程](https://zhuanlan.zhihu.com/p/28544384)
		- Xshell无法连接Ubuntu [solution](https://linuxize.com/post/how-to-enable-ssh-on-ubuntu-18-04/)
- 再次`lshw -c video` 查看显卡驱动，发现Nvidia driver=nvidia，可以了。

#### 给Ubuntu扩容
之前分配的空间可能不够
准备: 之前安装Ubuntu用到的U盘系统盘
[ref](http://www.cxyzjd.com/article/Mr_Sherlock/109840276)
- 首先在window下压缩出分配给ubuntu的空间
- reboot，进入Ubuntu `sudo apt-get install gparted`
- reboot，BIOS设置U盘启动，选择第一个不安装使用ubuntu。原因是在安装好的ubuntu下，即使两个ubuntu的两个分区相邻，也不能移动，因为前面会有锁的标志。
- `sudo gparted`
- 将unallocated paritition移动到需要合并的分区旁边，因为gparted只能对相邻的分区进行合并。但unallocated partition是无法移动的，因此只能挨个移动unallocated partition和目标分区, 等挨在一起了再合并 [ref](https://cn.noblenaz.org/870166-cannot-move-unallocated-space-QWYXQB-article)
- Linux swap area partition可能锁住了，there is a "key" on swap partition marking it as locked. The "lock" icon means that swap partition is mounted, that's why you can't modify it.  In GParted you can right click on the swap partition and select "Swapoff". This will disable the swap partition so you can move or resize it.

#### Reinstall Ubuntu
安装不同的驱动容易导致Ubuntu崩溃，与其找问题修复，不如直接重装来得快。
- 使用之前的系统U盘
- BIOS设置U盘启动
- 选择`install Ubuntu`
- 分区的时候把以前的Ubuntu的3个分区(swap, /, /home )删掉，再重建. efi无法删掉重建，留着就行。 参考[1](https://itsfoss.com/replace-linux-from-dual-boot/), [2](https://www.cnblogs.com/masbay/p/10745170.html)的情况C UEFI新式bios+单硬盘

## Install CARLA on Ubuntu
[Offical Doc](https://carla.readthedocs.io/en/latest/build_linux/)
我主要参考这个[ref](https://zhuanlan.zhihu.com/p/338927297) 
注意:
- 把UnrealEngine的路径加到bashrc里面
```text
export UE4_ROOT=~/UnrealEngine_4.24
```
- 运行Python前先在UI启动
### Troubleshooting
- 装完Ubuntu双系统后Windows无法启动，卡在品牌图标一直重启。
[Solution](https://askubuntu.com/questions/1053589/how-can-i-boot-in-raid-mode)把Win10改为AHCI模式。

- apt-get stuck at 0 [Connecting to us.archive.ubuntu.com]
	- 常规的[ipv6 fix](https://askubuntu.com/questions/574569/apt-get-stuck-at-0-connecting-to-us-archive-ubuntu-com) 对我不work
	- software&update -> download from main server. 这个成功了

- Ubuntu无法ping通Windows，但Windows能ping通Ubuntu [solution](https://blog.csdn.net/Ceosat/article/details/105057554)
- `make PythonAPI` ImportError: No module named distro
[Solution](https://github.com/carla-simulator/carla/issues/3166)`pip3 install --user distro`. `sudo apt-get install distro` and `pip install distro`不行

- 进入Ubuntu登录界面，鼠标键盘失灵。可能安装了某些软件导致的。
重新开机，按ESC键（如果无效就试试长按shift键）进入system recovery
进入到 grub的命令行界面，输入normal命令后回车，然后再按ESC
选择Advanced选项，然后再选择Recovery Mode
选择Network （Enable Network）回车，等完成后返回到上一层页面
选择root回车，输入sudo apt-get install xserver-xorg-input-all命令安装
退出当前页面（Menu），输入命令reboot，重启后即可使用键盘鼠标

- 不论是server还是client都只有3FPS
Editor->Preference->Performance->uncheck “use less cpu while in background”

- 有的python包已经安装了，仍然显示找不到
Carla仅支持Python3。很有可能是安装了Python2的包，再用pip3安装一遍就好了。

## Docker 构建 CARLA 镜像
[ref](https://blog.csdn.net/qwe900/article/details/116041960) 未做


`make launch`: LogMaterial: Display: Missing cached shader map for material
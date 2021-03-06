
Armbian 的客户化编译方法主要有两种模式，一种就是修改已有的编译选项和参数，另一种就是注入客户化配置及代码。第一种模式针对的需求比如配置过程的参数化、修改主机名称、设置网络、设置登录后初始界面的名称等；第二种模式针对的需求比如修改目标系统的配置、编译/安装软件包等。

首先看一下 Armbian 的目录结构及作用：

build：Armbian下载完成后的根目录，所有的文件保存在这个目录下，编译过程中动态产生及下载的文件也会存放在这里，编译运行的入口文件 compile.sh 也在这个目录下。
config：所有编译所需的配置信息都存放在这个目录下，比如板子的配置、系统引导的环境变量、引导时运行的脚本、内核的配置、U-boot 配置、编译时生成的用户可配置参数文件所用的模板以及下载编译工具用的种子文件等。
lib：存放编译运行的入口文件（compile.sh）所调用的其它的编译程序脚本。
packages：一些扩展包以及针对不同板子的配置及程序存放在这里。
patch：补丁程序的存放位置。
了解了目录结构和内容后，定制编译就是在相应的目录中增加、修改对应的配置及程序文件，举例如下：

需求一：非交互式编译，即不通过窗口菜单的方式进行编译。
方法是修改 “userpatches” 目录中的 config-example.conf 文件，这个目录是第一次编译后自动生成的，这个配置文件是从 config/templates/ 目录下的同名文件直接拷贝过来的，这里介绍几个主要的配置参数，完整的参数列表请看官网的说明。

KERNEL_ONLY（yes|no）：如果只是编译内核，设成 “yes” ，要完整的编译设成 “no”，对应窗口界面的内核编译或者完整映像编译的选项，设成空（“”）代表显示窗口配置界面；

KERNEL_CONFIGURE（yes|no）：设成 “yes” 代表编译前要配置内核，“no” 代表不修改缺省的配置，空（“”）表示显示内核的窗口配置界面；

CLEAN_LEVEL：每次编译的时候要清理的内容，“make” 代表执行 make clean，“images” 代表清除上次编译生成的映像文件（output/images/），“cache” 代表清除根文件系统缓冲文件（cache/rootfs/），多选用逗号分隔；

BOARD：开发板的名称，对应窗口界面的开发板选择部分；

BRANCH（legacy|current|dev）：内核版本设置，对应窗口界面的内核选择部分；

RELEASE（stretch|buster|bionic|focal）：操作系统设置，对应窗口界面的操作系统版本选择部分； 

BUILD_MINIMAL（yes|no）：“yes” 代表编译最小的可运行系统，而且不能编译有桌面环境的运行系统，“no” 则可以有桌面环境设置；

BUILD_DESKTOP（yes|no）：“yes” 代表有桌面环境，“no” 表示只有字符接口；

COMPRESS_OUTPUTIMAGE：生成的映像文件的压缩格式，是否生成映像的签名文件，是否生成映像的哈希文件。

那么在 config-example.conf 文件里按照下面的设置就可以实现非交互式编译并生成完整的不带桌面环境的标准 Ubuntu（focal）系统映像：

KERNEL_ONLY=”no”
KERNEL_CONFIGURE=”no”
CLEAN_LEVEL=”make,debs,cache”
BOARD=”lepotato”
BRANCH=”current”
RELEASE=”focal”
BUILD_MINIMAL=”no”
BUILD_DESKTOP=”no”
COMPRESS_OUTPUTIMAGE=”sha,gpg,img”

也可以通过把参数设置传递给编译入口程序 compile.sh 的方式实现同样的效果：

$sudo ./compile.sh BOARD=lepotato BRANCH=current RELEASE=focal BUILD_MINIMAL=no BUILD_DESKTOP=no KERNEL_ONLY=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img

需求二：修改登录欢迎提示以及主机名称（hostname）
修改欢迎提示首先是确定编译所用的开发板型号，然后到 config/board/ 目录下找到对应板子的配置说明文件，比如 lepotato.conf，编辑该文件，找到 BOARD_NAME 这一行，修改成所需要的字符串即可。

修改主机名称只要在 lepotato.conf 中增加一行 HOST=”hostname”，就可以了，放在下面要介绍的 lib.config 中也可以。

需求三：修改缺省的域名服务器地址，安装特定的软件包
修改缺省域名比较简单，在 userpatches/ 目录中创建文件 lib.config，然后编辑该文件并增加一行 NAMESERVER=x.x.x.x 就可以了，编译完成后在映像的根文件系统的 /etc/resolv.conf 文件中就是所设定的域名服务器地址。

安装特定的软件包只要在 lib.config 中增加下面一行说明即可：

PACKAGE_LIST_ADDITIONAL=”$PACKAGE_LIST_ADDITIONAL package_name”

另外，建议仔细阅读分析一下 lib/configuration.sh 这个文件，里面定义了重要的编译中所使用的环境变量，修改相应的变量可以改变编译结果，当然不是在这个文件中直接修改，而是放到 lib.config 当中做调整。

需求四：使用自定义的内核配置文件
如果在编译内核的时候希望使用客户自定义的配置，则可以在 userpatches/ 目录中放置相应的配置文件，格式为 linux-$KERNELFAMILY-$KERNELBRANCH.config。比如要对 lepotato 这个板子使用自定义的内核配置文件，则对应的配置文件名可以设置为 linux-meson64-current.config，该配置文件在编译内核的时候将会替换原有缺省的配置文件。

需求五：在打包的映像中运行客户化脚本
方法是在 userpatches/ 目录中编辑文件 customize-image.sh，写入需要执行的命令，这些命令是在 chroot 环境中运行的，该脚本文件是在编译完成后并在生成最终目标映像之前被执行。比如取消登录后创建新用户的步骤，可以使用下面的命令：

#rm -f /root/.not_logged_in_yet ;放到 customize-image.sh 中

需求六：其它的客户需求
上面的客户化定制方法可以满足很大一部分需求，如果需要做更深入的定制，可以研究源代码中已有的配置信息、代码。另外，通过查看整个编译过程的输出信息，还可以详细了解编译执行的阶段、所作操作等内容，这对做客户化编译配置非常有帮助，这些信息保存在 output/debug/ 目录中。另外官网的这个页面也有很详细的说明。

最后，生成的映像文件可以通过工具写入 U 盘或 SD 卡，有两种方法，一种是借助写盘工具，比如 rufus 和 balenaEtcher；另外一个就是在 config-example.conf 中增加一行定义 CARD_DEVICE=/dev/sdx，sdx 就是所连接的 U 盘或 SD 卡的设备名。
runner@fv-az58:~/work/Actions-Armbian-s905/Actions-Armbian-s905/armbian/output$ ls -R
.:
config  debs  debs-beta  debug  images  patch

./config:

./debs:
armbian-config_20.08.0-trunk_all.deb         
armbian-firmware_20.08.0-trunk_all.deb  focal                                              
linux-headers-current-meson64_20.08.0-trunk_arm64.deb  
linux-source-current-meson64_20.08.0-trunk_all.deb
armbian-firmware-full_20.08.0-trunk_all.deb  extra                                   
linux-dtb-current-meson64_20.08.0-trunk_arm64.deb  
linux-image-current-meson64_20.08.0-trunk_arm64.deb    
linux-u-boot-current-lepotato_20.08.0-trunk_arm64.deb

./debs/extra:

./debs/focal:
armbian-focal-desktop_20.08.0-trunk_all.deb  
linux-focal-root-current-lepotato_20.08.0-trunk_arm64.deb

./debs-beta:
extra

./debs-beta/extra:

./debug:
compilation.log  
compiler.log  
install.log  
installed-packages-focal.list  
logs-.tgz  
logs-06_08_2020-18_35_54.tgz  
output.log  
patching.log  
timestamp

./images:
Armbian_20.08.0-trunk_Lepotato_focal_current_5.7.13.img  
Armbian_20.08.0-trunk_Lepotato_focal_current_5.7.13.img.sha  
Armbian_20.08.0-trunk_Lepotato_focal_current_5.7.13.img.txt


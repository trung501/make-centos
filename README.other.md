本文主要介绍如何根据官方的Centos镜像文件，在保留原有默认安装的RPM包的基础下，添加自己所需要的RPM包的，最终生成一个自定制版的ISO，节省了宝贵的时间并确保了安装的定制性。对于其他没有介绍的修改，后续在实践中会进行更新。

本文基于Centos7.3版本制作的ISO镜像，其他版本可能本文介绍的制作过程有所差别。

 

搭建基础环境

#yum install createrepo mkisofs isomd5sum squashfs-tools

#mkdir /root/PanISO

将/root/PanISO作为ISO的制作目录

#mount /dev/cdrom /media/

#cp -r /media/* /root/PanIOS/

#cp  /media/.discinfo /root/PanIOS/

#cp  /media/.treeinfo /root/PanIOS/

# chmod +w /root/PanISO/isolinux/isolinux.cfg

注意：如果你想用Centos7.3的ISO进行挂载也是可以的

#mount -o loop /root/CentOS-7-x86_64-DVD-1611.iso /mnt

 

修改isolinux.cfg文件，将“append initrd=initrd.img”后面的当前行内容删除，并加入“ks=cdrom:/isolinux/ks.cfg”。

复制代码
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install CentOS Linux 7
  kernel vmlinuz
  append initrd=initrd.img ks=cdrom:/isolinux/ks.cfg 
复制代码
 这步的作用是实现自动化安装，如果不加上就需要手动配置参数就行下一步安装了。

 

修改自动化安装配置文件

#cp /root/anaconda-ks.cfg /root/PanISO/isolinux/ks.cfg

本次制作iso用的是系统安装成功生成的默认的anaconda-ks.cfg文件，并未做修改，如果有需求可以自行修改。下面是我使用的示例，并稍加了一些文件配置说明（详细配置说明可以查找kickstart配置文件）：

复制代码
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media 从光驱安装  
cdrom
# Use graphical install 图形化安装
graphical
# Run the Setup Agent on first boot  
firstboot --enable 
ignoredisk --only-use=sda
# Keyboard layouts 美式键盘
keyboard --vckeymap=us --xlayouts='us'
# System language 美式英语
lang en_US.UTF-8

# Network information 网卡配置
network  --bootproto=dhcp --device=ens33 --onboot=off --ipv6=auto --no-activate
network  --hostname=localhost.localdomain

# Root password root用户的密码
rootpw --iscrypted $6$Ok9Jcj51va/3x830$/6rLkpu8k2tPCmd7byUBE7wuRexmuoMzp0jAelDRYMAIk9yRL/84mCFrOTp5QYWJNVcEIB7wWgw8byp0r21vT0
# System services
services --disabled="chronyd"
# System timezone 时区
timezone Asia/Shanghai --isUtc --nontp
user --name=pan --password=$6$ONSyoQ.S58OJpcnj$jUz6vDadzY5wZ39fr0dEONbI/iNIeVkpRMaUjz9ZJbIqQLPLKqq8ZJWRoDGjolLJfkwmw58Dp5xPhKufAca8y/ --iscrypted --gecos="pan"
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
clearpart --none --initlabel

%pre
%end

#安装包的信息
%packages
@^minimal
@core
kexec-tools

%end

%post
%end
 
%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=50 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=50 --notstrict --nochanges --notempty
pwpolicy luks --minlen=6 --minquality=50 --notstrict --nochanges --notempty
%end
复制代码
 

 %pre表示系统安装前，%post 表示在系统安装后执行，这样可以定制自己的自动化脚本

 

 获取系统默认安装的RPM包和需要添加的RPM包

在使用Centos系统安装完成后会生成/root/install.log，该文件记录了系统安装时安装的RPM包信息。如果没有该文件，可以手动生成（新安装的干净系统）：

#rpm -qa >> /root/install.log

清空ISO制作目录里的Packages和repodata两个目录里的所有内容，并根据install.log将所需安装包放入Packages文件夹内：

 

# awk '{print $2}'  /root/install.log |xargs -i cp /media/Packages/{}.rpm /root/PanIOS/Packages/

 

 注：如果是手动生成的install.log，将'{print $2}' 改为'{print $0}' 。

 

因为需要自定制iso，需要预安装其他的包，将解决好依赖关系的包全部放入/root/PanIOS/Packages/中：

多数情况下我们会根据yum来下载安装包，下面介绍两种获取下载安装包的方法：

1.修改yum的配置文件，将yum下载的安装包保存起来

#vim /etc/yum.conf

修改keepcache=1 （1为保存，0为不保存，默认是0）

修改后使用yum安装的包会保存在“/var/cache/yum/”下。

2.通过yum指令的--downloadonly可以只下载安装包，不进行安装

#yum -y install --downloadonly --downloaddir=/root/test/ <file.name>

该指令我会将安装的包统一放在/root/test/目录下，yum update同样可以使用该方法，这样定制后的ISO中RPM包都是最新版本的。

 

修改comp.xml文件，定义RPM包组

.xml文件的写法如下：

 

复制代码
<group>  
   <id>组的ID，非数字</id>  
   <name>组的名字</name>  
   <description>组的描述</description>  
   <default>是否预安装，true或者false</default>  
   <uservisible>是否可见，true或者false</uservisible>  
   <langonly>zh</langonly>  #仅在某个语系的安装界面中显示，可选项  
   <packagelist>  
      <packagereq requires="依赖包" type="conditional">软件包1</packagereq>  
      <packagereq type="default">软件包2</packagereq>  
      <packagereq type="default">软件包3</packagereq>  
   </packagelist>  
  </group>  

......
  <category>  
   <id>分类的名字，非数字</id>  
   <name>将显示在左侧列表里</name>  
   <description>将显示在下面的描述栏里</description>  
   <grouplist>  
    <groupid>组1的ID</groupid>  
    <groupid>组2的ID</groupid>  
   </grouplist>  
  </category>  
  <category>  
复制代码
 

简单来说，一个group中包含若干个RPM包，一个category则包含了若干个group，在安装系统的时候，在选择自定义安装的步骤中，左侧的是category，右侧是group。

1.group id我们可以在ks.cfg来指定

%packages
@^minimal
@core
kexec-tools
示例中的@core就是一个组的名称，如果我们想添加指定的包组，就可以在ks.cfg文件中指定；

2.在group中的包含的RPM只需要填入完整包的名称即可，不需要把依赖也添加进去，只需要在Packages文件夹内将所需的依赖添加完整即可；

3.在group中添加完整包的名称是yum安装下显示的包名，比如vim，它的完整包的名称是vim-enhanced；

4.当然你也可以在Packages中保留完整的Centos系统的RPM包，将自定制的分类和组添加进去，这样在安装界面就可以发现我们自定义的category和group。

 

重新生成repo

#cd PanISO

#createrepo -g comps.xml .

注意：在CentOS下需要根据'.discinfo'来设置'baseurl'(declare -x discinfo=head -1 .discinfo; createrepo -u "media://$discinfo"...); 在CentOS7中不再需要如此做，实际上如果在CentOS7中执行了这个命令，在安装的过程中，可能会报错"RepoError after 10 retries: Insufficient space in download directory /run/install/repo/Packages"

在其他版本中可执行如下指令：

# declare -x discinfo=$(head -1 /root/PanIOS/.discinfo)

# createrepo -u "media://$discinfo" -g /root/minimal-x86_64.xml /root/PanIOS/

 

修改安装界面图标背景

以图标为例，其他操作类似

将安装界面左上角的该图标换为

解压

# unsquashfs /root/PanISO/LiveOS/squashfs.img

把解压后的文件进行挂载，然后操作

#mount -o loop,rw squashfs-root/LiveOS/rootfs.img /media

/media/usr/share/anaconda/pixmaps/sidebar-logo.png为该安装界面的图标，只需根据自己的需要替换即可，分辨率要跟原图保持基本一致，要不会出现图标过大的情况

将解压后的文件重新打包

#mksquashfs squashfs-root/   squashfs.img

并将生成的squashfs.img替换原来的squashfs.img

 

制作ISO

#mkisofs -o Pan-7.3.iso -input-charset utf-8 -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -R -J -v -T -joliet-long  /root/PanIOS/

#implantisomd5 Pan-7.3.iso

 

# DGX-A100-PXE-INSTALL

#下方操作已封装成镜像，可参考https://hub.docker.com/r/zhangyukang/pxe-server-for_dgx-a100-a800 ； docker pull zhangyukang/pxe-server-for_dgx-a100-a800

DGX A100 PXE安装流程

参考资料： https://docs.nvidia.com/dgx/dgx-os-5-user-guide/appendix_d_pxe_boot_setup.html

1. 概述

   官方给出的方案有很多选择，我们此次使用最简单的方案 dnsmasq + apache2 + PXE

   dnsmasq 提供DHCP、DNS、TFTP三个服务，我们只需要DHCP和TFTP这两个。

   为了避免对宿主机造成污染，我们选用docker进行部署，由于容器内一般不允许执行systemctl命令，此处选用dockerhub上别人封装好的镜像 

   https://hub.docker.com/r/jrei/systemd-ubuntu

2. 创建容器

```bash
docker pull jrei/systemd-ubuntu:20.04
docker run -idt --name pxe-server --tmpfs /tmp --tmpfs /run/lock -v /sys/fs/cgroup:/sys/fs/cgroup:ro --network host --privileged -e TZ=Asia/Shanghai jrei/systemd-ubuntu:20.04
```



#### 以下操作均在docker容器内进行，如果不采用docker方式也可以在宿主机直接配置



3. 配置时间同步

   ```bash
   # 安装ntp客户端及基本调试工具
   apt update
   apt install vim net-tool* chrony inetutils-ping wget curl
   # 配置NTP
   vim /etc/chrony/chrony.conf
   '''
   末尾添加 
   server ntp.aliyun.com
   '''
   systemctl enable chrony.service
   systemctl enable chrony.service
   # 注意 这跟centos系统有所不同，centos系统里是 chronyd
   chronyc -a makestep
   chronyc sources -v
   ```

   

4. 安装所需的包

   ```bash
   apt install dnsmasq apache2
   
   ```

   

5. 配置dnsmasq ，在 /etc/dnsmasq.conf中输入以下内容

   写该文档时dnsmasq所在节点地址是 172.17.224.139 ，目标要装机的服务器 MAC  ab-cd-ef-gh-ij-kl ，需要为其分配的地址是 172.17.226.46（网关 172.17.226.254，子网掩码 /24），

   ```bash
   cat /etc/dnsmasq.conf
   '''
   conf-dir=/etc/dnsmasq.d,.rpmnew,.rpmsave,.rpmorig
   
   # port=0 意思是关闭DNS功能，如果需要开启DNS功能删除该行即可
   port=0
   
   # 绑定网卡
   interface=enp26s0f1
   dhcp-range=enp97s0f1,172.17.226.46,static,2h
   # 指定按mac分配
   dhcp-host=ab:cd:ef:gh:ij:kl,DGX-046,172.17.226.46
   
   dhcp-option=option:router,172.17.226.254
   dhcp-option=option:netmask,255.255.255.0
   #dhcp-mac=set:DGXA100,ab:cd:ef:gh:ij:kl
   #dhcp-broadcast=option:DGXA100:172.17.226.255
   
   #dhcp-option=3,172.17.226.254
   #dhcp-option=6,172.17.224.139
   dhcp-option=66,172.17.226.254
   pxe-service=x86PC, "Boot from Network pxelinux", syslinux.efi
   enable-tftp
   tftp-root=/local/syslinux/efi64/
   dhcp-boot=syslinux.efi
   resolv-file=/etc/resolv.dnsmasq.conf
   strict-order
   listen-address=::1,172.17.224.139,127.0.0.1
   no-hosts
   log-queries
   log-facility=/var/log/dnsmasq/dnsmasq.log
   log-async=50
   
   '''
   
   # 如果不需要DNS功能可以忽略此配置
   vim /etc/resolv.dnsmasq.con
   '''
   nameserver 223.5.5.5
   nameserver 119.29.29.29
   nameserver 114.114.114.114
   nameserver 8.8.8.8
   '''
   
   systemctl restart dnsmasq
   ```

   

6. 配置apache2，提供HTTP服务

   ```bash
   ## 修改index位置
   vim /etc/apache2/apache2.conf
   # 修改前
   <Directory /var/www/html>
           Options Indexes FollowSymLinks
           AllowOverride None
           Require all granted
   </Directory>
   # 修改后
   <Directory /local/http>
           Options Indexes FollowSymLinks
           AllowOverride None
           Require all granted
   </Directory>
   
   ## 配置http服务文件路径
   vim /etc/apache2/sites-available/000-default.conf
   # 修改前
   DocumentRoot /var/www/html
   # 修改后
   DocumentRoot /local/http
   
   systemctl restart apache2
   
   
   
   ```

   

7. 准备装机镜像和引导文件

   该部分参考：  https://docs.nvidia.com/dgx/dgx-os-5-user-guide/appendix_d_pxe_boot_setup.html#configuring-the-http-file-directory-and-iso-image

   ```bash
   # 下载DGX的镜像文件，略
   # 例如 DGX的ISO文件在 /tmp下
   
   # 挂载 （docker内执行mount可能有问题，可以先在宿主机上挂载，然后docker cp进容器内）
   mount -o loop /tmp/base_os_5.x.y.iso /mnt
   cp /mnt/{initrd,vmlinuz} /local/http/dgxbaseos-5.x.y/
   cp /tmp/dgxbaseos-5.0.2.iso /local/http/dgxbaseos-5.x.y/
   
   # 完成以上操作后，http服务根目录下的文件结构如下
   /local/http
   └── dgxbaseos-5.x.y
       ├── base_os_5.x.y.iso
       ├── initrd
       └── vmlinuz
   ```

   

8. 准备pxe引导文件

   该部分参考：https://docs.nvidia.com/dgx/dgx-os-5-user-guide/appendix_d_pxe_boot_setup.html#configuring-the-syslinux-bootloader

   ```bash
   cd /local/
   wget https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/Testing/6.04/syslinux-6.04-pre1.tar.gz
   tar -xvf syslinux-6.04-pre1.tar.gz
   mkdir /local/syslinux/efi64/pxelinux.cfg
   cp syslinux-6.04-pre1/bios/core/pxelinux.0 syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/efi/syslinux.efi syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/com32/elflink/ldlinux/ldlinux.e64 syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/com32/menu/vesamenu.c32 syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/com32/menu/menu.c32 syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/com32/libutil/libutil.c32 syslinux/efi64/
   cp syslinux-6.04-pre1/efi64/com32/lib/libcom32.c32 syslinux/efi64/
   
   # 编辑default文件
   # 如下是官方给的建议，通过PXE安装后会进入配置界面，选择国家、键盘、配置user等
   # 如果将config改为 nooemconfig ，会自动安装，不进入配置界面，默认用户密码为 nvidia:nvidia
   '''
   default menu
   prompt 0
   timeout 300
   menu title DGX Install Images
   label dgxbaseos-5.x.y
   menu label DGXBaseOS 5.x.y (Base OS)
   kernel http://<http-address>/dgxbaseos-5.x.y/vmlinuz
   initrd http://<http-address>/dgxbaseos-5.x.y/initrd
   append boot=live config components union=overlay noswap noeject ip::enp226s0:dhcp ethdevice-timeout=60 fetch=http://<http-address>/dgxbaseos-5.x.y/DGXOS-5.x.y-<release-labels>.iso apparmor=0 elevator=noop nvme-core.multipath=n nouveau.modeset=0 rebuild-raid offwhendone console=tty0 console=ttyS1,115200n8
   '''
   
   # 如下是进行部分修改的，后台静默安装，添加用户并设置密码，以及进行其他必要的配置
   '''
   default menu
   prompt 0
   timeout 300
   menu title DGX Install Images
   label http/dgxbaseos-5.4.1
   menu label DGXBaseOS 5.4.1 (Base OS)
   kernel http://172.17.224.139/dgxbaseos-5.4.1/vmlinuz
   initrd http://172.17.224.139/dgxbaseos-5.4.1/initrd
   append boot=live nooemconfig components union=overlay noswap noeject ip::enp97s0f1:dhcp ethdevice-timeout=60 fetch=http://172.17.224.139/dgxbaseos-5.4.1/DGXOS-5.4.1-2022-10-11-17-49-32.iso apparmor=0 elevator=noop nvme-core.multipath=n nouveau.modeset=0 rebuild-raid offwhendone console=tty0 console=ttyS1,115200n8  set-nvme-boot-1st force-curtin=http://172.17.224.139/curtin/dgx_a800-curtin.yaml
   '''
   
   curtin文件获取及配置见下方步骤
   
   ```

   

9. curtin文件获取及配置

   该部分参考： https://docs.nvidia.com/dgx/dgx-os-5-user-guide/appendix_d_pxe_boot_setup.html#customizing-the-curtin-yaml

   ```bash
   mount -o loop /tmp/base_os_5.x.y.iso /mnt
   cp /mnt/live/filesystem.squashfs .
   unsquashfs filesystem.squashfs
   # 将curtin放到 http目录下
   cp  squashfs-oot/curtin  /local/http/
   
   # 修改 /local/http/curtin/dgx_a100-curtin.yaml，在末尾添加如下几行
   # 注意，不要直接复制官方文档里的内容，会因为命令与参数之间缺少空格而报错
   '''
       # Create user
     50_create_user: ["curtin", "in-target", "--", "sh", "-c",
       "adduser --disabled-password --gecos --quiet supdev"]
     51_add_user_group: ["curtin", "in-target", "--", "sh", "-c",
       "usermod -aG sudo supdev"]
     52_set_user_password: ["curtin", "in-target", "--", "sh", "-c",
       "echo 'supdev:jd@123456' | chpasswd"]
   '''
   ```

   

10. 通过带外进入BMC，重启测试节点，按DEL进入BIOS，开启网络协议及PXE启动

    ![](.\实验截图\1684145362308.jpg)

    ![1684145398936](.\实验截图\1684145398936.png)

    

11. 通过ipmitool指定主机从PXE启动

    ```bash
    ipmitool -I lanplus -H <DGX-BMC-IP> -U <username> -P <password> chassis bootdev pxe options=efiboot
    ipmitool -I lanplus -H <DGX-BMC-IP> -U <username> -P <password> chassis power reset
    ```

    

12. 注意事项

    如果物理环境中PXE-SERVER节点与目标节点之间跨多个子网，需要在中间的交换机上配置dhcp relay（dhcp中继）

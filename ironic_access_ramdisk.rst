=======================================
ironic部署裸机过程中登陆ramdisk方法总结
=======================================

通过console登陆ramdisk
======================

**方法一(说明：本方法采用的镜像是官方提供的镜像，采用镜像里面默认打包进去的用户core来登陆console)：**

1. 修改ironic配置文件/etc/ironic/ironic.conf，在pxe_append_paramscansh参数后添加参数coreos.autologin：::

    [pxe]

    pxe_append_params=nofb nomodeset vga=normal coreos.autologin

2. 重启ironic-conductor服务：::

    systemctl restart openstack-ironic-conductor

**方法二（说明：通过对官方镜像进行重新打包压缩，注入自定义用户名和密码）：**

1. 在镜像文件的同级目录下新建文件夹tmp_folder，然后进入该文件夹下，将官方镜像解压至该文件夹下：::

    cd tmp_folder

    zcat ../coreos_production_pxe_image-oem-stable-liberty.cpio.gz \

    | cpio --extract --make-directories

2. 找到cloud_config.yml文件，编辑该文件，在其末尾加上自定义用户名和密码信息：::

    cat ./usr/share/oem/cloud_config.yml

    ...

    users:

    -name : easystack

    passwd: XXXXXXXXXXXXXXX

    groups:
          -sudo

   上述name后跟的是用户名，可以根据自己喜好设定；passwd后跟的是密码的hash值，

   可以通过下述命令来获得密码的hash值：::

    mkpasswd --method=SHA-512 --rounds=4096

3. 重新打包上传镜像，回到tmp_folder目录，对镜像进行打包：::

    cd ..

    find .| cpio --create --format='newc' | gzip -c -9 > \

    ../coreos_production_pxe_image_oem-stable.NEW.cpio.gz

4. 待上述步骤完成之后，回到父目录，会发现重新生成的名字为coreos_production_pxe_image

   _oem-stable.NEW.cpio.gz的镜像，上传即可。部署过程中可以使用自己定义的用户名和密码登陆ramdisk。

通过ssh登陆ramdisk
==================

**方法一（通过使用官方提供的镜像ssh登陆ramdisk）：**

1. 修改ironic配置文件/etc/ironic/ironic.conf，在pxe_append_paramscansh

   参数后添加参数sshkey="xxxxx"：::

    [pxe]

    pxe_append_params=nofb nomodeset vga=normal sshkey="xxxxxxxxx"

2. 重启ironic-conductor服务：::

    systemctl restart openstack-ironic-conductor

   上述sshkey的内容是你用来ssh登陆ramdisk的主机的公钥信息，一般情况下是和裸机部署网络通的主机

   的公钥信息，推荐为controller节点的公钥信息。配置完成后即可使用以下命令来访问ramdisk：::

    ssh core@$ramdisk_ip

**方法二（使用console登陆的方法二重新打包的镜像）：**

1. 参考console登陆的方法二对镜像进行解压重新打包，注入自定义用户名和密码；

2. 通过用户名密码，ssh访问ramdisk。

**重要说明：上述登陆ramdisk的方法都是默认的非root用户，通过执行#sudo su -  无密码获取root权限。**
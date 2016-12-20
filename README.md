ironic node-create -d fuel_ipmitool

ironic driver-properties fuel_ipmitool
  
ironic node-update $NODE_UUID add name=$NODE_NAME

ironic node-update $NODE_UUID add driver_info/ipmi_username=USERID driver_info/ipmi_password=PASSW0RD driver_info/ipmi_address=9.111.115.35
  
ironic node-update $NODE_UUID add driver_info/deploy_kernel=6f24c390-f6ec-4ce5-816f-d14cb8287964 driver_info/deploy_ramdisk=a7c0a89d-708d-4791-bf05-ea266d36902e driver_info/deploy_squashfs=9bcc0dfc-a758-4f26-92d6-3aaaa9fa592a
  
ironic node-update $NODE_UUID add properties/cpus=32 properties/memory_mb=130000 properties/local_gb=2560 properties/cpu_arch=x86_64
  
ironic node-update $NODE_UUID add properties/capabilities="boot_option:local"

ironic node-update $NODE_UUID add driver_info/ipmi_terminal_port=9000

nova flavor-create $FLAVOR_NAME auto 4096 50 1
nova flavor-key $FLAVOR_NAME set cpu_arch="x86_64"
nova flavor-key $FLAVOR_NAME set capabilities:boot_option="local"
标签:方便选择部署在哪个物理机上

ironic node-update $NODE_UUID add properties/capabilities='profile:$NODE_NAME,boot_option:local'
 
nova flavor-key $FLAVOR_NAME set capabilities:profile="$NODE_NAME"
Image

部署物理机的image是需要单独制作的.使用diskimage-builder工具即可.安装及使用如下: 在一台可以访问internet的机器上安装pip

yum install python-pip
pip install --upgrade pip
pip install diskimage-builder
定义image的一些属性

export DIB_CLOUD_INIT_DATASOURCES="Ec2, ConfigDrive, OpenStack"
export DIB_DEV_USER_USERNAME=$USERNAME
export DIB_DEV_USER_PASSWORD=$PASSWORD
export DIB_DEV_USER_PWDLESS_SUDO=yes
export DIB_DEV_USER_AUTHORIZED_KEYS=/root/.ssh/id_rsa.pub (option)
export DIB_EPEL_MIRROR=http://dl.fedoraproject.org/pub/epel (option)
    开始制作
disk-image-create -a amd64 -p parted,vim bootloader dhcp-all-interfaces enable-serial-console cloud-init-datasources devuser (epel) selinux-permissive baremetal centos7 -o mustang_centos7
上传和更新image

glance image-create --name $NAME.initrd --visibility public --disk-format ari --container-format bare < $NAME.initrd

glance image-create --name $NAME.vmlinuz --visibility public --disk-format aki --container-format bare < $NAME.vmlinuz

glance image-create --name $NAME --visibility public --disk-format qcow2 --container-format bare --property kernel_id=$ID --property ramdisk_id=$ID < user_image.qcow2
然后使用此image启动一个instance,更改其cloud-init,sshd配置文件,使其允许root使用密码登录.

vim /etc/cloud/cloud.cfg
disable_root：0
ssh_pwauth：  1
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
然后清理instance,关机,做snapshot.将做好的snapshot下载下来 openstack image save $IMAGE_UUID --file $FILE_NAME 然后转换成raw格式

qemu-img convert -f qcow2 -O raw INPUTE_FILENAME $OUTPUT_FILENAME
将raw格式文件上传至glance,并且添加自定义参数,使用不久之前上传的kernel和ramdisk.

glance image-create --name $NAME --visibility public --disk-format raw --container-format bare --property kernel_id=$ID --property ramdisk_id=$ID < user_image.raw
最后更新image的metadata

glance image-update $IMAGE_UUID --property cpu_arch=x86_64 --property hypervisor_type="baremetal" --property fuel_disk_info='[{"name": "sda", "extra": [], "free_space": 51200, "type": "disk", "id": "vda", "size": 51200, "volumes": [{"mount": "/", "type": "partition", "file_system": "ext4", "size": 40000}]}]'
Web Console

安装相关软件包

Ubuntu:
    sudo apt-get install shellinabox

Fedora 21/RHEL7/CentOS7:
    sudo yum install shellinabox

Fedora 22 or higher:
    sudo dnf install shellinabox
更改ironic.conf文件

pxe_append_params = nofb nomodeset vga=normal console=tty0 console=ttyS0,115200n8

ironic node-update $NODE_UUID add driver_info/ipmi_terminal_port=9000

ironic node-set-console-mode $NODE_UUID true

ironic node-get-console $NODE_UUID

# 部署Overcloud

---

overcloud 部署步骤

1. 收集物理机的pxe网卡 mac地址。 ipmi地址、用户名、密码。
2. 定义overcloud 节点配置文件,填入上一步收集的overcloud所有节点的ipmi地址、用户名密码 、pxe mac 。
3. 上传overcloud镜像
4. 导入 节点
5. inspection node
6. 定义根磁盘
7. 定义节点类型
8. 部署overcloud

在这一步，我们已经完成了undercloud的部署。现在要开始部署overcloud。

## 1. 准备overcloud 镜像

Overcloud 镜像可以自己制作也可以下载现成的, 如果要使用ustack内部的镜像，请使用lftp 下载。tripleo.ustack.com 对应的 IP 是10.0.100.250.

```bash
$ lftp tripleo.ustack.com:/pub/images/newton/
cd ok, cwd=/pub/images/newton
lftp tripleo.ustack.com:/pub/images/newton> ls
-rw-r--r--    1 0        0        1670748160 Jan 08 10:49 overcloud_image.tar
lftp tripleo.ustack.com:/pub/images/newton> get overcloud_image.tar
```

这里的演示使用下载的镜像。[Overcloud 镜像下载地址](http://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/)

下载后解压，将这些文件放到stack用户的根目录底下：

```
$ ls ~/images/
ironic-python-agent.initramfs
ironic-python-agent.kernel
overcloud-full.initrd
overcloud-full.qcow2
overcloud-full.vmlinux
```

> 如果要修改镜像的root密码： `$ virt-customize -a overcloud-full.qcow2 --root-password password:<my_root_password>`

## 2. 上传镜像

第一次部署推荐使用社区镜像，PoC 环境和生产环境一定要自己制作镜像。

```
$ . stackrc
$ openstack overcloud image upload --image-path /home/stack/images
```

> 默认情况下，我们的glance使用的是swift的存储后端，所以在进行这个步骤时必须保证你的swift的服务是可以使用的。

上传完成之后，查看镜像：

```
$ openstack image list
+--------------------------------------+------------------------+
| ID | Name |
+--------------------------------------+------------------------+
| 765a46af-4417-4592-91e5-a300ead3faf6 | bm-deploy-ramdisk      |
| 09b40e3d-0382-4925-a356-3a4b4f36b514 | bm-deploy-kernel       |
| ef793cd0-e65c-456a-a675-63cd57610bd5 | overcloud-full         |
| 9a51a6cb-4670-40de-b64b-b70f4dd44152 | overcloud-full-initrd  |
| 4f7e33f4-d617-47c1-b36f-cbe90f132e5d | overcloud-full-vmlinuz |
+--------------------------------------+------------------------+
```

这个会显示在收集物理机信息的时候使用的PXE镜像，上传的时候会把这些镜像都拷贝到/httpboot这个目录下面

## 3. 收集物理机信息

我们现在已经有了镜像，紧接着就是定义我们overcloud主机了。我们将overcloud vm的信息写入nodes.yaml。参照以下格式：

```
nodes:
    - name: rack2-3-compute
      driver: pxe_ipmitool
      driver_info:
        ipmi_address: 10.0.108.119
        ipmi_username: root
        ipmi_password: PQ79ISF7ha7G
      properties:
        cpus: 4
        memory_mb: 12288
        local_gb: 60
        capabilities: 'boot_option:local'
        root_device:
          name: /dev/sda
      ports:
        - address: 24:6E:96:06:A2:1D
```

## 4. 导入overcloud信息

```
openstack baremetal create nodes.yaml
```

为 nodes 分配ramdisk 和kernal

```
DEPLOY_KERNEL=$(openstack image show bm-deploy-kernel -f value -c id)
DEPLOY_RAMDISK=$(openstack image show bm-deploy-ramdisk -f value -c id)

for uuid in $(openstack baremetal node list -f value -c UUID);
do
    openstack baremetal node set $uuid \
        --driver-info deploy_kernel=$DEPLOY_KERNEL \
        --driver-info deploy_ramdisk=$DEPLOY_RAMDISK
done
```

进行introspection收集overcloud node 的信息

```
openstack baremetal introspection bulk start
```

## 5. 为物理机定义节点类型

在规划节点时，希望特定的物理机作为特定的角色。比如有一台物理机，我们在ironic 配置文件里将它定义为overcloud 中ceph节点，不要任性的变成计算节点或者控制节点。这样，我们就需要为这些节点定义类型。  
定义类型有两种方法，比如我只需要ceph节点，安装在ceph host 上\(在ironic 配置文件中叫做ceph\)，nova 节点安装在nova host 上，控制节点安装在controller host 上。

正常写法：

```
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' <compute node uuid>
openstack baremetal node set --property capabilities='profile:control,boot_option:local' <control node uuid>
openstack baremetal node set --property capabilities='profile:ceph-storage,boot_option:local' <ceph node uuid>
```

一句话写法：

```
ironic node-list|grep 'controller'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:control,boot_option:local'

ironic node-list|grep 'compute'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:compute,boot_option:local'

ironic node-list|grep 'ceph'|awk '{print $2}'|xargs -I{} ironic node-update {} add properties/capabilities='profile:ceph-storage,boot_option:local'
```

## 4. 定义根磁盘

在执行完`openstack baremetal introspection bulk start`之后，根据得到的信息来定义overcloud节点的根磁盘。  
根磁盘可以通过以下参数来指定。

```
model (String): Device identifier.
vendor (String): Device vendor.
serial (String): Disk serial number.
wwn (String): Unique storage identifier.
size (Integer): Size of the device in GB.
```

查看introspection得到的磁盘信息，确认sda是不是我们想要的根磁盘。

```bash
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]} ;do openstack baremetal introspection data save $i | jq ".inventory.disks" ;done
```

### 如果sda是我们想要的根磁盘:

```bash
list=(`ironic node-list|grep power|awk '{print $2}'`);for i in ${list[*]};do openstack baremetal node set --property root_device='{"name":"/dev/sda"}' $i;done
```

### 如果sda不是我们想要的根磁盘

那就需要使用wwn来定义根磁盘，手动对每一个node依次执行定义:

```
#wwn
ironic node-update $i add properties/root_device='{"wwn": "xxx"}'
```

然后需要修正logic\_gb，可以重新执行introspection或者手动指定logic\_gb

* 重新执行introspection
  ```
  openstack baremetal introspection bulk start
  ```
* 或者更新logic\_gb
  ```
  ironic node-update <UUID> add properties/local_gb=<NEW VALUE>
  ```

## 4. 部署

在进行自定义部署之前，应该使用简单部署验证以上步骤都没有问题，确保主机可以被正常部署出来。  
现在开始我们的部署之旅 -- 最简单的部署，等待一杯咖啡的时间。

```
openstack overcloud deploy --template
```




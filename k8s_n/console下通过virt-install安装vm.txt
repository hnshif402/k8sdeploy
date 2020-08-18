#### 1 console下通过virt-install安装vm

```
virt-install --name=vm1 --ram 4096 --vcpus=4 \
--disk path=/mydata/data2/vms/vm1/vm1.qcow2,size=15,format=qcow2,bus=virtio \
--location=/mydata/data2/CentOS-8.2.2004-x86_64-minimal.iso --network bridge=virbr0,model=virtio \
--graphics=none --console=pty,target_type=serial \
--extra-args="console=tty0 console=ttyS0"
```



#### 2 复制vm

```
virt-clone --original vm1 --name vm2 --file /mydata/data2/vms/vm2/vm2.qcow2
```



#### 3 冷迁移或跨物理机复制

将192.168.6.43上的已经创建好的虚拟机vm1拷贝到192.168.6.42上

##### 3.1 关闭vm1

````
#virsh shutdown vm1
````

##### 3.2 将vm1的镜像文件拷贝到目标主机

````
#scp /mydata/data2/vms/vm1.qcow2 root@192.168.6.42:/data2/lsf/vms/
````

##### 3.3 将vm1的启动配置文件拷贝到目标主机

```` 
#scp /etc/libvirt/qemu/vm1.xml root@192.168.6.42:/root/
````

##### 3.4 修改192.168.6.42上的vm1.xml配置

```
#vim /root/vm1.xml
将vm1.xml中的镜像目录修改为192.168.6.42上存放的目录
将
<source file='/mydata/data2/vms/vm1.qcow2'/>
修改为
<source file='/data2/lsf/vms/vm1.qcow2'/>
```

##### 3.5 在192.168.6.42中定义虚拟机

```
#virsh define /root/vm1.xml
```

##### 3.6 启动vm1

```
#virsh start vm1
```


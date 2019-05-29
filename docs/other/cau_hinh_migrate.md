# Cấu hình Cold-migrate (SCP - SSH connection)

Cấu hình tạo key nova trên node Compute01
```sh 
[root@compute01 ~]# usermod -s /bin/bash nova
[root@compute01 ~]# su nova 
bash-4.2$ cd /var/lib/nova/
bash-4.2$ ls
buckets  instances  keys  networks  tmp
bash-4.2$ pwd 
/var/lib/nova
bash-4.2$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/nova/.ssh/id_rsa): 
Created directory '/var/lib/nova/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/nova/.ssh/id_rsa.
Your public key has been saved in /var/lib/nova/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:QFzexFdc5dejzL4N61aj+8b91aLCxf6U46tK0M0m8ag nova@compute01
The key's randomart image is:
+---[RSA 2048]----+
|     ......  o..+|
|     ... o. . ...|
|      . . o.   .+|
|       . . *o . o|
|        S +.=+   |
|         o o+  +.|
|        E..o oO =|
|         .o .*==o|
|          .o=BB+o|
+----[SHA256]-----+
bash-4.2$ 
```


Trên node compute01 (node đã có).
Thực hiện với quyền root, scp key pair tới compute node
```sh 
scp /var/lib/nova/.ssh/id_rsa compute02:~/
scp /var/lib/nova/.ssh/authorized_keys compute02:~/
```


Trên node compute02 mới : 
 - Cho phép login với user nova
```sh 
usermod -s /bin/bash nova
```

- Copy keypair vào thư mục nova 
```sh 
mkdir -p /var/lib/nova/.ssh
cp id_rsa /var/lib/nova/.ssh/
cat authorized_keys >> /var/lib/nova/.ssh/authorized_keys
echo 'StrictHostKeyChecking no' >> /var/lib/nova/.ssh/config
cd /var/lib/nova/.ssh/
chown nova:nova id_rsa
chown nova:nova authorized_keys
chown nova:nova config
```

- Kiểm tra việc ssh bằng user nova
```sh
su nova
ssh compute01
```
> Nếu đăng nhập thành công thì tức là chức năng cold-migrate đã sẵn sàng.  

## Kiểm tra việc migrate

- Thực hiện login và controller và source admin environment
```sh
ssh root@192.168.70.11
source admin-openrc
```

- Migrate máy ảo. 
```sh 
nova migrate --poll $VM_ID
```
> Thay $VM_ID với id của máy ảo. 

Sau khi máy ảo chuyển trạng thái Finished thực hiện confirm.  
```sh
nova resize-confirm $VM_ID
```

- Start máy ảo
```sh 
nova start $VM_ID 
```

- Login vào máy ảo vào kiểm tra

# Cấu hình Live migrate (TCP connection)
Sửa thông tin trong libvirt của VM mơi
```sh 
cp /etc/libvirt/libvirtd.conf /etc/libvirt/libvirtd.conf.orig
sed -i 's|#listen_tls = 0|listen_tls = 0|'g /etc/libvirt/libvirtd.conf
sed -i 's|#listen_tcp = 1|listen_tcp = 1|'g /etc/libvirt/libvirtd.conf
sed -i 's|#tcp_port = "16509"|tcp_port = "16509"|'g /etc/libvirt/libvirtd.conf
sed -i 's|#auth_tcp = "sasl"|auth_tcp = "none"|'g /etc/libvirt/libvirtd.conf
cp /etc/sysconfig/libvirtd /etc/sysconfig/libvirtd.orig 
sed -i 's|#LIBVIRTD_ARGS="--listen"|LIBVIRTD_ARGS="--listen"|'g /etc/sysconfig/libvirtd
```

Restart dịch vụ
```sh 
systemctl restart libvirtd
systemctl restart openstack-nova-compute.service
```

Kiểm tra tính năng `live-migrate `
```sh 
nova live-migration $vm-id $compute_node
```

Trong đó : 
    - $vm-id : ID của máy ảo muốn migrate
    - $compute_node : hostname của node compute đích

Thực hiện login vào máy ảo và kiểm tra.


# Tài liệu tham khảo

- https://docs.openstack.org/nova/pike/admin/ssh-configuration.html
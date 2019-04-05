# Install OpenStack sử dụng PackStack

## 0.Giới thiệu

- Packstack là một công cụ cài đặt OpenStack nhanh chóng.
- Packstack được phát triển bởi redhat
- Chỉ hỗ trợ các distro: RHEL, Centos
- Tự động hóa các bước cài đặt và lựa chọn thành phần cài đặt.
- Nhanh chóng dựng được môi trường OpenStack để sử dụng làm PoC nội bộ, demo khách hàng, test tính năng.
- Nhược điểm 1 : Đóng kín các bước cài đối với người mới.
- Nhược điểm 2: Khó bug các lỗi khi cài vì đã được đóng gói cùng với các tool cài đặt tự động (puppet)

## 1. Chuẩn bị môi trường cài đặt 

- Env: VM ESXi 5.5
- Iso: CentOS-7-x86_64-Minimal-1804.iso
- Controller: 
    - HDD: 32G
    - RAM: 4G
    - CPU: 4Cores
    - Network: 4line 
        - 172.16.0.63/20
        - 172.16.4.63/24
        - 10.0.2.63/24
        - null
- Compute: 
    -HDD: 50G
    - RAM: 4G
    - CPU: 4Cores
    - Network: 4line 
        - 172.16.0.64/20
        - 172.16.4.64/24
        - 10.0.2.64/24
        - null

## 2.Mô hình cài đặt 

```
------------+-------API/Managerment------+-- Internet upd packet
      ens160|172.16.4.63           ens160|172.16.4.64
            |                            |
------------+------Provider(FLAT,VLAN)---+----- Internet 
            |                            |
      ens192|172.16.6.63           ens192|172.16.6.64
+-----------+-----------+    +-----------+-----------+
|                       |    |                       |
|    [ Control Node ]   |    |    [ Compute Node ]   |
|                       |    |                       |
+-----------------------+    +-----------------------+
      ens224|10.0.2.63              ens224|10.0.2.64
------------+-------Tenant(Local))--------+---------

``` 

## 3.Cài đặt IP và môi trường 

Bước cài đặt được thực hiện cho tất cả các node

- Cài đặt hostname
    ```sh 
    hostnamectl set-hostname <hostname>
    ```

- Thiết lập IP 
    ```sh
    echo "Setup IP  ens160"
    nmcli c modify ens160 ipv4.addresses 172.16.6.0/24
    nmcli c modify ens160 ipv4.method manual
    nmcli con mod ens160 connection.autoconnect yes

    echo "Setup IP  ens192"
    nmcli c modify ens192 ipv4.addresses 172.16.4.0
    nmcli c modify ens192 ipv4.gateway 172.16.10.1
    nmcli c modify ens192 ipv4.dns 8.8.8.8
    nmcli c modify ens192 ipv4.method manual
    nmcli con mod ens192 connection.autoconnect yes

    echo "Setup IP  ens224"
    nmcli c modify ens224 ipv4.addresses 10.0.0.0/24
    nmcli c modify ens224 ipv4.method manual
    nmcli con mod ens224 connection.autoconnect yes
    ```

- Disable firewalld và SElinux
    ```sh
    sudo systemctl disable firewalld
    sudo systemctl stop firewalld
    sudo systemctl disable NetworkManager
    sudo systemctl stop NetworkManager
    sudo systemctl enable network
    sudo systemctl start network

    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
    ```
    
- Khai báo repos cho OpenStack Queens
    ```sh 
    yum install -y python-setuptools
    sudo yum install -y centos-release-openstack-queens
    yum update -y

    sudo yum install -y wget crudini fping
    yum install -y openstack-packstack

    yum install -y epel-release
    sudo yum install -y byobu 
    ```

## Cài đặt Packstack

Đứng trên node controller thực hiện 

- Khởi động tty byobu (tmux, screen)
    ```sh 
    byobu
    ```

- Tạo file packstack.sh
    ```sh 
    cat << EOF >> packstack.sh
    packstack packstack --gen-answer-file=/root/rdo_answer.txt \
        --allinone \
        --default-password=Welcome123 \
        --os-cinder-install=y \
        --os-ceilometer-install=y \
        --os-trove-install=n \
        --os-ironic-install=n \
        --os-swift-install=n \
        --os-panko-install=y \
        --os-heat-install=y \
        --os-magnum-install=n \
        --os-aodh-install=y \
        --os-neutron-ovs-bridge-mappings=extnet:br-ex \
        --os-neutron-ovs-bridge-interfaces=br-ex:ens192 \
        --os-neutron-ovs-bridges-compute=br-ex \
        --os-neutron-ml2-type-drivers=vxlan,flat \
        --os-controller-host=172.16.4.63 \
        --os-compute-hosts=172.16.4.64,172.16.4.65 \
        --os-neutron-ovs-tunnel-if=eth0 \
        --provision-demo=n

    packstack --answer-file rdo_answer.txt
    EOF
    ```

- Khởi chạy file packstack 
    ```sh 
    bash packstack.sh
    ```

    > Thời gian cài đặt 2-4 tiếng đối với cụm LAB (Có thể nhanh hơn nếu tốc độ mạng ổn định)

- Cài đặt hoàn tất
    ```sh 
    **** Installation completed successfully ******

    Additional information:
    * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
    * File /root/keystonerc_admin has been created on OpenStack client host 172.16.68.201. To use the command line tools you need to source the file.
    * To access the OpenStack Dashboard browse to http://172.16.4.63/dashboard .
    Please, find your login credentials stored in the keystonerc_admin in your home directory.
    * Because of the kernel update the host 172.16.68.202 requires reboot.
    * Because of the kernel update the host 172.16.68.203 requires reboot.
    * The installation log file is available at: /var/tmp/packstack/20180309-001110-LD0XmO/openstack-setup.log
    * The generated manifests are available at: /var/tmp/packstack/20180309-001110-LD0XmO/manifests
    ```

# Multi Region 

# Khái niệm Multi Region trong Openstack
---
## Tổng quan

Trong Openstack có rất nhiều khái niệm dễ hiểu lầm như Domain, Region, Multi Site. Sau đây mình sẽ làm rõ các khái niệm

### Domain

Một hệ thống Cloud Openstack có thể được sử dụng bởi nhiều TỔ CHỨC, DOANH NGHIỆP, CÁ NHÂN, v.v, đồng thời openstack cũng cho phép tạo project có tên giống nhau. 

Ví dụ, Phòng Kỹ Thuật của Nhân Hòa sử dụng project có tên là phong_ky_thuat, Phòng Kỹ Thuật của công ty Cloud365 cũng đặt tên project là phong_ky_thuat, khi đó trên hệ thống openstack sẽ có 2 project "phong_ky_thuat" nhưng lại thuộc sử hữu của 2 công ty khác nhau, điều này sẽ gây lẫn cho người quản trị. 

Để hệ thống Openstack trở nên rõ ràng hơn, Keystone bổ sung khái niệm "Domain" nhằm cô lập các project giữa các tổ chức. Tức mỗi doanh nghiệp, tổ chức, cá nhân sẽ có 1 domain riêng, tại domain mỗi tổ chức, họ chỉ có thể thấy các project và user mà họ sở hữu, độc lập với "Domain" khác.

Quay lại ví dụ trên, công ty Nhân hoà sẽ sở hữu domain 'nhanhoa', công ty Cloud365 sẽ sở hữu domain 'cloud365'. Khi đó người quản trị của Nhân Hòa chỉ thấy các project, user thuộc sử hữu domain 'nhanhoa', tương tự với công ty Cloud365

![](../../images/multi-region/multi1.png)

### Region

Khi hệ thống Cloud Openstack phát triển tới mức độ nhất định, việc triển khai 1 cụm Openstack là không đủ, các doanh nghiệp sẽ tính đến việc triển khai nhiều hệ thống Openstack tại nhiều địa điểm khác nhau phục vụ cho bài toán tối ưu vị trí địa lý.

Ví dụ, Công ty Nhân Hòa đã có một cụm Openstack tại Hà Nội, Công ty quyết định triển khai một cụm Openstack khác tại Đà năng, một cụm Openstack Hồ Chí Minh để cung cấp dịch vụ tốt nhất cho khách hàng tại Hà Nội, Đà Năng, Hồ Chí Minh, đồng thời có yêu cầu 3 cụm Openstack phải sử dụng chung một hệ thống xác thực để phục vụ cho việc quản trị người dùng.

Khái niệm Region hay Multi Region sẽ được áp dụng cho ví dụ trên, khi chúng ta muốn triển khai nhiều cụm openstack khác nhau nhưng lại muốn sử dụng chúng Keystone.

![](../../images/multi-region/multi2.png)

### Multi Site

Khái niệm Multi Site cũng gần giống với khái niệm Region nhưng khác biệt điểm, chúng ta sẽ có nhiều cụm Openstack nhưng sẽ không chia sẻ dịch vụ Keystone (Mỗi cụm đều chạy độc lập).

### Bổ sung khái niệm Multi Region trong Openstack

Sử dụng lại ví dụ, Công ty Nhân Hòa sẽ triển khai tổng cộng 3 Region tại Hà Nội, Đà Nẵng, Hồ Chí Minh chia sẻ chung hệ thống xác thực. Để hiện thực yêu cầu trên với Openstack, chúng ta sẽ triển khai 3 cụm openstack tuy nhiên sẽ sử dụng chung project Keystone. Lưu ý, 3 cụm Openstack sẽ nằm trên 1 DC Khác nhau.

Để chia sẻ Keystone giữa 3 cụm Openstack ta sẽ có 3 sự lựa chọn:
- Centralized Keystone DB: Tức ta sẽ có 1 DB trung tâm của Keystone, các Region sẽ kết nối với Database thông qua đường WAN. (Không sử dụng Keystone DB)
- Asyncronous Keystone DB: Mỗi Region sẽ có 1 DB tuy nhiên sẽ duy trì 1 Master (Cho phép đọc ghi) còn lại sẽ là Slave (Chỉ đọc). Khi đó dữ liệu sẽ đồng bộ giữa các DB tại các Region
- Syncronous (Clustered) Keystone DB: Sử dụng MySQL/MariaDB Galera Cluster, Keystone database tại 3 Region sẽ được đồng bộ.

Các tùy chọn sẽ có ưu nhược điểm khác nhau.

![](../../images/multi-region/multi3.png)

## Nguồn

https://serverascode.com/2017/05/09/openstack-multisite-multicloud.html

https://www.cnblogs.com/nongchaoer/p/6297834.html


# Triển khai Multi Region trong Openstack
---
## Chuẩn bị

- Triển khai 2 cụm Openstack theo docs manual [Cài đặt](../setup/manual_install.md)

![](../../images/multi-region/Ip_planning.png)

## Triển khai

Lưu ý:
- Theo docs manual, Region mặc định sẽ là RegionOne, vì vậy chúng ta sẽ không phải cấu hình 2 node Controller 80.83 và compute 80.84
- Trong bài, chỉ cấu hình 2 node controller 80.83 và compute 80.86 sang thành RegionTwo, chia sẻ Keystone, Horizon với RegionOne
- Keystone chia sẻ giữa 2 cụm Openstack sẽ nằm trên Controller 80.83

### Bước 1: Tạo RegionTwo trên Controller 80.83

> Thực hiện trên Controller 80.83

Tạo mới Region 
> Lưu ý: Không `source admin-openrc` khi chạy keystone-manage

```
[root@mult-ctl1 ~]$ keystone-manage bootstrap --bootstrap-password passla123 \
  --bootstrap-admin-url http://192.168.80.83:5000/v3/ \
  --bootstrap-internal-url http://192.168.80.83:5000/v3/ \
  --bootstrap-public-url http://192.168.80.83:5000/v3/ \
  --bootstrap-region-id RegionTwo
```

Kiểm tra các Region trên hệ thống
```
[root@mult-ctl1 ~]$ source admin-openrc
[root@mult-ctl1 ~(admin-openrc)]$ openstack region list
+-----------+---------------+-------------+
| Region    | Parent Region | Description |
+-----------+---------------+-------------+
| RegionOne | None          |             |
| RegionTwo | None          |             |
+-----------+---------------+-------------+
```

Sau khi sử dụng keystone khởi tạo RegionTwo, keystone sẽ tự động tạo thêm endpoint identity mới

```
[root@mult-ctl1 ~(admin-openrc)]$ openstack endpoint list --service identity
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                         |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| 009af663df69409d8c86bb9125217b7c | RegionTwo | keystone     | identity     | True    | admin     | http://192.168.80.83:5000/v3/ |
| 0cc2b298947a4e96a8d4acf3bfbe6837 | RegionTwo | keystone     | identity     | True    | internal  | http://192.168.80.83:5000/v3/ |
| 4709d158e0fd45d98d2f2a1949e7877f | RegionTwo | keystone     | identity     | True    | public    | http://192.168.80.83:5000/v3/ |
| 6317fc49aa2b43b4ae28a7683e8e9943 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.80.83:5000/v3/ |
| f0384c9359c14afcbd6d8b5bddc15c90 | RegionOne | keystone     | identity     | True    | public    | http://192.168.80.83:5000/v3/ |
| f75e70e338be41cc8dbb7558ff08c249 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.80.83:5000/v3/ |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
```

### Bước 2: Khởi tạo các endpoint RegionTwo cho nova, cinder, glance, neutron (Thực hiện trên CTL 80.83)

Lưu ý các endpoint tạo cho RegionTwo sẽ sử dụng IP CTL 192.168.80.85
```sh 
[root@mult-ctl1 ~]$ source admin-openrc
[root@mult-ctl1 ~(admin-openrc)]$ 
```

Tạo các endpoint 
```
openstack endpoint create --region RegionTwo image public http://192.168.80.85:9292
openstack endpoint create --region RegionTwo image admin http://192.168.80.85:9292
openstack endpoint create --region RegionTwo image internal http://192.168.80.85:9292

openstack endpoint create --region RegionTwo network public http://192.168.80.85:9696
openstack endpoint create --region RegionTwo network internal http://192.168.80.85:9696
openstack endpoint create --region RegionTwo network admin http://192.168.80.85:9696

openstack endpoint create --region RegionTwo compute public http://192.168.80.85:8774/v2.1
openstack endpoint create --region RegionTwo compute admin http://192.168.80.85:8774/v2.1
openstack endpoint create --region RegionTwo compute internal http://192.168.80.85:8774/v2.1

openstack endpoint create --region RegionTwo placement public http://192.168.80.85:8778
openstack endpoint create --region RegionTwo placement admin http://192.168.80.85:8778
openstack endpoint create --region RegionTwo placement internal http://192.168.80.85:8778

openstack endpoint create --region RegionTwo volumev2 public http://192.168.80.85:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionTwo volumev2 internal http://192.168.80.85:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionTwo volumev2 admin http://192.168.80.85:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionTwo volumev3 public http://192.168.80.85:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionTwo volumev3 internal http://192.168.80.85:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionTwo volumev3 admin http://192.168.80.85:8776/v3/%\(project_id\)s
```

Kiểm tra

```
[root@mult-ctl1 ~(admin-openrc)]$ openstack endpoint list --region RegionTwo
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                       |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------------------+
| 009af663df69409d8c86bb9125217b7c | RegionTwo | keystone     | identity     | True    | admin     | http://192.168.80.83:5000/v3/               |
| 0cc2b298947a4e96a8d4acf3bfbe6837 | RegionTwo | keystone     | identity     | True    | internal  | http://192.168.80.83:5000/v3/               |
| 1cef864c85e04bb6aa1345efb1ef2dd4 | RegionTwo | nova         | compute      | True    | public    | http://192.168.80.85:8774/v2.1              |
| 21ed22d548d4422f97883e3341fd6ced | RegionTwo | nova         | compute      | True    | admin     | http://192.168.80.85:8774/v2.1              |
| 286b82697852401c9498695892a7d388 | RegionTwo | cinderv3     | volumev3     | True    | admin     | http://192.168.80.85:8776/v3/%(project_id)s |
| 3d85c57f745245bf94ab37e072ad9fd2 | RegionTwo | nova         | compute      | True    | internal  | http://192.168.80.85:8774/v2.1              |
| 4709d158e0fd45d98d2f2a1949e7877f | RegionTwo | keystone     | identity     | True    | public    | http://192.168.80.83:5000/v3/               |
| 53cd8b0aa4504a729456c32e8041e20a | RegionTwo | cinderv3     | volumev3     | True    | internal  | http://192.168.80.85:8776/v3/%(project_id)s |
| 56f20dbb82be4b1b8f5b9bc62ad7d401 | RegionTwo | cinderv3     | volumev3     | True    | public    | http://192.168.80.85:8776/v3/%(project_id)s |
| 5cae9e4cff9e43cb8c9d092d6b204658 | RegionTwo | neutron      | network      | True    | internal  | http://192.168.80.85:9696                   |
| 6dedfcfef07147e4b5547e8e43bc1d93 | RegionTwo | placement    | placement    | True    | public    | http://192.168.80.85:8778                   |
| 732c5b2e12974a35b534566b41f95254 | RegionTwo | cinderv2     | volumev2     | True    | internal  | http://192.168.80.85:8776/v2/%(project_id)s |
| 7aeb05e306fe44508e0334c149428e02 | RegionTwo | glance       | image        | True    | admin     | http://192.168.80.85:9292                   |
| 8758a4edbc1248b3ad8108f63985affe | RegionTwo | cinderv2     | volumev2     | True    | admin     | http://192.168.80.85:8776/v2/%(project_id)s |
| 90abe4b694a24edea9059b3125daf24e | RegionTwo | placement    | placement    | True    | internal  | http://192.168.80.85:8778                   |
| a98401bd172649b8b5c21bb26152f40b | RegionTwo | neutron      | network      | True    | admin     | http://192.168.80.85:9696                   |
| c309a0ccafe24ed2ad2bd9f31ad49cde | RegionTwo | cinderv2     | volumev2     | True    | public    | http://192.168.80.85:8776/v2/%(project_id)s |
| cfcfb92d8f974260853da7abbda06e6d | RegionTwo | neutron      | network      | True    | public    | http://192.168.80.85:9696                   |
| d2aab1e0a2ad4fedb654da4a9e18dc30 | RegionTwo | placement    | placement    | True    | admin     | http://192.168.80.85:8778                   |
| f63ef7faacc74b67be231ab90e4e6038 | RegionTwo | glance       | image        | True    | public    | http://192.168.80.85:9292                   |
| fa01094bc6e1498eb52388af566ee865 | RegionTwo | glance       | image        | True    | internal  | http://192.168.80.85:9292                   |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------------------+
```

### Bước 3: Tạo admin openstack resource trên cho cả 2 node CTL 80.83 và 80.85

```
cat << EOF >> admin-openrc-r2
export OS_REGION_NAME=RegionTwo
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=passla123
export OS_AUTH_URL=http://192.168.80.83:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(admin-openrc)]\$ '
EOF
```

### Bước 4: Chỉnh sửa dịch vụ Glance

> Truy cập controller 80.85

Với Glance, chúng ta sẽ chỉnh sửa mục xác thực keystone về CTL 80.83

Mục `[keystone_authtoken]`
- `auth_uri`, `auth_url` về CTL 80.83
- Chỉnh sửa `region_name` về `RegionTwo`.

Lưu ý, sẽ chỉnh sửa 2 file `glance-api.conf` và `glance-registry.conf`

Tại file: `/etc/glance/glance-api.conf`

```
[DEFAULT]
bind_host = 192.168.80.85
registry_host = 192.168.80.85
[cors]
[database]
connection = mysql+pymysql://glance:passla123@192.168.80.85/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
#auth_uri = http://192.168.80.85:5000
#auth_url = http://192.168.80.85:5000
auth_uri = http://192.168.80.83:5000
auth_url = http://192.168.80.83:5000
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = passla123
region_name = RegionTwo
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```

Tại file: `/etc/glance/glance-registry.conf`

```
[DEFAULT]
bind_host = 192.168.80.85
[database]
connection = mysql+pymysql://glance:passla123@192.168.80.85/glance
[keystone_authtoken]
#auth_uri = http://192.168.80.85:5000
#auth_url = http://192.168.80.85:5000
auth_uri = http://192.168.80.83:5000
auth_url = http://192.168.80.83:5000
memcached_servers = 192.168.80.85
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = passla123
region_name = RegionTwo
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
```
![](../../images/multi-region/glance.png)

Khởi động lại dịch vụ
```sh 
[root@mult-ctl2 ~]# systemctl restart openstack-glance-api.service openstack-glance-registry.service
```

Kiểm tra trên CTL1 hoặc CTL2 với điều kiện xác thực `admin-openrc-r2`
```sh 
[root@mult-ctl1 ~(admin-openrc-r2)]$ openstack --debug image list --os-region-name RegionTwo
```
Kết quả
![](../../images/multi-region/glance2.png)

Đứng trên CTL 80.85 tiến hành xác thực qua keystone 80.83 trên RegionTwo
```sh 
[root@mult-ctl2 ~]# source admin-openrc-r2
```

Upload image mới lên
```
[root@mult-ctl2 ~(admin-openrc-r2)]$ openstack image create "cirrosr2" --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

Kiểm tra image trên Node 80.83

```
[root@mult-ctl1 ~]# openstack image list --os-region-name RegionTwo
+--------------------------------------+----------+--------+
| ID                                   | Name     | Status |
+--------------------------------------+----------+--------+
| ecb55f8d-7262-468c-9368-586ea4daf3a0 | cirros   | active |
| 14a730c6-e1d4-477d-aea5-8d2b8ac36d45 | cirrosr2 | active |
+--------------------------------------+----------+--------+
```

### Bước 5: Chỉnh sửa dịch vụ Nova

Với Nova, ta cần chỉnh sửa lại settings trên CTL 80.85, COM1 80.86

> Thực hiện trên CTL 80.85

Với dịch vụ Nova, chúng ta sẽ chỉnh sửa lại mục các mục `[keystone_authtoken]`, `[neutron]`, `[placement]`, `[cinder]` tại `/etc/nova/nova.conf`.

Mục `[cinder]`:
- Chỉnh sửa `os_region_name` về `RegionTwo`

Mục `[keystone_authtoken]`:
- Chỉnh sửa `auth_url` về CTL 80.83 
- `region_name` về RegionTwo

Mục `[neutron]`:
- Chỉnh sửa `auth_url` về CTL 80.83 
- `region_name` về RegionTwo

Mục `[placement]`
- `os_region_name` về RegionTwo
- `auth_url` về CTL 80.83

Cấu hình mẫu

```
[DEFAULT]
my_ip = 192.168.80.85
enabled_apis = osapi_compute,metadata
use_neutron = True
osapi_compute_listen=192.168.80.85
metadata_host=192.168.80.85
metadata_listen=192.168.80.85
metadata_listen_port=8775    
firewall_driver = nova.virt.firewall.NoopFirewallDriver
allow_resize_to_same_host=True
notify_on_state_change = vm_and_task_state
transport_url = rabbit://openstack:passla123@192.168.80.85:5672
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:passla123@192.168.80.85/nova_api
[barbican]
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = 192.168.80.85:11211
[cells]
[cinder]
os_region_name = RegionTwo
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
connection = mysql+pymysql://nova:passla123@192.168.80.85/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://192.168.80.85:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
#auth_url = http://192.168.80.85:5000/v3
auth_url = http://192.168.80.83:5000/v3 
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = passla123
region_name = RegionTwo
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://192.168.80.85:9696
#auth_url = http://192.168.80.85:35357
auth_url = http://192.168.80.83:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionTwo
project_name = service
username = neutron
password = passla123
service_metadata_proxy = true
metadata_proxy_shared_secret = passla123
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
os_region_name = RegionTwo
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
#auth_url = http://192.168.80.85:5000/v3
auth_url = http://192.168.80.83:5000/v3
username = placement
password = passla123
[quota]
[rdp]
[remote_debug]
[scheduler]
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
novncproxy_host=192.168.80.85
enabled = true
vncserver_listen = 192.168.80.85
vncserver_proxyclient_address = 192.168.80.85
novncproxy_base_url = http://192.168.80.85:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
```

Khởi động lại dịch vụ

```
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-consoleauth.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

Kiểm tra 
```sh 
GET call to compute for http://192.168.80.85:8774/v2.1/flavors/detail used request id req-62e730eb-e8df-4d20-be53-b153c232df40

clean_up ListServer: 
END return value: 0
[root@mult-ctl1 ~(admin-openrc-r2)]$ openstack --debug server list --os-region-name RegionTwo
```


> Thực hiện trên COM 80.86

Với dịch vụ nova chỉnh sửa `/etc/nova/nova.conf` tại COM 80.86, chúng ta sẽ chỉnh sửa lại `[cinder]`, `[keystone_authtoken]`, `[neutron]`, `[placement]`

Mục `[cinder]`:
- Chỉ định sử dụng `cinder` service của RegionTwo (os_region_name = RegionTwo)

Mục `[keystone_authtoken]`:
- Chỉnh sửa `auth_url` về CTL 80.83 
- `region_name` về RegionTwo

Mục `[neutron]`:
- Chỉnh sửa `auth_url` về CTL 80.83 
- `region_name` về RegionTwo

Mục `[placement]`:
- Chỉnh sửa `auth_url` về CTL 80.83
- Sửa `os_region_name` về RegionTwo

Cấu hình mẫu

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:passla123@192.168.80.85:5672
my_ip = 192.168.80.86
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
[barbican]
[cache]
[cells]
[cinder]
os_region_name = RegionTwo
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://192.168.80.85:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
#auth_url = http://192.168.80.85:5000/v3
auth_url = http://192.168.80.83:5000/v3
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = passla123
region_name = RegionTwo
[libvirt]
# egrep -c '(vmx|svm)' /proc/cpuinfo = 0
virt_type = qemu
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://192.168.80.85:9696
#auth_url = http://192.168.80.85:35357
auth_url = http://192.168.80.83:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionTwo
project_name = service
username = neutron
password = passla123
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
os_region_name = RegionTwo
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
#auth_url = http://192.168.80.85:5000/v3
auth_url = http://192.168.80.83:5000/v3
username = placement
password = passla123
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = 192.168.80.86
novncproxy_base_url = http://192.168.80.85:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
```

Khởi động lại dịch vụ tại node COM

```
systemctl restart libvirtd.service openstack-nova-compute
```

Lưu ý: Kiểm tra log tại Nova Compute

```
[root@compute01 ~]# cat /var/log/nova/nova-compute.log | grep 'placement'

2019-04-11 10:36:33.694 14368 ERROR nova.scheduler.client.report [req-6c9a2cb8-b840-4345-bb9e-088068c8568f - - - - -] [req-5d53d9f5-99a8-4ce3-9579-92d93ec5f31f] Failed to retrieve resource provider tree from placement API for UUID 52517eca-5525-4905-aaa1-fed226b3366f. Got 401: {"error": {"message": "The request you have made requires authentication.", "code": 401, "title": "Unauthorized"}}.
``` 

- Nếu xuất hiện, kiểm tra config tại nova controller và nova compute, sau đó khởi động os CTL 80.85 vầ 80.86 thuộc RegionTwo (do db hoặc cache)


> Trở lại CTL 80.83

Kiểm tra dịch vụ bằng câu lệnh
```
[root@mult-ctl1 ~(admin-openrc-r2)]$ openstack compute service list --os-region-name RegionTwo
+----+------------------+-----------+----------+---------+-------+----------------------------+
| ID | Binary           | Host      | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
|  1 | nova-conductor   | mult-ctl2 | internal | enabled | up    | 2019-06-19T04:29:59.000000 |
|  2 | nova-scheduler   | mult-ctl2 | internal | enabled | up    | 2019-06-19T04:29:53.000000 |
|  3 | nova-consoleauth | mult-ctl2 | internal | enabled | up    | 2019-06-19T04:30:02.000000 |
|  8 | nova-compute     | mult-com2 | nova     | enabled | up    | 2019-06-19T04:29:57.000000 |
+----+------------------+-----------+----------+---------+-------+----------------------------+
[root@mult-ctl1 ~(admin-openrc-r2)]$
```

### Bước 6: Chỉnh sửa dịch vụ Cinder

> Thực hiện trên CTL 80.85

Với dịch vụ Cinder, ta sẽ chỉnh sửa các mục `[keystone_authtoken]` tại `/etc/cinder/cinder.conf`

Mục `[keystone_authtoken]`
- Sửa `auth_uri`, `auth_url` về CTL 80.83
- `region_name` về RegionTwo

Cấu hình mẫu
```
[DEFAULT]
transport_url = rabbit://openstack:passla123@192.168.80.85
auth_strategy = keystone
my_ip = 192.168.80.85
enabled_backends = lvm
glance_api_servers = http://192.168.80.85:9292
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:passla123@192.168.80.85/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
#auth_uri = http://192.168.80.85:5000
#auth_url = http://192.168.80.85:35357
auth_uri = http://192.168.80.83:5000
auth_url = http://192.168.80.83:35357
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = passla123
region_name = RegionTwo
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[profiler]
[service_user]
[ssl]
[vault]
```

Khởi động lại dịch vụ
```
systemctl restart openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
```

Kiểm tra dịch vụ

> Trở về CTL 80.83, kiểm tra service

```
[root@mult-ctl1 ~(admin-openrc)]$ openstack volume service list --os-region-name RegionTwo
+------------------+---------------+------+---------+-------+----------------------------+
| Binary           | Host          | Zone | Status  | State | Updated At                 |
+------------------+---------------+------+---------+-------+----------------------------+
| cinder-scheduler | mult-ctl2     | nova | enabled | up    | 2019-06-19T04:34:13.000000 |
| cinder-volume    | mult-ctl2@lvm | nova | enabled | up    | 2019-06-19T04:34:50.000000 |
+------------------+---------------+------+---------+-------+----------------------------+
[root@mult-ctl1 ~(admin-openrc)]$ 
```

Trở về Horion thử tạo Volume + Volume có chưa Image và kiểm tra log.

### Bước 7: Chỉnh sửa dịch vụ Neutron

Với Neutron, ta cần chỉnh sửa lại settings trên CTL 80.85, COM1 80.86

> Thực hiện trên CTL 80.85

Chỉnh sửa các mục `[keystone_authtoken]`, `[nova]` tại `/etc/neutron/neutron.conf`

Mục `[keystone_authtoken]`
- Sửa `auth_uri`, `auth_url` về CTL 80.83
- `region_name` về RegionTwo

Mục `[nova]`
- `auth_url` về  CTL 80.83
- `region_name` = `RegionTwo`

Cấu hình mẫu
```
[DEFAULT]
bind_host = 192.168.80.85
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:passla123@192.168.80.85
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
dhcp_agents_per_network = 2
[agent]
[cors]
[database]
connection = mysql+pymysql://neutron:passla123@192.168.80.85/neutron
[keystone_authtoken]
#auth_uri = http://192.168.80.85:5000
#auth_url = http://192.168.80.85:35357
auth_uri = http://192.168.80.83:5000
auth_url = http://192.168.80.83:35357
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = passla123
region_name = RegionTwo
[matchmaker_redis]
[nova]
#auth_url = http://192.168.80.85:35357
auth_url = http://192.168.80.83:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionTwo
project_name = service
username = nova
password = passla123
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues = true
rabbit_ha_queues = true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
```

Khởi động lại dịch vụ `nova` và `neutron`

```
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-consoleauth.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl restart neutron-server.service neutron-linuxbridge-agent.service \
neutron-l3-agent.service
```

> Thực hiện trên COM1 80.86

Chỉnh sửa lại các mục `[keystone_authtoken]` tại `/etc/neutron/neutron.conf`
- Sửa `auth_uri`, `auth_url` về CTL 80.83
- `region_name` về RegionTwo

```
[DEFAULT]
transport_url = rabbit://openstack:passla123@192.168.80.85:5672
auth_strategy = keystone
[agent]
[cors]
[database]
[keystone_authtoken]
#auth_uri = http://192.168.80.85:5000
#auth_url = http://192.168.80.85:35357
auth_uri = http://192.168.80.83:5000
auth_url = http://192.168.80.83:35357
memcached_servers = 192.168.80.85:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = passla123
region_name = RegionTwo
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]

```

Khởi động lại dịch vụ
```
systemctl restart neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

Kiểm tra

> Trở về CTL 80.83

```
[root@mult-ctl1 ~(admin-openrc)]$ openstack network agent list --os-region-name RegionTwo
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host      | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
| 1776c225-bb36-4337-a5da-bd5d2239c928 | Linux bridge agent | mult-ctl2 | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 1ae77c12-0918-4df8-abb6-bd43b9db6517 | Metadata agent     | mult-com2 | None              | :-)   | UP    | neutron-metadata-agent    |
| 6465c0f0-90de-41d1-a7ab-75c99ad08764 | DHCP agent         | mult-com2 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| b734222f-4895-4930-a71b-b82dae54a186 | L3 agent           | mult-ctl2 | nova              | :-)   | UP    | neutron-l3-agent          |
| c90bcd55-8a07-4b7d-bf03-3940a28e41d7 | Linux bridge agent | mult-com2 | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+-----------+-------------------+-------+-------+---------------------------+
[root@mult-ctl1 ~(admin-openrc)]$ 
```

### Bước 8: Kiểm tra lại Region

![](../../images/multi-region/dashboard.png)

Truy cập RegionOne, thử tạo VM với volume cinder


Truy cập RegionTwo, thử tạo VM với volume cinder

Nếu xuất hiện lỗi tại Node Compute

```
[root@compute01 ~]# cat /var/log/nova/nova-compute.log | grep ERROR

2019-04-11 11:40:05.363 15299 ERROR nova.compute.manager [instance: 5350c69f-24de-4345-9556-0cc92faa3ef2] BuildAbortException: Build of instance 5350c69f-24de-4345-9556-0cc92faa3ef2 aborted: Invalid input received: Invalid image identifier or unable to access requested image. (HTTP 400) (Request-ID: req-1e69aa25-4f63-477d-a8a5-678ebf1bb869)
```

Kiểm tra lại cấu hình cinder:
- Tại Controller, có thể thiếu `glance_api_servers` section `[glance_api_servers]` (`/etc/cinder/cinder.conf`)
- Tại Compute, có thể thiếu `os_region_name` tại section `[cinder]` (`/etc/nova/nova.conf`)


Nếu xuất hiện lỗi tại Node Compute
```
2019-04-11 10:58:40.625 14019 ERROR nova.compute.manager [instance: 66eef324-058d-443e-afa6-8893f183a7db] PortBindingFailed: Binding failed for port 68e62053-fed2-4bd8-b3a8-0755012774ad, please check neutron logs for more information.
```
Kiểm tra lại cấu hình Neutron:
- Dịch vụ Neutron đã cấu hình không chính xác, kiểm tra lại service neutron

### Bước 9: Redirect Horizon cụm 2 về horizon cụm 1 

Login vào Server 192.168.80.85

Chỉnh sửa file direct
```sh 
mv /var/www/html/index.{html,html.bk}
filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://192.168.80.83/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF
```
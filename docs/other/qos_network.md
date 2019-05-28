# 1.Network QoS
https://en.wikipedia.org/wiki/Quality_of_service

https://docs.openstack.org/neutron/pike/admin/config-qos.html


Valid DSCP mark values are even numbers between 0 and 56, except 2-6, 42, 44, and 50-54. 

# Một vài định nghĩa cần nắm

- [Bandwidth_limit]()
- [DSCP Marking](http://help.sonicwall.com/help/sw/eng/5800/25/9/0/content/Ch78_Firewall_Managing_QoS.088.08.html)
- [Minimum_bandwidth]()

- [Khái niệm bổ sung DSCP Marking](https://vnpro.vn/thu-vien/cac-cong-cu-de-danh-dau-trong-qos-2163.html)


## QoS in OpenStack using Linuxbridge 
![](http://i.imgur.com/tkEMQBd.png)

![](http://i.imgur.com/OgllhFX.png)

### Mô hình cài đặt 

Sử dụng 1 trong các [mô hình sau](https://github.com/uncelvel/tutorial-openstack/#c%C3%A0i-%C4%91%E1%BA%B7t-openstack) 

> Lưu ý hệ thống cài đặt sử dụng LinuxBridge

###  Controller 
Chỉnh sửa `/etc/neutron/neutron.conf`
```sh 
[DEFAULT] 
service_plugins = neutron.services.qos.qos_plugin.QoSPlugin
```

Chỉnh sửa `/etc/neutron/plugins/ml2/ml2_conf.ini`
```sh 
[ml2]
extension_drivers = port_security, qos
```

Chỉnh sửa `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```sh 
[agent]
extensions = qos
```

**Option** nếu muốn set QoS cho Floating IP `/etc/neutron/l3_agent.ini`
```sh 
[agent]
extensions = fip_qos 
```

**Option** Cho phép user tự tạo QoS rule `/etc/neutron/policy.json`
```sh 
"get_policy": "rule:regular_user",
"create_policy": "rule:regular_user",
"update_policy": "rule:regular_user",
"delete_policy": "rule:regular_user",
"get_policy_bandwidth_limit_rule": "rule:regular_user",
"create_policy_bandwidth_limit_rule": "rule:regular_user",
"delete_policy_bandwidth_limit_rule": "rule:regular_user",
"update_policy_bandwidth_limit_rule": "rule:regular_user",
"get_rule_type": "rule:regular_user",
```

Restart lại service 
```sh 
systemctl restart neutron-server.service neutron-linuxbridge-agent.service neutron-l3-agent.service
```

### Compute 

Bổ sung cấu hình `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```sh 
[agent]
extensions = qos
```

Restart lại service 
```sh 
systemctl restart neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

> Chú ý : Priority về Qos của Port sẽ cao hơn network. Trong mọi trường hợp, Qos cho port sẽ ghi đè Qos cho Network

### Tạo policy rule 

Tạo policy 
```sh
openstack network qos policy create Policy100MB --share
```

Tạo rule 
```sh 
openstack network qos rule create --type bandwidth-limit --max-kbps 100000 --max-burst-kbits 100000 --egress Policy100MB
openstack network qos rule create --type bandwidth-limit --max-kbps 100000 --max-burst-kbits 100000 --ingress Policy100MB
```
Trong đó 
- max-kbps : Băng thông tối đa bị giới hạn của máy ảo 
- max-burst-kbits : Lượng dữ liệu có thể vượt ngưỡng
- egress : Giới hạn băng thông truyền ra
- ingress : Giới hạn băng thông truyền vào

https://www.juniper.net/documentation/en_US/junos/topics/concept/policer-mx-m120-m320-burstsize-determining.html

## Các thao tác trên CLI 
```sh 
OPENSTACK 
network qos policy create
network qos policy delete
network qos policy list
network qos policy set 
network qos policy show
network qos rule create 
network qos rule delete
network qos rule list
network qos rule set
network qos rule show
network qos rule type list
network qos rule type show
```

# Hướng dẫn cấu hình message Openstack đối với mô hình HA
---
## Chuẩn bị
- Cụm Openstack HA 3 node CTR và 1 node COM
- Thực hiện theo thứ tự Glance, Cinder, Neutron, Nova, Keystone

## Glance
- Kiểm tra Exchange `glance` trước khi thực hiện, nếu tồn tại => Xóa Exchange `glance`
- Thực hiện lần lượt từ CTR1 tới CTR3
- Tạo file backup
    ```
    cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
    cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak
    ```
- Cấu hình (Bổ sung trên cả 2 file glance-api.conf và glance-registry)
    ```
    vi /etc/glance/glance-api.conf
    vi /etc/glance/glance-registry.conf
    ```
    - Nội dung
        ```
        [oslo_messaging_notifications]
        driver = messagingv2

        [oslo_messaging_rabbit]
        rabbit_ha_queues = true
        rabbit_retry_interval = 1
        rabbit_retry_backoff = 2
        amqp_durable_queues= true
        ```
- Khởi động lại dịch vụ
    ```
    systemctl restart openstack-glance-api.service openstack-glance-registry.service
    ```
- Thử tạo image và check queue notification.info

## Cinder
- Thực hiện lần lượt từ CTR1 tới CTR3

- Tạo file backup
    ```
    cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
    ```
- Cấu hình:
    ```
    vi /etc/cinder/cinder.conf
    ```
    - Bổ sung
        ```
        [DEFAULT]
        ...
        rpc_backend = rabbit
        control_exchange = cinder

        [oslo_messaging_notifications]
        driver = messagingv2

        [oslo_messaging_rabbit]
        rabbit_retry_interval = 1
        rabbit_retry_backoff = 2
        amqp_durable_queues = true
        rabbit_ha_queues = true
        ```
- Khởi động lại dịch vụ
    ```
    systemctl restart openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service
    ```
- Tạo Volume, kiểm tra message trên queue notification.info

## Tại Neutron
- Thực hiện lần lượt từ CTR1 tới CTR3
- Tạo file backup
    ```
    cp /etc/neutron/neutron.conf  /etc/neutron/neutron.conf.bak
    ```
- Cấu hình:
    ```
    vi /etc/neutron/neutron.conf
    ```
    - Nội dung
        ```
        [oslo_messaging_notifications]
        driver = messagingv2

        [oslo_messaging_rabbit]
        rabbit_retry_interval = 1
        rabbit_retry_backoff = 2
        amqp_durable_queues = true
        rabbit_ha_queues = true
        ```
- Khởi động lại dịch vụ
    ```
    systemctl restart neutron-server.service neutron-linuxbridge-agent.service neutron-l3-agent.service
    ```

- Tạo Network private hoặc tạo 1 instance, Kiểm tra queue notification.info (sẽ thấy port thay đổi trạng thái khi tạo máy ảo)

## Nova
- Cần cấu hình bổ sung tại cả Controller và Compute

### Tại Controller
- Thực hiện lần lượt từ CTR1 tới CTR3
- Tạo file backup
    ```
    cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
    ```
- Cấu hình:
    ```
    vi /etc/nova/nova.conf
    ```
    - Nội dung
        ```
        [DEFAULT]
        ...
        notify_on_state_change = vm_and_task_state

        [oslo_messaging_notifications]
        driver = messagingv2

        [oslo_messaging_rabbit]
        rabbit_ha_queues = true
        rabbit_retry_interval = 1
        rabbit_retry_backoff = 2
        amqp_durable_queues= true
        ```
- Khởi động lại dịch vụ
    ```
    systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service  openstack-nova-conductor.service openstack-nova-novncproxy.service
    ```

### Tại Compute
- Thực hiện lần lượt trên các Compute
- Tạo file backup
    ```
    cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
    ```
- Cấu hình:
    ```
    vi /etc/nova/nova.conf
    ```
    - Nội dung
        ```
        [oslo_messaging_notifications]
        driver = messagingv2

        [oslo_messaging_rabbit]
        rabbit_ha_queues = true
        rabbit_retry_interval = 1
        rabbit_retry_backoff = 2
        amqp_durable_queues= true
        ```
- Khởi động lại dịch vụ
    ```
    systemctl restart openstack-nova-compute.service
    ```

### Kiểm tra
- Tạo Instance, kiểm tra queue notification xem đã đẩy đủ event

## Tại Keystone
- Kiểm tra Exchange `keystone` trước khi thực hiện, nếu tồn tại => Xóa Exchange `keystone`
- Tạo file backup
    ```
    cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
    ```
- Cấu hình:
    ```
    vi /etc/keystone/keystone.conf
    ```
    - Nội dung
    ```
    [oslo_messaging_notifications]
    driver = messagingv2

    [oslo_messaging_rabbit]
    rabbit_ha_queues = true
    rabbit_retry_interval = 1
    rabbit_retry_backoff = 2
    amqp_durable_queues= true
    ```
- Khởi động lại dịch vụ
    ```
    systemctl restart httpd
    ```
- Tạo project, kiểm tra queue notification xem đã đẩy đủ event

## Purge queue channel 
`images/ms_queue`
>## Không áp dụng Product

```sh 
# Reset app
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app
# Tạo lại user & gán quyền
rabbitmqctl add_user openstack passla123
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
systemctl enable rabbitmq-server.service
rabbitmqctl set_user_tags openstack administrator
```
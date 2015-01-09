# Hướng dẫn sử dụng shinken 2.0 monitor các service trong OpenStack (mô hình 3 node)

##1. Giới thiệu

Hiện tại tôi đang thử nghiệm giám sát hệ thống OpenStack bằng cách sử dụng kết hợp 3 phương pháp sau:
- Giám sát các máy chủ vật lý (Controller, Network, Compute,...) bằng công cụ [Zabbix](https://github.com/hocchudong/zabbix-2.2) (bài viết đang hoàn thiện)
- Giám sát các máy ảo (VM) trong hệ thống bằng [ZabbixCeilometer-Proxy](https://github.com/hocchudong/ZabbixCeilometer-Proxy)
- Giám sát các service của hệ thống OpenStack bằng Shinken 2.0

Trong bài viết này tôi sẽ hướng dẫn các bạn mục thứ 3: "Giám sát các service của OpenStack"

##2. Các thành phần cần giám sát

####2.1. Controller Node

- NTP (có thể monitor bằng Shinken với phương pháp linux-ssh)
- Mysql (có thể monitor bằng Zabbix)
- Rabbitmq (bổ sung sau)
- Keystone
- Glance
- Nova
- Newtron server 
- ML2
- Cinder
- Horizon

####2.2. Network Node

- ML2
- L2 agent
- L3 agent
- DHCP agent
- Metadata agent

####2.3. Các node compute

- Nova
- ML2
- OpenvSwitch

##3. Thực hiện 

Cài đặt shinken theo [hướng dẫn](https://github.com/hocchudong/shinken-2.0)

**Chú ý:** bạn cần đảm bảo rằng máy chủ shinken dùng để monitor của bạn có thể giám sát các máy chủ OpenStack bằng phương pháp nrpe.
Bạn có thể tham khảo bài viết [sau](https://github.com/hocchudong/shinken-2.0/blob/master/Huong%20dan/nrpe.md) để cài đặt và cấu hình nrpe.

####3.1. Node Controller

Thêm các command check nrpe:

`vi /usr/local/nagios/etc/nrpe.cfg`

    #Thêm vào các dòng sau
    #horizon
    command[check_horizon]=/usr/lib/nagios/plugins/check_http localhost -u /horizon -R username

    #keystone
    command[check_keystone_proc]=/usr/lib/nagios/plugins/check_procs -w 1 -u keystone
    command[check_keystone_http]=/usr/lib/nagios/plugins/check_http localhost -p 5000 -R application/vnd.openstack.identity-v3

    #neutron
    command[check_neutron_api_http]=/usr/lib/nagios/plugins/check_http localhost -p 9696
    command[check_neutron_server_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-server

    #nova
    command[check_nova_proc]=/usr/lib/nagios/plugins/check_procs -w 4: -u nova

    #Glance
    command[check_glance_http]=/usr/lib/nagios/plugins/check_http localhost -p 9292
    command[check_glance_proc]=/usr/lib/nagios/plugins/check_procs -w 4: -u glance

    #cinder
    command[check_cinder_proc]=/usr/lib/nagios/plugins/check_procs -w 4: -u cinder
    command[check_cinder_http]=/usr/lib/nagios/plugins/check_http localhost -p 8776

Restart nrpe:

`service xinetd restart`

####3.2. Node Network

`vi /usr/local/nagios/etc/nrpe.cfg`

    #neutron
    command[check_newtron_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -u neutron
    command[check_lbaas]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-lbaas-agent
    command[check_ovs_agent]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-openvswitch-agent
    command[check_neutron_rootwrap]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-rootwrap
    command[check_l3_agent]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-l3-agent
    command[check_dhcp_agent]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-dhcp-agent
    command[check_metadata_agent]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-metadata-agent
    command[check_metadata_proxy]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-ns-metadata-proxy


    #nova
    command[check_nova_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -u nova

    #openvswitch
    command[check_ovswitch_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -C ovs-vswitchd
    command[check_ovswitch_server_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -C ovsdb-server

`service xinetd restart`

####3.3. Node Compute1

`vi /usr/local/nagios/etc/nrpe.cfg`

    #ML2
    command[check_ovs_agent]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-openvswitch-agent

    #Neutron rootwrap
    command[check_neutron_rootwrap]=/usr/lib/nagios/plugins/check_procs -c 1: -a neutron-rootwrap

    #nova
    command[check_nova_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -u nova

    #openvswitch
    command[check_ovswitch_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -C ovs-vswitchd
    command[check_ovswitch_server_proc]=/usr/lib/nagios/plugins/check_procs -c 1: -C ovsdb-server

`service xinetd restart`

####3.4. Máy chủ giám sát Shinken

Tôi đã tạo sẵn các file cấu hình của shinken trong bài viết này, bạn chỉ cần tải chúng về vào đúng các thư mục của shinken và đổi IP là xong.

Các file cấu hình host:

```sh
cd /etc/shinken/hosts/
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/hosts/controller.cfg
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/hosts/network.cfg
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/hosts/compute1.cfg
```

Sau đó bạn chỉ cần sửa 3 file này, sửa dòng `address` thành IP của máy chủ OpenStack tương ứng.

Các file cấu hình service

```sh
cd /etc/shinken/hosts/
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/services/controller.cfg
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/services/network.cfg
wget https://raw.githubusercontent.com/hocchudong/monitor-openstack-services/master/shinken-config/services/compute1.cfg
```

Restart lại shinken:

`service shinken restart`

##Enjoy!
<img src=http://i.imgur.com/azi7OUE.png width="60%" height="60%" border="1">

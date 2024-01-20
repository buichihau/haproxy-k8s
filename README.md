# Triển khai dịch vụ High Available với Keepalived + HAproxy cho K8S

## 1. Install Keepalived
* Install all host

```
yum install -y keepalived
```

* Thêm vào file sysctl.conf
```
echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
```
* Chạy lệnh để hệ thống nhận file cấu hình và kiểm tra lại xem đã có dòng vừa thêm chưa:
```
sysctl -p
```
* Cài thêm gói psmisc để thêm tiện ích killall (Cài trên cả 2 node)
```
yum install -y psmisc
```
## 2. Config Keepalived
 
### conf node 1

* vim /etc/keepalived/keepalived.conf
```
global_defs {
  router_id node1                            #khai báo route_id của keepalived
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 100
  state MASTER
  interface ens192                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng
  virtual_ipaddress {
    192.168.2.148 dev ens192           #Khai báo Virtual IP cho interface tương ứng
  }
 authentication {
     auth_type PASS
     auth_pass Abc123qweqaz192                   #Password này phải khai báo giống nhau giữa các server keepalived
     }
  track_script {
    chk_haproxy
  }
}
vrrp_instance VI_2 {
  virtual_router_id 52
  advert_int 1
  priority 100
  state MASTER
  interface ens224                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng
  virtual_ipaddress {
    103.29.26.148 dev ens224           #Khai báo Virtual IP cho interface tương ứng
  }
 authentication {
     auth_type PASS
     auth_pass Abc123qweqaz224                  #Password này phải khai báo giống nhau giữa các server keepalived
     }
  track_script {
    chk_haproxy
  }
}
```

### conf node 2
* vim /etc/keepalived/keepalived.conf
```
global_defs {
  router_id node2                           #khai báo route_id của keepalived
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 99
  state BACKUP
  interface ens192                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng
  virtual_ipaddress {
    192.168.2.148 dev ens192           #Khai báo Virtual IP cho interface tương ứng
  }
 authentication {
     auth_type PASS
     auth_pass Abc123qweqaz192                   #Password này phải khai báo giống nhau giữa các server keepalived
     }
  track_script {
    chk_haproxy
  }
}
vrrp_instance VI_2 {
  virtual_router_id 52
  advert_int 1
  priority 99
  state BACKUP
  interface ens224                            #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng
  virtual_ipaddress {
    103.29.26.148 dev ens224           #Khai báo Virtual IP cho interface tương ứng
  }
 authentication {
     auth_type PASS
     auth_pass Abc123qweqaz224                  #Password này phải khai báo giống nhau giữa các server keepalived
     }
  track_script {
    chk_haproxy
  }
}
```

### Start keepalived
```
systemctl start keepalived
```

### Check trên node 1

```
ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:8f:bb:6c brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.146/23 brd 192.168.3.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet 192.168.2.148/32 scope global ens192
       valid_lft forever preferred_lft forever
3: ens224: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:8f:92:cd brd ff:ff:ff:ff:ff:ff
    inet 103.29.26.146/24 brd 103.29.26.255 scope global noprefixroute ens224
       valid_lft forever preferred_lft forever
    inet 103.29.26.148/32 scope global ens224
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:55:36:ae:c7 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```


### Giải thích:
* priority 100: set node1 làm MASTER
* priority 99: set node2 làm BACKUP
* script "killall -0 haproxy": Kiểm tra dịch vụ HAProxy còn hoạt động trên node hay không, nếu không VIP sẽ tự động nhảy sang node còn lại.
* Khai báo vrrp_script: 
```
vrrp_script chk_haproxy {
script "killall -0 haproxy"           #check pid của dịch vụ haproxy có tồn tại hay không
interval 2                                     #thời gian lặp lại đoạn script đơn vị là second
weight 2                                      #trọng số khấu trừ priority 2
}
track_script {
  chk_haproxy                             #khai báo tên đoạn script 
  }

```
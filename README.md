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
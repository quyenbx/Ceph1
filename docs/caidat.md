## Hướng dẫn cài đặt CEPH MiMic

## 1. Chuẩn bị môi trường Lab
Mô hình sử dụng 3 server cài đặt OS Centos 7.6.1810

- Host Ceph-mon: cài đặt ceph-deploy, ceph-mon, ceph-osd, ceph-mgr

- Host Ceph1: cài đặt ceph-osd

- Host Ceph2: cài đặt ceph-osd

Mô hình cài đặt như hình ảnh bên dưới

![](./images/ceph1.png)

IP Planning

![](./images/IP1.png)

- Cài đặt NTP
```sh
yum install chrony -y
```

- Enable NTP
```sh
systemctl start chronyd 
systemctl enable chronyd
```

- Kiểm tra chronyd hoạt động
```sh
chronyc sources -v
```

- Disabled NetworkManager
```sh
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network
```

- Cài đặt epel-relese và update OS
```sh
yum install epel-release -y
yum update -y
```

- Tắt selinux
```sh
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

- Disable firewalld
```sh
sudo systemctl disable firewalld
sudo systemctl stop firewalld
```

- Cấu hình file hosts
```sh
cat << EOF >> /etc/hosts
10.10.14.181 ceph-mon
10.10.14.182 ceph1
10.10.14.183 ceph2
EOF
```

## 2. Cài đặt Ceph
Thực hiện trên Node Ceph-mon
- Cài đặt ceph-deploy
```sh
rpm -ivh https://download.ceph.com/rpm-mimic/el7/noarch/ceph-deploy-2.0.1-0.noarch.rpm
```

- Cài đặt python-setuptools để ceph-deploy có thể hoạt động ổn định
```sh
curl https://bootstrap.pypa.io/ez_setup.py | python
```

- Kiểm tra cài đặt
```sh
ceph-deploy --version
```

- Tạo SSH Key và copy sang các Node Ceph khác

- Tạo các thư mục ceph-deploy để thao tác cài đặt vận hành Cluster
```sh
mkdir /ceph-deploy && cd /ceph-deploy
```

- Khởi tại file cấu hình cho cụm với node quản lý là Ceph-mon
```sh
ceph-deploy new ceph-mon
```

- Kiểm tra lại thông tin folder ceph-deploy
```sh
[root@ceph-mon ceph-deploy]# ls -alh
total 20K
drwxr-xr-x   2 root root 4.0K Feb 14 14:28 .
dr-xr-xr-x. 19 root root 4.0K Feb 14 14:28 ..
-rw-r--r--   1 root root  198 Feb 14 14:28 ceph.conf
-rw-r--r--   1 root root 3.0K Feb 14 14:28 ceph-deploy-ceph.log
-rw-------   1 root root   73 Feb 14 14:28 ceph.mon.keyring
```

- Chúng ta sẽ bổ sung thêm vào file ceph.conf một vài thông tin cơ bản như sau:
```sh
cat << EOF >> /ceph-deploy/ceph.conf
osd pool default size = 2
osd pool default min size = 1
osd crush chooseleaf type = 0
osd pool default pg num = 128
osd pool default pgp num = 128

public network = 10.10.14.0/24
cluster network = 10.10.15.0/24
EOF
```

- Cài đặt ceph trên toàn bộ các node ceph
```sh
ceph-deploy install --release mimic ceph-mon ceph1 ceph2
```

- Kiểm tra sau khi cài đặt
```sh
ceph -v
```

- Khởi tạo cluster với các node mon (Monitor-quản lý) dựa trên file ceph.conf
```sh
ceph-deploy mon create-initial
```

- Sau khi thực hiện lệnh phía trên sẽ sinh thêm ra 05 file : ceph.bootstrap-mds.keyring, ceph.bootstrap-mgr.keyring, ceph.bootstrap-osd.keyring, ceph.client.admin.keyring và ceph.bootstrap-rgw.keyring

- Để node ceph-mon có thể thao tác với cluster chúng ta cần gán cho node ceph-mon với quyền admin bằng cách bổ sung cho node này admin.keying
```sh
ceph-deploy admin ceph-mon
```

- Kiểm tra bằng lệnh
```sh
ceph -s
```

- Khởi tạo MGR: Ceph-mgr là thành phần cài đặt cần khởi tạo từ bản Luminous, có thể cài đặt trên nhiều node hoạt động theo cơ chế Active-Passive
  - Cài đặt ceph-mgr trên ceph-mon
  ```sh
  ceph-deploy mgr create ceph-mon
  ```
  
  - Ceph-mgr hỗ trợ dashboard để quan sát trạng thái của cluster, Enable mgr dashboard trên host ceph-mon
  ```sh
  ceph mgr module enable dashboard
  ceph dashboard create-self-signed-cert
  ceph dashboard set-login-credentials <username> <password>
  ceph mgr services
  ```
  
- Khởi tạo OSD
Tạo OSD thông qua ceph-deploy tại host ceph-mon
  - Dùng ceph-deploy để format ổ đĩa dùng làm OSD
  ```sh
  ceph-deploy disk zap ceph-mon /dev/sdb
  ```
  
  - Tạo OSD với ceph-deploy
  ```sh
  ceph-deploy osd create --data /dev/sdb ceph-mon
  ```
  
  - Kiểm tra osd vừa tạo bằng lệnh
  ```sh
  ceph osd tree
  ```
  
  - Kiểm tra ID của OSD bằng lệnh
  ```sh
  lsblk
  ```
  ----------------------------------------------------
  
  







 
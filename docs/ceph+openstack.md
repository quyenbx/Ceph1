## Tích hợp Ceph với Openstack

Mô hình kết nối

Đối với OpenStack có 3 thành phần có thể kết nối được với Ceph.

- Images: OpenStack Glance quản lý các image cho các VM, lưu trữ các images dưới dạng nhị phân.
- Volumes: Các Block Devices được sử dụng để boot các VM hoặc là các ổ đĩa đính kèm thêm cho VM
- Guest Disk: Mặc định khi khởi động VM thì disk của nó được xuất hiện lưu trữ dưới dạng filesystems (Thường là /var/lib/nova/instances/uuid/)

Cài đặt lib ceph python cho các node Compute và Controller
```sh
yum install -y python-rbd ceph-common
```

Tạo pool trên Ceph
- Lưu ý: Có thể tính toán trước số PG khi tạo các pool bằng cách sử dụng công cụ tính toán có sẵn trên trang chủ http://ceph.com/pgcalc
```sh
ceph osd pool create volumes 128 128
ceph osd pool create vms 128 128
ceph osd pool create images 128 128
ceph osd pool create backups 128 128
```

- Khởi tạo ban đầu trước khi sử dụng pool
```sh
rbd pool init volumes
rbd pool init vms
rbd pool init images
rbd pool init backups
```

- Thực hiện copy cấu hình qua các node Controller, Compute
```sh
ssh 10.10.10.206 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 10.10.10.207 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 10.10.10.208 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
```

## 1. Cấu hình CEPH làm backend cho glance images
Tạo user glance trên node ceph
```sh
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
```

Chuyển key glance sang node glance
```sh
ceph auth get-or-create client.glance | ssh 10.10.10.206 sudo tee /etc/ceph/ceph.client.glance.keyring
ssh 10.10.10.206 sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring

Thêm cấu hình /etc/glance/glance-api.conf trên node ctl
```sh
[DEFAULT]
show_image_direct_url = True

[glance_store]
show_image_direct_url = True
default_store = rbd
stores = file,http,rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

Restart lại dịch vụ glance trên node ctl
```sh
systemctl restart openstack-glance-*
```

Tạo thử images
```sh
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img 
openstack image create "cirros-ceph" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

Kiểm tra trên node ceph
```sh
rbd -p images ls
```

## 2. Cấu hình CEPH làm backend cho cinder và cinder backup
Thao tác trên Node Ceph
Di chuyển vào ceph-deploy folder
```sh
cd ceph-deploy
```

Tạo user cinder, cinder-backup
```sh
ceph auth get-or-create client.cinder mon 'allow r, allow command "osd blacklist", allow command "blacklistop"' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images' > ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' > ceph.client.cinder-backup.keyring
```

Chuyển key cinder và key cinder-backup sang các node cài đặt Cinder
```sh
ceph auth get-or-create client.cinder | ssh 10.10.10.206 sudo tee /etc/ceph/ceph.client.cinder.keyring
ssh 10.10.10.206 sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup | ssh 10.10.10.206 sudo tee /etc/ceph/ceph.client.cinder-backup.keyring
ssh 10.10.10.206 sudo chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring
```

Chuyển key cinder sang các node Compute
```sh
ceph auth get-or-create client.cinder | ssh 10.10.10.207 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-key client.cinder | ssh 10.10.10.207 tee /root/client.cinder
ceph auth get-or-create client.cinder | ssh 10.10.10.208 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-key client.cinder | ssh 10.10.10.208 tee /root/client.cinder
```

Thao tác trên Node Compute
- Khởi tạo 1 uuid mới cho Cinder
```sh
uuidgen
```

- Kết quả của lệnh trên
```sh
b2dbf29e-74ab-4032-9aa9-b068ff29d477
```

- Tạo file xml cho phép Ceph RBD (Rados Block Device) xác thực với libvirt thông qua uuid vừa tạo
```sh
cat > ceph-secret.xml <<EOF
<secret ephemeral='no' private='no'>
<uuid>b2dbf29e-74ab-4032-9aa9-b068ff29d477</uuid>
<usage type='ceph'>
	<name>client.cinder secret</name>
</usage>
</secret>
EOF

sudo virsh secret-define --file ceph-secret.xml
```

- Gán giá trị của client.cinder cho uuid
```sh
virsh secret-set-value --secret b2dbf29e-74ab-4032-9aa9-b068ff29d477 --base64 $(cat /root/client.cinder)
```

Bổ sung cấu hinh /etc/cinder/cinder.conf tren cac node controller
```sh
[DEFAULT]
notification_driver = messagingv2
enabled_backends = ceph
glance_api_version = 2
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = cinder-backup
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = b2dbf29e-74ab-4032-9aa9-b068ff29d477
report_discard_supported = true
```

Enable cinder-backup và restart dịch vụ cinder
```sh
systemctl enable openstack-cinder-backup.service
systemctl start openstack-cinder-backup.service
```

Restart lại dịch vụ trên Node Controller
```sh
systemctl restart openstack-cinder-api.service openstack-cinder-volume.service openstack-cinder-scheduler.service openstack-cinder-backup.service
```

Tạo volume type node controller
```sh
cinder type-create ceph
cinder type-key ceph set volume_backend_name=ceph
```

Restart lai dich vu nova-compute tren node compute
```sh
systemctl restart openstack-nova-compute
```

## 3. Cấu hình CEPH làm backend cho nova
Mặc định các VM được tạo từ Images sẽ lưu file disk ngay chính trên Compute, Việc tích hợp này cho phép file disk này được tạo 1 symlink lưu trữ dưới Ceph thay vì lưu local.

Thao tác trên Node Ceph

- Tạo keyring cho Cinder
```sh
ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' -o /etc/ceph/ceph.client.nova.keyring 
```

- Copy keyring tới node COM
```sh
scp /etc/ceph/ceph.client.nova.keyring root@10.10.10.207:/etc/ceph/
scp /etc/ceph/ceph.client.nova.keyring root@10.10.10.208:/etc/ceph/
```

- Chuyển key cinder sang các node Compute
```sh
ceph auth get-or-create client.cinder | ssh 10.10.10.207 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-key client.cinder | ssh 10.10.10.207 tee /root/client.cinder
ceph auth get-or-create client.cinder | ssh 10.10.10.208 sudo tee /etc/ceph/ceph.client.cinder.keyring
ceph auth get-key client.cinder | ssh 10.10.10.208 tee /root/client.cinder
```

- Thao tác trên Node Compute
Set quyền trên node COM
```sh
chgrp nova /etc/ceph/ceph.client.nova.keyring
chmod 0640 /etc/ceph/ceph.client.nova.keyring
```

- Genkey UUID
```sh
uuidgen
```

Kết quả của lệnh trên
```sh
3a485975-13eb-42cf-a6bf-0125050223de
```

- Tạo file xml cho phép Ceph RBD (Rados Block Device) xác thực với libvirt thông qua uuid vừa tạo
```sh
cat << EOF > nova-ceph.xml
<secret ephemeral="no" private="no">
<uuid>3a485975-13eb-42cf-a6bf-0125050223de</uuid>
<usage type="ceph">
<name>client.nova secret</name>
</usage>
</secret>
EOF

sudo virsh secret-define --file nova-ceph.xml
```

- Gán giá trị của client.nova cho uuid

```sh
virsh secret-set-value --secret b2dbf29e-74ab-4032-9aa9-b068ff29d477 --base64 $(cat /root/client.nova)
```

- Chỉnh sửa nova.conf trên COM /etc/nova/nova.conf
```sh
[libvirt]
images_rbd_pool=vms
images_type=rbd
rbd_secret_uuid=3a485975-13eb-42cf-a6bf-0125050223de
rbd_user=nova
images_rbd_ceph_conf = /etc/ceph/ceph.conf
```

- Restart service
```sh
systemctl restart openstack-nova-compute
```






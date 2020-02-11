# DM-Cache

DM-Cache là một công nghệ cache disk được cung cấp bởi RHEL. Nó có thể tạo một phân vùng trên ổ SSD cục bộ để sử dụng làm bộ đệm cho OSD



- enable giao thức TRIM , nhằm xóa các dữ liệu cũ không còn sử dụng và thống kê các block free để ghi dữ liệu vào nhanh chóng trong logical volumes

```
vi /etc/lvm/lvm.conf
...
issue_discards = 1
```

kiểm tra bằng lệnh `lsblk -D` nếu option DISC-GRAN và DISC-MAX là 0 có nghĩa là đang ko được bật

- Tái tạo lại ram cho lần load đầu tiên

`dracut -f`

- Đồng bộ và reboot hệ thống

```
sync
reboot 
```

--------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------

# Kịch bản

Sử dụng ổ vdb1 và vdb2 làm OSD, ổ vdc1 và vdc2 làm journal cho 2 osd, ổ vdd1 và vdd2 làm cache


<img src="https://i.imgur.com/1NahcaZ.png">


**Trước khi muốn tạo cache pool cho ổ OSD ta cần ghép ổ đó vào OSD bằng vgextend**

- Kiểm tra volume group và volume group của 2 OSD đã tạo

`lvs`

<img src="https://i.imgur.com/62jD1fy.png">

- Gán ổ làm cache vào volume group của OSD 

  Gán OSD thứ nhất cho ổ vdd1:

  `vgextend ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5 /dev/vdd1`

  Gán OSD thứ hai cho ổ vdd2:

  `vgextend ceph-91195b48-a909-4c2e-a15e-7fadf33859a6 /dev/vdd2`


Kiểm tra ổ đã gán bằng lệnh : `pvs -o+pv_used`

Nếu muốn gỡ ổ ra dùng lệnh `vgreduce <tên VG đang được gán> /dev/<ổ đang gán>` nếu mục `Used` không có dung lượng, còn nếu có dung lượng dùng lệnh `pvmove /dev/<ổ đang gán>` sau đó mới dùng lệnh vgreduce

- Do 2 ổ vdd1 và vdd2 đã add vào 2 OSD và 2 OSD đã có  physical volume và volume group, nên không cần tạo cho disk vdd1 và vdd2

- Tạo cachedisk và metacache cho 2 ổ vdd1 và vdd2. (*lưu ý việc tạo cache sẽ cần dư 1 khoảng dung lượng cho PE, pool càng to thì PE cần càng nhiều*

  vdd1:
  
`lvcreate -L 15G -n cachedisk1 ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5 /dev/vdd1`  (*pool cache data*)

`lvcreate -L 15G -n metadisk1 ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5 /dev/vdd1`  (*pool cache metadata*)

  vdd2:
  
`lvcreate -L 15G -n cachedisk2 ceph-91195b48-a909-4c2e-a15e-7fadf33859a6 /dev/vdd2`  (*pool cache data*)

`lvcreate -L 15G -n metadisk2 ceph-91195b48-a909-4c2e-a15e-7fadf33859a6 /dev/vdd2`  (*pool cache metadata*)
  
- Add pool cachedisk , metacache vào sử dụng cho cache-pool

  Tạo Cache-pool cho OSD thứ nhất:
  
  `lvconvert --type cache-pool /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/cachedisk1 --poolmetadata /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/metadisk1`

  Tạo Cache-pool cho OSD thứ hai:
  
  `lvconvert --type cache-pool /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/cachedisk2 --poolmetadata /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/metadisk2`



- Apply cache-pool cache cho OSD

  Cache OSD thứ nhất:

  `lvconvert --type cache --cachepool cachedisk1 /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/osd-data-d5c22d57-f0ab-41c8-9597-d5f7d0730afd`

  <img src="https://i.imgur.com/1ZO8X44.png">

  Cache OSD thứ hai:
  
  `lvconvert --type cache --cachepool cachedisk2 /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/osd-data-f4dc90f7-16d6-4060-8e22-ac1bc0b3ab28`
  
  <img src="https://i.imgur.com/xpOh0Co.png">
  
  

- Kiểm tra

<img src="https://i.imgur.com/o31k5Lx.png">

`lvs -a`

```
dmsetup status /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/osd-data-d5c22d57-f0ab-41c8-9597-d5f7d0730afd

dmsetup status /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/osd-data-f4dc90f7-16d6-4060-8e22-ac1bc0b3ab28

```

```
lvs -o name,cache_read_hits,cache_read_misses /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/osd-data-d5c22d57-f0ab-41c8-9597-d5f7d0730afd

lvs -o name,cache_read_hits,cache_read_misses /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/osd-data-f4dc90f7-16d6-4060-8e22-ac1bc0b3ab28
```


```
lvs /dev/ceph-e056d48f-4a2e-4b45-92f8-88e5f54303d5/osd-data-d5c22d57-f0ab-41c8-9597-d5f7d0730afd
lvs /dev/ceph-91195b48-a909-4c2e-a15e-7fadf33859a6/osd-data-f4dc90f7-16d6-4060-8e22-ac1bc0b3ab28
```


`dmsetup table`

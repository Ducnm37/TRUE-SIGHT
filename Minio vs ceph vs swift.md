# So sánh giải pháp object storage giữa minio và swift và ceph


Đặc điểm                |Minio                              |Swift                                    |Ceph
------------------------|-----------------------------------|-----------------------------------------|----------------------------------------
Giao tiếp API           |S3 amazon, SDK                     |Swift                                    |S3 amazon, swift
Cơ chế bảo vệ dữ liệu   |erase-coding có thể khôi phuck dữ liệu khi 1 nửa ổ cứng bị lỗi còn raid 6 chỉ khôi phục dc 2 điểm lỗi, đồng thời tiết kiệm dung lượng hơn replication            |không có                     |replication
Mở rộng hạ tầng         |

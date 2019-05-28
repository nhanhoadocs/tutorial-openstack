# QoS cho Disk 

## Phương án 1: Sử dụng trực tiếp trên KVM 

Thực hiện thiết lập QoS với thông số Read là 2500 và Write 2500:
```sh
virsh blkdeviotune instance-00000875 vda --read-iops-sec 2500 --write-iops-sec 2500 --live
```

Thực hiện limit bandwidth read hoặc write  byte /s (Trong ví dụ là 200MB/s)
```sh
virsh blkdeviotune instance-00000875 vda --write_bytes_sec $(expr 1024 \* 1024 \* 200)
virsh blkdeviotune instance-00000875 vda --read_bytes_sec $(expr 1024 \* 1024 \* 200)
```

Thực hiện limit bandwidth cả read, write  byte /s (Trong ví dụ là 200MB/s)
```
virsh blkdeviotune instance-00000875 vda --write_bytes_sec $(expr 1024 \* 1024 \* 200) --read_bytes_sec $(expr 1024 \* 1024 \* 200)
Kiểm tra lại:
virsh blkdeviotune instance-00000992 vda
```
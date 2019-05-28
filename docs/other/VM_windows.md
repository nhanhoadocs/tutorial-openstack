# Các lưu ý với VM windows

## Desktop 

## Server 

- User `Administrator` thường disable sau khi boot VM lên

Cloud-init Win2k12
```sh 
#ps1
net user administrator /active:yes
net user administrator <PASSWORD>
net user admin /active:no
net user admin /delete
net user cloud /active:no
net user cloud /delete
shutdown /r
```

- Khái niệm Socket-Core-Threat
    - Win 2k8 chỉ nhận max 8 Socket
    - Win7 chỉ nhận max 2 Socket

Tạo Flavor > số core thì để mặc định metadata 
```sh 
hw:cpu_max_sockets:8
```
> Apply cho Win2k8 

```sh 
hw:cpu_max_sockets:2
```
> Apply cho Win7

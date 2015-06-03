# Một số note khi sử dụng packer để tạo image

Packer là một kỹ thuật tạo image có thể sử dụng cho nhiều nền tảng khác nhau (Amaxone EC2, Digtal Ocean, Docker, Openstack, QEMU,...). Chi tiết tham khảo tại https://packer.io/docs/
Trong bài viết này sẽ sử dụng Packer để đóng một image Ubuntu 14.04 sử dụng cho Cloud OpenStack

###1. Cài đặt
Packer có thể cài đặt trên bất cứ hệ điều hành nào (https://packer.io/docs/installation.html), ở đây tôi sử dụng luôn Node Controller của OpenStack (Ubuntu 14.04) để đóng image. Bạn có thể sử dụng chính Laptop của bạn để đóng image (Windows)

Các bước thực hiện để cài Packer tại node Controller:
```sh
cd /usr/local/
mkdir packer
cd packer/
wget https://dl.bintray.com/mitchellh/packer/packer_0.7.5_linux_amd64.zip
apt-get install unzip
unzip packer_0.7.5_linux_amd64.zip
export PATH=$PATH:/usr/local/packer
echo "PATH=$PATH:/usr/local/packer" >> ~/.bashrc
cd ~
```

Trên terminal thực hiện lệnh `packer version` để kiểm tra xem đã cài đặt thành công hay chưa

###2. Thực hiện tạo image

####Đầu tiên bạn phải đảm bảo rằng hệ thống OpenStack của bạn đã có một Image Ubuntu 14.04 sử dụng Cloud-init

-  Về cách đóng image cho OpenStack có thể tham khảo tại: https://github.com/vdcit/Tao-image
-  Về Cloud-init có thể tham khảo tại: https://github.com/vdcit/Cloud-Init

Hoặc cách đơn giản nhất bạn có thể sử dụng image tại: https://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img và tạo một image như sau:
![Alt text](http://i.imgur.com/evCnBmO.png?1 )

####Tạo một file với tên là imageubuntu.json với nội dung như sau:

vi imageubuntu.json
```sh
{
  "builders": [
    {
      "type": "openstack",
      "username": "admin",
      "password": "Welcome123",
      "provider": "http://controller:35357/v2.0",
      "ssh_username": "ubuntu",
      "project": "admin",
      "region": "RegionOne",
      "image_name": "Packer image apache_mysql",
      "source_image": "d17be9e3-3f7a-4909-a297-d6fee91d87da",
      "flavor": "2",
      "networks": ["79832073-d2ad-45cd-bfc6-7bbcb5cc554d"],
      "use_floating_ip": true,
      "floating_ip": "172.16.69.38",
      "ssh_timeout": "5m"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "script": "provision-apache.sh"
    }
  ]
}
```

Trong đó `builders` là thành phần của packer có nhiệm vụ đọc các cấu hình bên trong nó và tạo image cho nền tảng mong muốn theo đúng như những dòng cấu hình đó. Thành phần của `builders` gồm:
- type: nền tảng mong muốn (OpenStack, VMware, QEMU,...)
- username, password, provider, project, region là những thông tin xác thực keystone trong hệ thống OpenStack của bạn. Có thể xem thông tin này tại file `admin-openrc`
- ssh_username: tên user dùng để đăng nhập SSH, với cloud-init tên này mặc định là ubuntu
- image_name: tên của image mà bạn sắp tạo ra
- source_image: ID của image cơ sở (trong trường hợp này là image Ubuntu14cloud), có thể lấy ID này bằng lệnh nova image-list. 
![Ảnh minh họa](http://i.imgur.com/O5hsWmV.png)
- flavor: ID của flavor sử dụng để tạo image, ở đây sử dụng flavor `m1.small`, để liệt kê danh sách sử dụng `nova flavor-list`. 
![Ảnh minh họa](http://i.imgur.com/Kmq02eT.png)
- networks: ID của dải mạng private sử dụng cho máy ảo (thường là `br-int`), để liệt kê sử dụng lệnh `neutron net-list`. Trong trường hợp máy ảo muốn sử dụng nhiều dải mạng thì khai báo một list ở tham số này. 
![Ảnh minh họa](http://i.imgur.com/5gBk2IY.png)
- use_floating_ip: cho phép floating IP cho máy ảo trong quá trình tạo image
- floating_ip: xác định IP sẽ cấp cho máy ảo 
- ssh_timeout: thời gian timeout khi ssh vào máy ảo trong quá trình tạo image

Thành phần `provisioners`có nhiệm vụ cài đặt và cấu hình phần mềm bên trong một image
- type: phương pháp sử dụng (Shell script, Chef, Puppet,...). Trong ví dụ này sử dụng Shell
- script: đường dẫn đến script được sử dụng để cài đặt, cấu hình

####Tạo script nói ở bước trên. Script này có nhiệm vụ cài đặt `apache` và `mysql-server` cho image

vi provision-apache.sh
```sh
#Install Mysql and Apache
sudo sh -c "echo mysql-server mysql-server/root_password password "Admin123" | debconf-set-selections"
sudo sh -c "echo mysql-server mysql-server/root_password_again password "Admin123" | debconf-set-selections"
sudo apt-get update
sudo apt-get install -y apache2
sudo apt-get install mysql-server  -y
sleep 10
##Write index.html
sudo sh -c "echo \"<html><h3>Hello, PTCC@VDC</h3></html>\" > /var/www/html/index.html"
```

####Cuối cùng sử dụng lệnh sau để thực hiện tạo image
`packer build imageubuntu.json`



####Quá trình thực hiện:
- Packer launch một máy ảo tạm thời tên là `Packer image apache_mysql` từ image ban đầu `Ubuntu14cloud`
- Floating IP 172.16.69.38 cho máy ảo đó
- SSH bằng tài khoản ubuntu vào máy ảo
- Chạy script `provision-apache.sh` để cài đặt các gói cần thiết
- Sau khi hoàn tất các bước packer sẽ tạo một image mới dưới dạng snapshot từ máy ảo `Packer image apache_mysql` 
- Xóa máy ảo tạm `Packer image apache_mysql` và thông báo kết quả
- Trong quá trình trên nếu có bất cứ một lỗi gì thì quá trình sẽ không hoàn tất và image không thể tạo thành công

####Kết quả thu được:
```sh
root@controller:~# packer build imageubuntu.json
openstack output will be in this color.

==> openstack: Creating temporary keypair for this instance...
==> openstack: Waiting for server (d44a13f1-e50c-4ca5-9efa-c81c514211f5) to become ready...
==> openstack: Added floating IP 172.16.69.38 to instance...
==> openstack: Waiting for SSH to become available...
==> openstack: Connected to SSH!
==> openstack: Provisioning with shell script: provision-apache.sh
...............
...............
==> openstack: Creating the image: Packer image apache_mysql
==> openstack: Image: d9a93ab2-72ef-4972-8b31-ff12cca3519c
==> openstack: Waiting for image to become ready...
==> openstack: Terminating the source server...
==> openstack: Deleting temporary keypair...
Build 'openstack' finished.

==> Builds finished. The artifacts of successful builds are:
--> openstack: An image was created: d9a93ab2-72ef-4972-8b31-ff12cca3519c
```

Image mới được tạo trên OpenStack
![Ảnh minh họa](http://i.imgur.com/jWu6SGN.png)

###3. Một số lỗi và debug

- Không floating được IP: bạn phải chắc chắn rằng IP mà bạn sử dụng để float tồn tại và chưa được sử dụng cho host nào. http://i.imgur.com/xBdjQN0.png 
 
- SSH time out: Đôi khi bạn cấu thình tham số `ssh_timeout` quá ngắn khiến cho packer chưa kịp SSH vào máy ảo tạm thời thể thực hiện bước chạy script `provision-apache.sh` => xuất hiện lỗi `SSH time out` => không tạo được image

- Lỗi Script: khi packer ssh vào máy ảo tạm thời thành công, nó sẽ sử dụng user ubuntu để thực hiện các script. Do đó cần phải test kỹ càng script trước đó.
 
####Sử dụng chế độ debug:

Packer cung cấp một chế độ mà trong đó cho phép chúng ta theo dõi quá trình tạo image qua từng bước. Để sử dụng chức năng này thực hiện lệnh sau `packer build --debug imageubuntu.json`


### 4. Tham khảo

http://www.solinea.com/blog/image-creation-packer-and-openstack

https://packer.io/docs/



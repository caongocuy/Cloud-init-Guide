# Cloud-init-Guide
========================

#### 1. Giới Thiệu
  Người dùng thường muốn thực hiện một số cấu hình cho VM của họ sau khi nó khởi động. Ví dụ như, cài đặt một số gói, 
khởi động dịch vụ hoặc quản lí các VM. Khi tạo instances trong 1 hệ thống OpenStack cloud, có hai kỹ thuật làm việc với nhau
để hỗ trợ việc cấu hình tự động cho các instances tại thời điểm khởi động: user-data và cloud-init.

#### 2. User data
- Là cơ chế mà theo đó người dùng có thể chuyển thông tin chứa trong một tập tin đặt trên local vào trong một instance trong thời gian boot.
Trường hợp điển hình chuyển thông tin bằng cách dùng 1 file shell scrip hoặc một file cấu hình user data.

- user data được gửi bằng cách sử dụng tùy chọn *--user-data /path/to/filename* khi gọi lệnh *nova-boot*. Ví dụ như
```
$ echo "This is some text" > myfile.txt
$ nova boot --user-data ./myfile.txt --image myimage myinstance
```
- Các instances có thể lấy user-data bằng cách truy vấn meta-data service thông qua OpenStack meta-data API hoặc các API tương thích EC2.

- Trong ví dụ trên sử dụng file text, user-data có thể được định dạng nhiều định dạng khác.

#### 3. Cloud-init
- Khi sử dụng user-data thì VM image phải được cấu hình chạy một dịch vụ khi boot để lấy use data từ server meta-data và thực hiện một số hoạt động dựa trên nội dung của dữ liệu đưa vào.
Gói phần mềm Cloud-init được thiết kế thực thi điều này. đặc biệt, cloud-init tương thích với dịch vụ Compute meta-data giống như là các drive dùng để cấu hình máy tính.

- Lưu ý rằng cloud-init không phải là một công nghệ của OpenStack, mà nó là một gói phần mềm được thiết kế để hỗ trợ nhiều cloud providers, để các VM image có thể được sử 
dụng trong các cloud khác nhau mà không cần sửa đổi. Cloud-init là một dự án mã nguồn mở và source code có sẵn trên Launchpad.( http://launchpad.net/cloud-init )

- Việc cài đặt cloud-init trên các images mà bạn tạo ra là rất cần thiết để đơn giản hóa các thao tác cấu hình khi instances được boot lên. Thậm chí nếu bạn không muốn sủ dụng user-data để cấu hình hoạt động của instances trong khi boot, cloud-init cung cấp
chức năng hữu ích như việc sao chép public key cho một tài khoản.

- Nếu bạn không cài đặt cloud-init, ban sẽ cần phải tự cấu hình image của bạn để lấy public key từ service meta-data khi khởi động và sau đó copy nó vào tài khoản thích hợp.

###### 3.1. Cloud-init hỗ trợ các loại định dạng
cloud-init hỗ trợ các định dạng đầu vào khác nhau cho user-data. Tôi đưa ra 2 định dạng phổ biến nhất
- shell scrip ( bắt đầu với #! )
  Giả sử như bạn đã cài đặt cloud-init, cách đơn giản nhất để cấu hình một instance khi boot là truyền một shell scrip làm user-data. file shell bắt đầu bằng #! để cho cloud-init nhận ra
đó là một shell scrip. 
Ví dụ một scrip tạo user tên là clouduser.
```
#!/bin/bash
adduser --disabled-password --gecos "" clouduser
```
  Gửi một shell scrip làm user-data có một hiệu ứng tương tự như viết một kịch bản /etc/rc.local: nó được thực thi sau cùng trong quá trình boot.
- Cloud config files ( bắt đầu với #cloud-config )
  Cloud-init hỗ trợ định dạng cấu hình YAML cho phép user cấu hình số lượng lớn các tùy chọn. user-data bắt đầu với #cloud-config.
Ví dụ cấu hình hostname.
```
#cloud-config
hostname: mynode
fqdn: mynode.example.com
manage_etc_hosts: true
```
###### 3.2. Một số tính năng của Cloud-init
- set root password
- set timezone
- add ssh authorized keys for root user
- set static networking
- set hostname
- create users and/or set user passwords
- run custom scripts and/or commands
....

###### 3.3. Cloud-init hoạt động như thế nào
- cloud-config rất dễ sử dụng trong việc cấu hình các dịch vụ và ban hành lệnh thông qua các tập tin trong user-data. ví dụ ta tạo một file user-data với nội dung sau đây:
```
#cloud-config
# Your cloud config file must always start with this first line, indicating that you want CloudInit to interpret this user-data as a cloud config configuration
# We can run some bash script commands for example
runcmd:
- [ echo, "Hello World. I am being run by CloudInit!" ]
 ```
- lưu lại và tôi gọi nó là myfile. Bây giờ nếu bạn boot một instance, chẳng hạn như:
```
nova boot --image IMAGE-ID --flavor m1.small --key_name YOUR-KEY --meta cern-services=false --user_data myfile INSTANCE-NAME
```
- Khi instance ACTIVE bạn có thể ssh vào nó và nếu bạn mở tập tin /var/log/boot.log bạn sẽ thấy
```
Starting cloud-init-cfg: Hello World. I am being run by CloudInit!
```
- Trong khi boot cloud-init sẽ lấy thông tin từ user-data và xem bên trong có gì và thực thi. Dòng đầu tiên của tập tin này cho biết định dạng đầu vào. Cú pháp này
được xây dựng thông qua các mô-đun ( viết bằng python ), chúng có thể được tìm thấy trong /usr/lib/python-'version'/dist-packages/cloudinit/CloudConfig/.
Cloud-init biết mô-đun nào có thể được xử lí thông qua file /etc/cloud/cloud.cfg - list tất cả các module python. 
Bạn chỉ cần gọi đúng tên module trong file user-data và nắm được tác dụng từng module, như vậy bạn đã sử dụng được cloud-init

###### 3.4. Tạo module CLoudconfig của riêng bạn
- Như đã đề cập ở trên, tất cả cloud-config module được viết bằng python, để có những cái nhìn đầu tiên bạn nên đọc một vài module đã có sẵn
Sau khi bạn hoàn thành module của riêng bạn, bạn chỉ cần đặt file python trong /usr/lib/python-'version'/dist-packages/cloudinit/CloudConfig/ ( lưu ý đường dẫn này thay đổi theo phiên bản của cloud-init )
. Sau khi hoàn tất điều này, ban cần sửa file /etc/cloud/cloud.cfg và add thêm tên module bạn viết vào danh sách. Ví dụ như tôi muốn thêm
một module mới gọi là mymodule. Chúng ta có thể viết các tập tin cc_mymodule.py trực tiếp trong thư mục module ( trong instance chẳng hạn ) hoặc viết nó ở một nơi khác và sao chép nó vào trong thư mục module trong instance.
Sau đó, chúng ta cần phải chỉnh sửa file "cloud.cfg" và thêm module của chúng ta vào. Và thêm dưới phần "cloud_config_modules" như sau:
```
cloud_config_modules:
 - mounts
 - ssh-import-id
 - locale
 - set-passwords
 - timezone
 - puppet
 - disable-ec2-metadata
 - runcmd
 - mymodule
```
###### 3.5. Các bước để sử dụng Cloud-init trong Image
- Tạo một vm mới
- Cài đặt OS
- cài đặt cloud-init
- Cấu hình lại cloud-init ( nếu cần thiết )
- Tạo template (image)
- Boot Vm mới từ template ( thử nghiệm các module có sẵn hoặc do bạn viết ra )
- Chờ cho cloud-init cấu hình xong
- Vm cấu hình xong và sẵn sàng thao tác
###### 3.6. Một vài ví dụ  làm việc với Cloud-init
MỘt vài ví dụ làm việc với cloud-init

- thay đổi mk của user mặc định ( đối với ubuntu là ubuntu )
```
#cloud-config
password: 123456
ssh_pwauth: True
chpasswd: {expire: True}
```
- thay đổi user mặc định 
```
#cloud-config
user: uycn
password: 123456
chpasswd: {expire: False}
ssh_pwauth: True
```
- đặt password cho root
```
#cloud-config
ssh_pwauth: True
chpasswd:
  list: |
     root:1111
     ubuntu:uycn
  expire: False
```
- add thêm user 
```
#cloud-config
users: 
  - default
  - name: foobar
    gecos: User N. Ame
    selinux-user: staff_u
    groups: users,wheel
    ssh_pwauth: True

chpasswd:
  list: |
    root:password
    cloud-user:atomic
    foobar:foobar
  expire: False
```  
- Chạy một số lệnh khi boot
```
#cloud-config
chpasswd:
  list: |
    root:5555
    ubuntu:5555
  expire: False
runcmd:
 - echo yyyyyyyyyyyyyy >> /etc/apt/sources.list
``` 
- Cài đặt một số gói khi boot xong
```
#cloud-config
chpasswd:
  list: |
    root:123456
    ubuntu:123456
  expire: False
  
packages:
 - apache2
 - php5
 - php5-mysql
 - mysql-server

runcmd:
 - wget http://wordpress.org/latest.tar.gz -P /tmp/
 - tar -zxf /tmp/latest.tar.gz -C /var/www/
 - mysql -e "create database wordpress; create user 'wpuser'@'localhost' identified by '123456'; grant all privileges on wordpress . \* to 'wpuser'@'localhost'; flush privileges;"
 - mysql -e "drop database test; drop user 'test'@'localhost'; flush privileges;"
 - mysqladmin -u root password '123456'
 ```
###### 3.7. Sử dụng Cloud-init trong OpenStack
  Sử dụng cloud-init trong các instance của OpenStack trước hết các image đã phải cài đặt sẵn cloud-init. Bạn không cần phải cấu hình thêm gì cả, chỉ cần cài đặt và sử dụng.
ta có thể sử dụng cloud-init bằng cách sử dụng dòng lệnh hoặc dùng đồ họa
- Khi thao tác với dòng lệnh, bạn thêm tùy chon --user-data /path/to/filename vào sau lệnh nova-boot
- Khi sử dụng dashboard bạn cần thực hiện như sau:
    + chuẩn bị sẵn file cloud-config
    + thực hiện các bước tạo instance như bình thường, sau bước chọn card mạng bạn click vào tab Post-Creation rồi copy file cloud-config vào rồi click tạo là xong.

###### Link tham khảo
[Link Ubuntu](https://help.ubuntu.com/community/CloudInit)

[Link trang chủ](https://cloudinit.readthedocs.org/en/latest/)

[Link Rackspace](https://developer.rackspace.com/blog/using-cloud-init-with-rackspace-cloud/)

[Link Launchpad](http://bazaar.launchpad.net/~cloud-init-dev/cloud-init/trunk/files/head:/doc/examples/)

Chúc các bạn thành công.

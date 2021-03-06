# Hướng dẫn cài đặt và cấu hình cụm Graphite + CollectD + Grafana

Yêu cầu OS: Cài đặt trên hệ điều hành CentOS7

Mô hình
<img src="../../img/5.png">

## Install graphite web on CentOS7

Cài đặt Carbon và Graphite

```sh
yum install epel-release -y
yum --enablerepo=epel -y install graphite-web python-carbon
```

Chỉnh sửa cấu hình, bỏ comment và sửa trong file `/etc/graphite-web/local_settings.py` ở các dòng sau:

```sh
...
SECRET_KEY = 'UNSAFE_DEFAULT'
TIME_ZONE = 'Asia/Ho_Chi_Minh'
```

Cấu hình trong file `/etc/httpd/conf.d/graphite-web.conf`

```sh
...
Require local
Require ip 192.168.20.0/24 # Your ip local
...
```

#### Tạo Database

	/usr/lib/python2.7/site-packages/graphite/manage.py syncdb 

Sau đó nhập username và password để quản trị. `root:ITC*123@654`

Một số tùy chọn thao tác khác như:

* Thay đổi password cho user 
		
		/usr/lib/python2.7/site-packages/graphite/manage.py changepassword {YOUR_USERNAME}

* Tạo một username mới:

		/usr/lib/python2.7/site-packages/graphite/manage.py createsuperuser

Khởi động dịch vụ:

```sh
chown -R apache. /var/lib/graphite-web 
systemctl start carbon-cache 
systemctl enable carbon-cache 
systemctl restart httpd 
```

Nếu SELinux đang bật thì thay đổi giá trị boolean như sau:

	setsebool -P httpd_can_network_connect on

Nếu firewall đang bật:

	firewall-cmd --add-port=80/tcp
	firewall-cmd --add-port=80/tcp --permanent
	firewall-cmd --add-port=2003/tcp --permanent
	firewall-cmd --add-port=2003/udp --permanent
	firewall-cmd --reload

Nếu muốn log rotation hàng ngày thì sử cấu hình trong file sau: `/etc/carbon/carbon.conf`

	ENABLE_LOGROTATION = True

Nếu để dòng trên với giá trị `False` thì carbon sẽ tự động mở lại file cũ (ngày hôm trước) để tiếp tục ghi, còn nếu để `True` thì hàng ngày logrotate daemon sẽ được thực hiện, và carbon sẽ mổ một file mới để ghi.

#### Cấu hình file carbon-schemas.conf

	$ vim /etc/carbon/storage-schemas.conf
	...
	[default_1min_for_1day]
	pattern = .*
	retentions = 120s:120d
	...

Ở đây, tôi để cấu hình cho carbon lấy tất cả các metric đẩy về theo chu kỳ 120s lấy một lần (làm data point) và lưu trong 120 ngày

Ngoài ra có thể tham khảo thêm [ở đây](https://github.com/hocchudong/thuctap012017/blob/master/TamNT/Graphite-Collectd-Grafana/docs/5.Cai_dat_Graphite-Collectd.md)

Khởi động lại carbon-cache:

	systemctl start carbon-cache

Nếu muốn ban có thể cấu hình thêm phần `carbon-relay`, để chạy graphite cluster. Mặc định thì dịch vụ này bị tắt


## Install collectd on CentOS7

Cài đặt gói phần mềm:

	yum update
	yum install epel-release
	yum install collectd

Sửa file cấu hình: `/etc/collectd.conf`

```sh
...
Hostname    "compute2" # Đặt tên node
FQDNLookup   false
...
```

Khởi động dịch vụ:

	systemctl enable collectd
	systemctl start collectd

Cấu hình một số các plugin để đẩy log về graphite:

```sh
$ vim /etc/collectd.conf

...
LoadPlugin write_graphite
LoadPlugin unixsock
LoadPlugin virt
...

LoadPlugin interface
LoadPlugin load
LoadPlugin disk
LoadPlugin memory
Interval 120
<Plugin memory>
        ValuesAbsolute true
        ValuesPercentage false
</Plugin>

LoadPlugin network
LoadPlugin df
<Plugin df>
#       Device "/dev/sda1"
#       Device "192.168.0.2:/mnt/nfs"
#       MountPoint "/home"
#       FSType "ext3"

        # ignore rootfs; else, the root file-system would appear twice, causing
        # one of the updates to fail and spam the log
        FSType rootfs
        # ignore the usual virtual / temporary file-systems
        FSType sysfs
        FSType proc
        FSType devtmpfs
        FSType devpts
        FSType tmpfs
        FSType fusectl
        FSType cgroup
        IgnoreSelected true

#       ReportByDevice false
#       ReportReserved false
#       ReportInodes false

       ValuesAbsolute false
       ValuesPercentage true
</Plugin>

...
<Plugin unixsock>
    SocketFile "/var/run/collectd-unixsock"
    SocketGroup "collectd"
    SocketPerms "0770"
    DeleteSocket false
</Plugin>
...
<Plugin "virt">
   RefreshInterval 120
   Connection "qemu:///system"
   BlockDeviceFormat "target"
   HostnameFormat "uuid"
   InterfaceFormat "address"
   PluginInstanceFormat name
   ExtraStats "cpu_util disk_err domain_state fs_info job_stats_background perf vcpupin"
</Plugin>
...
<Plugin write_graphite>
  <Node "graphite">
	# Chanege Your ip graphite    
    Host "192.168.20.56"
    Port "2003"
    Protocol "tcp"
    LogSendErrors true
    Prefix "collectd.compute2."
    #Postfix "collectd"
    StoreRates true
    AlwaysAppendDS false
    EscapeCharacter "_"
  </Node>
</Plugin>
...
```

Xem chi tiết hơn về cấu hình một số các Plugin [ở đây](https://github.com/trangnth/Monitor/tree/master/Colletcd-Graphite-Grafana)

Nếu bị lỗi với plugin virt thi có thế khi cài đặt bị thiếu gói, cần cài thêm gói:

    yum install collectd-virt.x86_64


## Install Grafana on CentOS7

Cấu hình như sau trước khia cài đặt

```sh
cat > /etc/yum.repos.d/grafana.repo <<'EOF'
[grafana]
name=grafana
baseurl=https://packagecloud.io/grafana/stable/el/7/$basearch
gpgkey=https://packagecloud.io/gpg.key https://grafanarel.s3.amazonaws.com/RPM-GPG-KEY-grafana
enabled=0
gpgcheck=1
EOF
```

Install Grafana:

	yum install epel-release -y
	yum --enablerepo=grafana -y install grafana initscripts fontconfig


Edit the config file:

```sh
$ vim /etc/grafana/grafana.ini
...
[server]
# Protocol (http, https, socket)
;protocol = http

# The ip address to bind to, empty will bind to all interfaces
;http_addr =

# The http port  to use
;http_port = 3000
...
```

Khởi động dịch vụ:

	systemctl start grafana-server 
	systemctl enable grafana-server

Nếu firewall đang bật:

	firewall-cmd --add-port=3000/tcp --permanent 
	firewall-cmd --reload

Truy cập vào theo đường link sau: `http://<ip_server_grafana>:3000` với username và password là admin:admin.

Sau đó vào giao diện grafana add data source graphite vào để liên kết với graphite server

<img src="../../img/6.png">


## Tham khảo

https://collectd.org/wiki/index.php/Interval


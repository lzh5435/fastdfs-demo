 # 安装fastdfs

## 安装依赖
````shell
fastdfs 是c++ 语言开发
yum install gcc-c++  libevent libevent-devel -y
````
## 安装lib
````shell
tar -zxf libfastcommon-1.0.41.tar.gz 
cd libfastcommon-1.0.41
./make.sh
./make.sh install
````
## 安装fastdfs
````shell
tar -zxf fastdfs-6.02.tar.gz
cd fastdfs-6.02
./make.sh
./make.sh install
 
usr/bin 目录下会出现多个fdfs_* 命令

cd /etc/fdfs
会看到多个配置文件
client.conf.sample  storage.conf.sample  storage_ids.conf.sample  tracker.conf.sample

cd fastdfs-6.02/conf/
将http.conf mime.types 复制到/etc/fdfs
cp http.conf /etc/fdfs/
cp mime.types /etc/fdfs/

cd /etc/fdfs

````
##修改配置
````shell
先改名
cp storage.conf.sample  storage.conf
cp tracker.conf.sample tracker.conf

vim  tracker.conf 
port=22122
base_path=/opt/fdfs/tarcker   log目录  修改为自己创建得已存在的目录

vim  storage.conf 
port=23000
base_path=/opt/fdfs/storage   log目录 修改为自己创建得已存在的目录
store_path0=/opt/fdfs/storage/files   文件存储目录
tracker_server=192.168.81.166:22122  修改tracker追踪服务器的ip
````
##启动storage tracker

````shell
fdfs_trackerd /etc/fdfs/tracker.conf  [start | stop | restart]
fdfs_storaged /etc/fdfs/storage.conf  [start | stop | restart]

查看启动情况
ps -ef | grep fdfs

启动storage后 会在配置的存储目录创建data文件夹
cd  /opt/fdfs/storage/files/data
data下有256个文件夹 每个文件夹下还有256个文件夹 


pkill -9 fdfs
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf

````
开机自启
````shell
确认tracker正常启动后可以将tracker设置为开机启动，
打开/etc/rc.d/rc.local并在其中加入以下配置：
service fdfs_trackerd start

配置storage的开启自启
打开/etc/rc.d/rc.local并将如下配置追加到文件中：
service fdfs_storaged start


````

##测试
````shell
cd /etc/fdfs
cp client.conf.sample  client.conf
vim client.conf
base_path=/opt/fdfs/client
tracker_server=192.168.81.166:22122

开始上传
fdfs_test /etc/fdfs/client.conf upload aa.text 

查看 各目录内存使用情况
du --max-depth=1 -h 

这里遇到个问题 
[2020-07-11 17:34:32] ERROR - file: tracker_proto.c, line: 49, server: 192.168.81.166:22122, response status 28 != 0
[2020-07-11 17:34:32] ERROR - file: ../client/tracker_client.c, line: 1077, fdfs_recv_response fail, result: 28
[2020-07-11 17:34:32] ERROR - file: tracker_proto.c, line: 49, server: 192.168.81.166:22122, response status 28 != 0
[2020-07-11 17:34:32] ERROR - file: ../client/tracker_client.c, line: 899, fdfs_recv_response fail, result: 28
tracker_query_storage fail, error no: 28, error info: No space left on device
说磁盘空间不足
修改 tracker.conf 里面 reserved_storage_space  这是预留磁盘大小 缺省4GB 写着20% 改小点
常见问题  https://blog.csdn.net/xyw591238/article/details/51487710
````
重启 tracker storage
````shell
再次上传 
fdfs_test /etc/fdfs/client.conf upload aa.tex
出现这些信息 代表成功
tracker_query_storage_store_list_without_group: 
	server 1. group_name=, ip_addr=192.168.81.166, port=23000

group_name=group1, ip_addr=192.168.81.166, port=23000
storage_upload_by_filename
group_name=group1, remote_filename=M00/00/00/wKhRpl8JiJ-AUVLXAAAAHBDfNP056.text
source ip address: 192.168.81.166
file timestamp=2020-07-11 17:38:39
file size=28
file crc32=283063549
example file url: http://192.168.81.166/group1/M00/00/00/wKhRpl8JiJ-AUVLXAAAAHBDfNP056.text
storage_upload_slave_by_filename
group_name=group1, remote_filename=M00/00/00/wKhRpl8JiJ-AUVLXAAAAHBDfNP056_big.text
source ip address: 192.168.81.166
file timestamp=2020-07-11 17:38:39
file size=28
file crc32=283063549
example file url: http://192.168.81.166/group1/M00/00/00/wKhRpl8JiJ-AUVLXAAAAHBDfNP056_big.text

````
fastdfs 无法使用http直接访问

##安装nginx 和 扩展模块
````shell
tar -zxvf nginx
tar -zxvf fastdfs-nginx-modle-master

cd fastdfs-nginx-modle-master/src
pwd
复制src路径  用于下方--add-module

进入nginx解压后的目录并输入以下命令进行配置：
./configure  --prefix=/usr/local/nginx  --add-module=/root/fastdfs-nginx-module-1.21/src
make
make install

````

##配置扩展模块
````shell
yum install -y openssl openssl-devel pcre pcre-devel zlib zlib-devel
将扩展模块 下的 mod_fastdfs.conf 复制到 /etc/fdfs/
cp -r /root/fastdfs-nginx-module-1.21/src/mod_fastdfs.conf /etc/fdfs/

vim mod_fastdfs.conf
1.	base_path=/opt/fdfs/nginx_mod #保存日志目录
2.	tracker_server=192.168.81.166:22122 #tracker服务器的IP地址以及端口号
3.	url_have_group_name = true #文件 url 中是否有 group 名
4.	store_path0=/opt/fdfs/storage/files # 存储路径

````

## 配置nginx
````shell
cd /usr/local/nginx/conf
vim nginx.conf
在server中 增加

listen    6789; 可以修改访问端口
location ~/group[0-9]/M0[0-9] {
     ngx_fastdfs_module;
  }

启动 nginx
cd /usr/local/nginx/sbin
./nginx 
关闭nginx进程
nginx -s stop
启动nginx进程
/usr/sbin/nginx     yum安装的nginx也可以使用      servic nginx start
检查配置文件是否有误
./nginx –t
重新加载配置文件
./nginx –s reload
````
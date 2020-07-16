#FastDFS 集群

图(\static\集群架构.jpg)

2个tracker 4个storage (2个group1,2个group2)

##全部安装依赖

```shell
yum install -y gcc perl openssl openssl-devel pcre pcre-devel zlib zlib-devel libevent libevent-devel  lrzsz wget vim unzip net-tools
```

##全部安装 libfastcommon

```shell
tar -zxvf libfastcommon
cd libfastcommon
./make.sh
./make.sh install

```

##全部安装fastdfs

```shell
tar -zxvf fastdfs
cd fastdfs
./make
./make install
cd conf
cp http.conf /etc/fdfs
cp mime.types /etc/fdfs

```


##tracker服务器
```shell
mkdir -p /opt/fastdfs/tracker

cd /etc/fdfs
mv tracker.conf.sample tracker.conf

vim tracker.conf
# 日志文件路径 必须存在
base_path=/opt/fastdfs/tracker

#启动
fdfs_trackerd /etc/fdfs/tracker.conf  [start | stop | restart]

#查看是否成功
ps -ef | grep fdfs

```

##tracker服务器安装nginx
```shell
########  nginx #########

tar -zxvf nginx
cd nginx
./configure  --prefix=/usr/local/nginx
make
make install

# 修改nginx配置文件
cd /usr/local/nginx/conf
vim nginx.conf
#在server上增加  各个storage所在服务器ip 和 所配nginx的监听端口
upstream fastdfs_storage_server {
   server ip:port;
   server ip:port;
   server ip:port;
}

#在server中 增加
listen    6666; 可以修改访问端口
location ~/group[0-9]/M0[0-9] {
    proxy_pass http://fastdfs_storage_server
}

#启动
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf -t #检测配置文件
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf 

```


##storage服务器
```shell
mkdir -p /opt/fastdfs/storage
mkdir -p /opt/fastdfs/files

cd /etc/fdfs
mv storage.conf.sample storage.conf

vim storage.conf
# 日志文件路径 必须存在
base_path=/opt/fastdfs/storage  

# 文件存储路径  必须存在
store_path0=/opt/fastdfs/files

#tracker服务地址 集群多个就写多个tracker_server=
tracker_server=ip:22122

######group名 在group1中是group1 在group2中是group2
group_name=group1

#启动
fdfs_storaged /etc/fdfs/storage.conf  [start | stop | restart]

#查看是否成功
ps -ef | grep fdfs

```
##storage服务器安装nginx和fastdfs扩展模块
```shell
############  nginx + fastdfs-modle ############

tar -zxvf nginx
tar -zxvf fastdfs-nginx-modle-master
cd fastdfs-nginx-modle-master/src

# 复制扩展模块的配置文件到/etc/fdfs下
cp mod_fastdfs.conf /etc/fdfs

pwd
复制src路径  用于下方--add-module

cd ../../nginx
./configure  --prefix=/usr/local/nginx  --add-module=/root/fastdfs-nginx-module-1.21/src
make
make install

# 修改nginx配置文件
cd /usr/local/nginx/conf
vim nginx.conf
#在server中 增加
listen    6789; 可以修改访问端口
location ~/group[0-9]/M0[0-9] {
     ngx_fastdfs_module;
}

# 修改modle配置文件
cd /etc/fdfs
mkdir -p /opt/fdfs/nginx_mod 
vim mod_fastdfs.conf
1. base_path=/opt/fdfs/nginx_mod #保存日志目录 必须存在
2. tracker_server=192.168.81.166:22122 #tracker服务器的IP地址以及端口号 有几个写几个tracker_server
3. url_have_group_name = true #文件 url 中是否有 group 名
4. store_path0=/opt/fastdfs/files # 存储路径
5. group_count=2 #有几个写几个 这里有2个
6. 在最后添加group*的信息
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/opt/fastdfs/files
[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/opt/fastdfs/files

```

构建成功后 每个服务器的ip都可以访问文件 
此时需要配置一个统一的入口
在单独的服务器上再配置一个nginx
upstream 指向 tracker 服务器




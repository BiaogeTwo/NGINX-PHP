### 一、基本信息
- 系统（L）：CentOS 6.9			#下载地址：[http://mirrors.sohu.com](http://mirrors.sohu.com)
- Web服务器(N)：NGINX 1.14.0 		#下载地址：[http://nginx.org/en/download.html](http://nginx.org/en/download.html)
- 数据库服务器(M)：MySQL 5.6.40	#下载地址：[https://dev.mysql.com/downloads/mysql](https://dev.mysql.com/downloads/mysql)
- PHP-FPM服务器(P)：php-5.6.8.tar.gz	#下载地址：[http://mirrors.sohu.com/php/](http://mirrors.sohu.com/php/)
- OPENSSL：openssl-1.0.2o.tar.gz	#下载地址：[https://www.openssl.org/source/](https://www.openssl.org/source/)

###### 指定服务安装的通用位置
```
mkdir /usr/local/services
SERVICE_PATH=/usr/local/services
```
###### 创建服务运行的账户
```
useradd -r -M -s /sbin/nologin www
```
###### 安装所需依赖包
```
yum -y install pcre pcre-devel \
gperftools gcc zlib-devel \
libxml2 libxml2-devel \
bzip2 bzip2-devel \
curl curl-devel \
libjpeg-devel libjpeg  \
libpng-devel libpng \
freetype freetype-devel \
libmcrypt libmcrypt-devel \
openssl-devel
```
### 二、软件安装配置
##### 1、NGINX+OPENSSL安装

###### 下载解压NGINX+OPENSSL
```
NGINX_URL="http://nginx.org/download/nginx-1.14.0.tar.gz"
OPENSSL_URL="https://www.openssl.org/source/openssl-1.1.0h.tar.gz"

wget -P ${SERVICE_PATH} ${NGINX_URL} && tar -zxvf  ${SERVICE_PATH}/nginx*.tar.gz -C ${SERVICE_PATH}
wget -P ${SERVICE_PATH} ${OPENSSL_URL} && tar -zxvf ${SERVICE_PATH}/openssl*.gz -C ${SERVICE_PATH}
```

###### 编译安装NGINX
```
cd ${SERVICE_PATH}/nginx-*;./configure \
--prefix=${SERVICE_PATH}/nginx \
--user=www --group=www \
--with-http_stub_status_module \
--with-http_ssl_module \
--with-http_flv_module \
--with-pcre \
--with-http_gzip_static_module \
--with-openssl=${SERVICE_PATH}/openssl-* \
--with-http_realip_module \
--with-google_perftools_module \
--without-select_module \
--without-mail_pop3_module \
--without-mail_imap_module \
--without-mail_smtp_module \
--without-poll_module \
--without-http_autoindex_module \
--without-http_geo_module \
--without-http_uwsgi_module \
--without-http_scgi_module \
--without-http_memcached_module \
--with-cc-opt='-O2' && cd ${SERVICE_PATH}/nginx-*;make && make install
```

###### 写入主配置文件nginx.conf（配置已优化）
```
cat << EOF > ${SERVICE_PATH}/nginx/conf/nginx.conf
user  www;
worker_processes  WORKERNUMBER;
worker_cpu_affinity auto;
worker_rlimit_nofile 655350;

error_log  /var/log/nginx_error.log;
pid  /tmp/nginx.pid;

google_perftools_profiles /tmp/tcmalloc;

events {
    use epoll;
    worker_connections  655350;
    multi_accept on;
}

http {
        charset  utf-8;
        include       mime.types;
        default_type  text/html;

       log_format  main  '"\$remote_addr" - [\$time_local] "\$request" '
                                      '\$status $body_bytes_sent "\$http_referer" '
                                      '"\$http_user_agent" '
                                      '"\$sent_http_server_name \$upstream_response_time" '
                                      '\$request_time '
                                      '\$args';


        sendfile        on;
        tcp_nopush     on;
        tcp_nodelay on;
        keepalive_timeout  120;
        client_body_buffer_size 512k;
        client_header_buffer_size 64k;
        large_client_header_buffers 4 32k;
        client_max_body_size 300M;
        client_header_timeout 15s;
        client_body_timeout 50s;
        open_file_cache max=102400 inactive=20s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 1;
        server_names_hash_max_size 2048;
        server_names_hash_bucket_size 256;
        server_tokens off;
        gzip  on;
        gzip_proxied any;
        gzip_min_length  1024;
        gzip_buffers     4 8k;
        gzip_comp_level 9;
        gzip_disable "MSIE [1-6]\.";
        gzip_types application/json test/html text/plain text/css application/font-woff  application/pdf application/octet-stream application/x-javascript application/javascript application/xml text/javascript;
        fastcgi_cache_path /dev/shm/ levels=1:2 keys_zone=fastcgicache:512m inactive=10m max_size=3g;
        fastcgi_cache_lock on;
        fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
        fastcgi_send_timeout 300;
        fastcgi_connect_timeout 300;
        fastcgi_read_timeout 300;
        fastcgi_buffer_size 256k;
        fastcgi_buffers 4 128k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;


        include vhost/*.conf;
}
EOF
```
###### NGINX worker进程数配置，指定为逻辑CPU数量的2倍
```
THREAD=`expr $(grep process /proc/cpuinfo |wc -l) \* 2`
sed -i s"/WORKERNUMBER/$THREAD/" ${SERVICE_PATH}/nginx/conf/nginx.conf
```
##### 2、PHP-FPM安装
###### 下载并解压PHP-FPM软件
```
FPM_URL="http://mirrors.sohu.com/php/php-5.6.8.tar.gz"
wget -P ${SERVICE_PATH} ${FPM_URL} && tar -zxvf ${SERVICE_PATH}/php*.tar.gz -C ${SERVICE_PATH}
```
###### 编译安装PHP-FPM
```
cd ${SERVICE_PATH}/php-*;./configure \
--prefix=${SERVICE_PATH}/php  \
--with-gd  \
--with-mcrypt  \
--with-mysql=mysqlnd  \
--with-mysqli=mysqlnd  \
--with-pdo-mysql=mysqlnd  \
--enable-maintainer-zts  \
--enable-ftp  \
--enable-zip  \
--with-bz2  \
-with-iconv-dir  \
--with-freetype-dir  \
--with-jpeg-dir  \
--with-png-dir  \
--with-config-file-path=${SERVICE_PATH}/php  \
--enable-mbstring  \
--enable-fpm \
--with-fpm-user=www \
--with-fpm-group=www \
--disable-debug  \
--enable-opcache  \
--enable-soap  \
--with-zlib  \
--with-libxml-dir=/usr  \
--enable-xml  \
--disable-rpath  \
--enable-bcmath  \
--enable-shmop  \
--enable-sysvsem  \
--enable-inline-optimization  \
--with-curl  \
--enable-mbregex  \
--enable-gd-native-ttf  \
--with-openssl \
--with-mhash  \
--enable-pcntl  \
--enable-sockets  \
--with-xmlrpc  \
--with-pear  \
--with-gettext  \
--disable-fileinfo  && cd ${SERVICE_PATH}/php-*;make && make install
```

###### 若FPM程序有插件需求，如mongo或redis连接插件，则可通过pecl安装php相关插件
###### ${SERVICE_PATH}/php/bin/pecl install mongo || exit
###### ${SERVICE_PATH}/php/bin/pecl install redis || exit


###### php.ini配置文件写入（配置已优化）
```
cat << EOF >${SERVICE_PATH}/php/php.ini 
[PHP]
engine = On
short_open_tag = Off
asp_tags = Off
precision = 14
output_buffering = 4096
zlib.output_compression = Off
implicit_flush = Off
unserialize_callback_func =
serialize_precision = 17
disable_functions = shell_exec,phpinfo,exec
disable_classes =
zend.enable_gc = On
expose_php = Off
max_execution_time = 60
max_input_time = 60
memory_limit = 128M
error_reporting = E_WARING & ERROR
display_errors = Off
display_startup_errors = Off
log_errors = On
log_errors_max_len = 2048
ignore_repeated_errors = Off
ignore_repeated_source = Off
report_memleaks = On
track_errors = Off
html_errors = Off
error_log = /var/log/php_errors.log
variables_order = "GPCS"
request_order = "GP"
register_argc_argv = Off
auto_globals_jit = On
post_max_size = 8M
auto_prepend_file =
auto_append_file =
default_mimetype = "text/html"
default_charset = "UTF-8"
doc_root =
user_dir =
enable_dl = Off
cgi.fix_pathinfo=0
file_uploads = On
upload_max_filesize = 2M
max_file_uploads = 20
allow_url_fopen = Off
allow_url_include = Off
default_socket_timeout = 60
[CLI Server]
cli_server.color = On
[Date]
[filter]
[iconv]
[intl]
[sqlite]
[sqlite3]
[Pcre]
[Pdo]
[Pdo_mysql]
pdo_mysql.cache_size = 2000
pdo_mysql.default_socket=
[Phar]
[mail function]
SMTP = localhost
smtp_port = 25
mail.add_x_header = On
[SQL]
sql.safe_mode = Off
[ODBC]
odbc.allow_persistent = On
odbc.check_persistent = On
odbc.max_persistent = -1
odbc.max_links = -1
odbc.defaultlrl = 4096
odbc.defaultbinmode = 1
[Interbase]
ibase.allow_persistent = 1
ibase.max_persistent = -1
ibase.max_links = -1
ibase.timestampformat = "%Y-%m-%d %H:%M:%S"
ibase.dateformat = "%Y-%m-%d"
ibase.timeformat = "%H:%M:%S"
[MySQL]
mysql.allow_local_infile = On
mysql.allow_persistent = On
mysql.cache_size = 2000
mysql.max_persistent = -1
mysql.max_links = -1
mysql.default_port =
mysql.default_socket =
mysql.default_host =
mysql.default_user =
mysql.default_password =
mysql.connect_timeout = 60
mysql.trace_mode = Off
[MySQLi]
mysqli.max_persistent = -1
mysqli.allow_persistent = On
mysqli.max_links = -1
mysqli.cache_size = 2000
mysqli.default_port = 3306
mysqli.default_socket =
mysqli.default_host =
mysqli.default_user =
mysqli.default_pw =
mysqli.reconnect = Off
[mysqlnd]
mysqlnd.collect_statistics = On
mysqlnd.collect_memory_statistics = Off
[OCI8]
[PostgreSQL]
pgsql.allow_persistent = On
pgsql.auto_reset_persistent = Off
pgsql.max_persistent = -1
pgsql.max_links = -1
pgsql.ignore_notice = 0
pgsql.log_notice = 0
[Sybase-CT]
sybct.allow_persistent = On
sybct.max_persistent = -1
sybct.max_links = -1
sybct.min_server_severity = 10
sybct.min_client_severity = 10
[bcmath]
bcmath.scale = 0
[browscap]
[Session]
session.save_handler = files
session.save_path = "/tmp"
session.use_strict_mode = 0
session.use_cookies = 1
session.use_only_cookies = 1
session.name = PHPSESSID
session.auto_start = 0
session.cookie_lifetime = 0
session.cookie_path = /
session.cookie_domain =
session.cookie_httponly =
session.serialize_handler = php
session.gc_probability = 1
session.gc_divisor = 1000
session.gc_maxlifetime = 1440
session.referer_check =
session.cache_limiter = nocache
session.cache_expire = 180
session.use_trans_sid = 0
session.hash_function = 0
session.hash_bits_per_character = 5
url_rewriter.tags = "a=href,area=href,frame=src,input=src,form=fakeentry"
[MSSQL]
mssql.allow_persistent = On
mssql.max_persistent = -1
mssql.max_links = -1
mssql.min_error_severity = 10
mssql.min_message_severity = 10
mssql.compatibility_mode = Off
mssql.secure_connection = Off
[Assertion]
[COM]
[mbstring]
[gd]
gd.jpeg_ignore_warning = 0
[exif]
[Tidy]
tidy.clean_output = Off
[soap]
soap.wsdl_cache_enabled=1
soap.wsdl_cache_dir="/tmp"
soap.wsdl_cache_ttl=86400
soap.wsdl_cache_limit = 5
[sysvshm]
[ldap]
ldap.max_links = -1
[mcrypt]
[dba]
[opcache]
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.validate_timestamps=1
opcache.revalidate_freq=30
opcache.fast_shutdown=1
opcache.enable_file_override=1
[curl]
[openssl]
extension_dir='${SERVICE_PATH}/php/lib/php/extensions/'
;extension=mongo.so
;extension=redis.so
EOF
```


###### php-fpm.conf配置文件写入（配置已优化）
```
cat << EOF >${SERVICE_PATH}/php/etc/php-fpm.conf
[global]
error_log = /var/log/php-fpm-error.log
log_level = warning
process_control_timeout = 10
rlimit_files = 655350
events.mechanism = epoll
[www]
user = www
group = www
listen = /dev/shm/php-fpm.sock
listen.backlog = 2048
listen.owner = www
listen.group = www
listen.mode = 0660
pm = dynamic
pm.max_children = 200
pm.start_servers = 105
pm.min_spare_servers = 10
pm.max_spare_servers = 200
pm.process_idle_timeout = 10s;
pm.max_requests = 1000
pm.status_path = /fpmstatus
ping.path = /ping
ping.response = pong
slowlog = /var/log/php-slow-$pool.log
request_slowlog_timeout = 10
request_terminate_timeout = 0
rlimit_files = 655350
security.limit_extensions = .php
EOF
```
### 三、基于以上配置PHP网站
`
mkdir /usr/local/nginx/conf/vhost
`
```
cat << EOF > /usr/local/nginx/conf/vhost/erbiao.ex.com.conf
server
        {
                listen 80 backlog=1024;
                server_name erbiao.ex.com;
                index index.php index.html ;
                root  /www/web/;
                access_log off;
                add_header Server-Name WEBerbiaoEX;

                location ~ \.php {
                   fastcgi_pass  unix:/dev/shm/php-fpm.sock;
                   fastcgi_index index.php;
                   include fastcgi.conf;
                   set \$real_script_name \$fastcgi_script_name;
                   if (\$fastcgi_script_name ~ "^(.+?\.php)(/.+)\$") {
                   set \$real_script_name \$1;
                   set \$path_info \$2;
          }
                   fastcgi_param SCRIPT_FILENAME \$document_root\$real_script_name;
                   fastcgi_param SCRIPT_NAME \$real_script_name;
                   fastcgi_param PATH_INFO \$fastcgi_path_info;


                location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
                        {
                                expires      30d;
                        }

                location ~ .*\.(js|css)?$
                        {
                                expires      12h;
                        }

        }

}
EOF
```
####### 若在同一服务器运行nginx和php-fpm，并发量不超过1000，选择unix socket，如此可避免一些检查操作(路由等)，因此更快，更轻。若是高并发业务，则选择使用更可靠的tcp socket，以负载均衡、内核优化等运维手段维持效率

### 四、安装完成后的清理与生成目录快捷方式
```
rm -rf ${SERVICE_PATH}/{nginx*.tar.gz,openssl*.tar.gz,php*.tar.gz}
rm -rfv ${SERVICE_PATH}/nginx/conf/*.default
rm -rfv ${SERVICE_PATH}/php/etc/*.default

ln -sv ${SERVICE_PATH}/nginx /usr/local/
ln -sv ${SERVICE_PATH}/php /usr/local/
```

### 五、启动服务
###### 生成php-fpm系统服务脚本，并加入开机启动项
```
ln -sv ${SERVICE_PATH}/php-*/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod a+x /etc/init.d/php-fpm
chkconfig php-fpm --add && chkconfig php-fpm on
```
###### 生成nginx系统服务脚本，并加入开机启动项
```
cat << EOF > /etc/init.d/nginx
#!/bin/bash
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig: - 85 15
# description: Nginx is an HTTP(S) server, HTTP(S) reverse
# proxy and IMAP/POP3 proxy server
# processname: nginx
# config: /etc/nginx/nginx.conf
# config: /etc/sysconfig/nginx
# pidfile: /var/run/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

#create temp dir in memory
mkdir -p /dev/shm/nginx/fastcgi_temp/

# Check that networking is up.
[ "\$NETWORKING" = "no" ] && exit 0

TENGINE_HOME="/usr/local/nginx/"
nginx=\$TENGINE_HOME"sbin/nginx"
prog=\$(basename \$nginx)

NGINX_CONF_FILE=\$TENGINE_HOME"conf/nginx.conf"

[ -f /etc/sysconfig/nginx ] && /etc/sysconfig/nginx

lockfile=/var/lock/subsys/nginx

start() {
    [ -x \$nginx ] || exit 5
    [ -f \$NGINX_CONF_FILE ] || exit 6
    echo -n \$"Starting \$prog: "
    daemon \$nginx -c \$NGINX_CONF_FILE
    retval=\$?
    echo
    [ \$retval -eq 0 ] && touch \$lockfile
    return \$retval
}

stop() {
    echo -n \$"Stopping \$prog: "
    killproc \$prog -QUIT
    retval=\$?
    echo
    [ \$retval -eq 0 ] && rm -f \$lockfile
    return \$retval
    killall -9 nginx
}

restart() {
    configtest || return \$?
    stop
    sleep 1
    start
}

reload() {
    configtest || return \$?
    echo -n \$"Reloading \$prog: "
    killproc \$nginx -HUP
    RETVAL=\$?
    echo
}

force_reload() {
    restart
}

configtest() {
    \$nginx -t -c \$NGINX_CONF_FILE
}

rh_status() {
    status \$prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "\$1" in
start)
    rh_status_q && exit 0
    \$1
;;
stop)
    rh_status_q || exit 0
    \$1
;;
restart|configtest)
    \$1
;;
reload)
    rh_status_q || exit 7
        \$1
;;
force-reload)
    force_reload
;;
status)
    rh_status
;;
condrestart|try-restart)
    rh_status_q || exit 0
;;
*)

echo \$"Usage: \$0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
exit 2
esac
EOF

chmod a+x /etc/init.d/nginx
chkconfig nginx --add && chkconfig nginx on
```
###### 启动服务
```
service nginx restart
service php-fpm restart
```
###### 命令其他选项
```
nginx
├── -s选项，向主进程发送信号
|	├── reload参数，重新加载配置文件
|	├── stop参数，快速停止nginx
|	├── reopen参数，重新打开日志文件
|	├── quit参数，Nginx在退出前完成已经接受的连接请求
├── -t选项，检查配置文件是否正确
├── -c选项，用于指定特定的配置文件并启动nginx
├── -V选项（大写），显示nginx编译选项与版本信息

php-fpm
├── -t选项，检查配置文件是否正确
├── -m选项，显示所有已安装模块
├── -i选项，显示PHP详细信息
├── -v选项，显示版本信息
```

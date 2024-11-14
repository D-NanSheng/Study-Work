Tengein以及xQuic安装过程及遇到的问题

**安装教程：**

参考链接

[https://github.com/TheDarkArchmageShangYang/networkLearning/blob/main/工具的安装及使用/xquic服务器使用.md]

(https://github.com/alibaba/tengine/blob/master/modules/ngx_http_xquic_module/README.md)

注意，安装的时候，xquic版本最好按照下面的教程，否可可能出现接口不一致的问题。

注意，xquic官方提供了两种ssl库，最好选择Tongsuo，选择boringssl也可能会出现接口不一致的问题，此外boringssl本身并不提倡别的应用使用它作为第三方库。

```
# 下载 Tongsuo，示例中下载 8.3.2 版本
wget -c "https://github.com/Tongsuo-Project/Tongsuo/archive/refs/tags/8.3.2.tar.gz"
tar -x 8.3.2.tar.gz

# 下载 xquic，示例中下载 1.6.0 版本
wget -c "https://github.com/alibaba/xquic/archive/refs/tags/v1.6.0.tar.gz"
tar -xf v1.6.0.tar.gz

# 下载 Tengine 3.0.0 以上版本，示例从 master 获取最新版本，也可下载指定版本
git clone git@github.com:alibaba/tengine.git

# 编译 Tongsuo
cd Tongsuo-8.3.2
./config --prefix=/usr/local/babassl  (ps 路径可自定义，但必须使用绝对路径)
make
make install
export SSL_TYPE_STR="babassl"
export SSL_PATH_STR="${PWD}"
export SSL_INC_PATH_STR="${PWD}/include"
export SSL_LIB_PATH_STR="${PWD}/libssl.a;${PWD}/libcrypto.a"
cd ../../

# 编译 xquic 库
cd xquic-1.6.0/
mkdir -p build; cd build
cmake -DXQC_SUPPORT_SENDMMSG_BUILD=1 -DXQC_ENABLE_TESTING=1 -DXQC_ENABLE_BBR2=1 -DXQC_DISABLE_RENO=0 -DSSL_TYPE=${SSL_TYPE_STR} -DSSL_PATH=${SSL_PATH_STR} -DSSL_INC_PATH=${SSL_INC_PATH_STR} -DSSL_LIB_PATH=${SSL_LIB_PATH_STR} ..
make
cp "libxquic.so" /usr/local/lib/     (这个一定要执行，或者把这个库所在地址设置一个环境变量)
cd ..

# 编译 Tengine
cd tengine

# 注：xquic 依赖 ngx_http_v2_module，需要参数 --with-http_v2_module
# 注：注意下面的 **..** 是安装对应文件的路径
./configure \
  --prefix=/usr/local/tengine \ (ps 路径可自定义 <**somewhere-tengine**>)
  --sbin-path=sbin/tengine \
  --with-xquic-inc="../xquic-1.6.0/include" \
  --with-xquic-lib="../xquic-1.6.0/build" \
  --with-http_v2_module \
  --without-http_rewrite_module \
  --add-module=modules/ngx_http_xquic_module \
  --with-openssl="../Tongsuo-8.3.2" \
  --with-http_mp4_module \ (ps 点播mp4需要)
  --add-module=modules/nginx-http-flv-module \ (ps rtmp推流，http-flv拉流需要，下载地址 https://github.com/winshining/nginx-http-flv-module/tree/master)

make
make install

```


依据nginx.conf配置启动tengine，注意，这里的<**/usr/local/tengine**> 是上面安装过程中的<**somewhere-tengine**>

`/usr/local/tengine/sbin/tengine -c /usr/local/tengine/conf/nginx.conf`



一个可用于xquic传输的配置
**nginx.conf**
```
worker_processes  1;
user root;
error_log  logs/error.log debug;
events {
    worker_connections  1024;
}
xquic_log   "pipe:rollback /home/guopeisheng/workbench/QuicBBR/tengine-xquic.log baknum=10 maxsize=1G interval=1d adjust=600" debug;

http {
    include       mime.types;
    default_type  application/octet-stream;
    xquic_ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.crt;
    xquic_ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.key;
    xquic_congestion_control bbr;
    xquic_socket_rcvbuf 5242880;
    xquic_socket_sndbuf 5242880;
    xquic_anti_amplification_limit 5;
    xquic_log_level debug;
    server {
        # listen 10.37.30.62:80;
        listen 8000 ssl http2;
        listen 8000 xquic reuseport;
        server_name guopeisheng;
        
        add_header Alt-Svc 'h3=":8000"; ma=2592000,h3-29=":8000"; ma=2592000;' always;
        
        ssl_protocols       TLSv1.3;
        ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.crt;
        ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.key;

        location / {
            root   html;
            index  index.html index.htm;

            autoindex on;
            autoindex_exact_size   on;
        }
    }
}
```


**ssl_certificate 文件生成**

参考 https://letsencrypt.org/zh-cn/docs/certificates-for-localhost/


**验证Quic通信的第一种方式**

利用xquic文件夹build中的测试客户端测试

`./test_client -a 127.0.0.1 -p 8000 -u https://guopeisheng.com/`


**验证Quic通信的第二种方式**

支持quic的curl：https://github.com/TheDarkArchmageShangYang/networkLearning/blob/main/工具的安装及使用/支持quic的curl的安装和使用.md

注意，**http3访问时必须带域名**，不能仅通过ip+端口访问。

由于我们时测试证书，DNS无法解析域名，所以，最好在本地域名转换文件中添加把域名转为ip的条目。

**mp4点播的配置**
```
worker_processes  1;
user root;
error_log  logs/error.log debug;
events {
    worker_connections  1024;
}
xquic_log   "pipe:rollback /home/guopeisheng/workbench/QuicBBR/tengine-xquic.log baknum=10 maxsize=1G interval=1d adjust=600" debug;

http {
    include       mime.types;
    default_type  application/octet-stream;
    xquic_ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.crt;
    xquic_ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.key;
    xquic_congestion_control bbr;
    xquic_socket_rcvbuf 5242880;
    xquic_socket_sndbuf 5242880;
    xquic_anti_amplification_limit 5;
    xquic_log_level debug;
    server {
        # listen 10.37.30.62:80;
        listen 10.37.30.62:8000 ssl http2;
        listen 10.37.30.62:8000 xquic reuseport;
        server_name guopeisheng;
        
        add_header Alt-Svc 'h3=":8000"; ma=2592000,h3-29=":8000"; ma=2592000;' always;
        
        ssl_protocols       TLSv1.3;
        ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.crt;
        ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.key;

        location / {
            root   html;
            index  index.html index.htm;

            autoindex on;
            autoindex_exact_size   on;
        }
        location /video/ {
            alias /usr/local/tengine/video/;  # 指定了绝对路径
            mp4;
            mp4_buffer_size       1m;
            mp4_max_buffer_size   5m;
            mp4_start_key_frame on;
            # mp4_limit_rate        on; # 商业版才有
            # mp4_limit_rate_after  30s; # 商业版才有
        }
    }
```

**rtmp推流+rtmp/hls/dash/http-flv拉流**

参考 https://github.com/winshining/nginx-http-flv-module/tree/master)

```
worker_processes  1;
user root;
error_log  logs/error.log debug;
events {
    worker_connections  1024;
}
xquic_log   "pipe:rollback /home/guopeisheng/workbench/QuicBBR/tengine-xquic.log baknum=10 maxsize=1G interval=1d adjust=600" debug;
# xuqic http 配置
http {
    include       mime.types;
    default_type  application/octet-stream;
    #keepalive_timeout  65;

    xquic_ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/server.crt;
    xquic_ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/server.key;
    xquic_congestion_control bbr;
    xquic_socket_rcvbuf 5242880;
    xquic_socket_sndbuf 5242880;
    xquic_anti_amplification_limit 5;
    xquic_log_level debug;
    server {
        listen 8000 ssl http2; # 在浏览器访问时，通过http2转quic
        listen 8000 xquic reuseport;
        server_name test.guopeisheng.com; # 域名
        
        add_header Alt-Svc 'h3=":8000"; ma=2592000,h3-29=":8000"; ma=2592000;' always;
        
        ssl_protocols       TLSv1.3;# quic只能用1.3
        ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.pem;
        ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.pem;
        # 根文件
        location / {
            root   html;
            index  index.html index.htm;

            autoindex on;
            autoindex_exact_size   on;
        }

        # http-flv直播 https://test.guopeisheng.com:8000/live?port=1935&app=myapp&stream=stream-123456
        location /live {
            proxy_pass http://本机ip或域名:80/live;  # 将请求代理到目标服务器的指定路径，注意要与目标服务器提供服务的路径一致，这里去掉了后面的查询参数，因为查询参数会在代理过程中自动传递
            # proxy_set_timeout 300;  # 设置代理请求的超时时间，可根据实际情况调整
            # flv_live on; #打开 HTTP 播放 FLV 直播流功能
            # chunked_transfer_encoding on; #支持 'Transfer-Encoding: chunked' 方式回复

            # add_header 'Access-Control-Allow-Origin' '*'; #添加额外的 HTTP 头
            # add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的 HTTP 头
            # #add_header Access-Control-Allow-Headers X-Requested-With;
            # add_header Access-Control-Allow-Methods 'GET,POST,OPTIONS';
            # add_header 'Cache-Control' 'no-cache';
        }
        # location /hls {
        #     types {
        #         application/vnd.apple.mpegurl m3u8;
        #         video/mp2t ts;
        #     }

        #     root /tmp;
        #     add_header 'Cache-Control' 'no-cache';
        # }

        # location /dash {
        #     root /tmp;
        #     add_header 'Cache-Control' 'no-cache';
        # }

        # location /stat {
        #     #推流播放和录制统计数据的配置

        #     rtmp_stat all;
        #     rtmp_stat_stylesheet stat.xsl;
        # }

        # location /stat.xsl {
        #     root /var/www/rtmp; #指定 stat.xsl 的位置
        # }

        # #如果需要 JSON 风格的 stat, 不用指定 stat.xsl
        # #但是需要指定一个新的配置项 rtmp_stat_format

        # #location /stat {
        # #    rtmp_stat all;
        # #    rtmp_stat_format json;
        # #}

        # location /control {
        #     rtmp_control all; #rtmp 控制模块的配置
        # }

    }

    server {
        listen       10.37.30.62:80 default;
        #listen 10.37.30.62:8080 ssl;

        # ssl_protocols       TLSv1.3;
        # ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.pem;
        # ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.pem;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        location /live {
            flv_live on; #打开 HTTP 播放 FLV 直播流功能
            chunked_transfer_encoding on; #支持 'Transfer-Encoding: chunked' 方式回复

            add_header 'Access-Control-Allow-Origin' '*'; #添加额外的 HTTP 头
            add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的 HTTP 头
        }

        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }

            root /tmp;
            add_header 'Cache-Control' 'no-cache';
        }

        location /dash {
            root /tmp;
            add_header 'Cache-Control' 'no-cache';
        }

        location /stat {
            #推流播放和录制统计数据的配置

            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            root /var/www/rtmp; #指定 stat.xsl 的位置
        }

        #如果需要 JSON 风格的 stat, 不用指定 stat.xsl
        #但是需要指定一个新的配置项 rtmp_stat_format

        #location /stat {
        #    rtmp_stat all;
        #    rtmp_stat_format json;
        #}

        location /control {
            rtmp_control all; #rtmp 控制模块的配置
        }
    }

}

rtmp_auto_push on;
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp;
# rtmp推流 
# ffmpeg -re -i MEDIA_FILE_NAME -c copy -f flv rtmp://example.com[:port]/appname/streamname   [:port]是指rtmp推流的端口，默认1935
rtmp {
    out_queue           4096;
    out_cork            8;
    max_streams         128;
    timeout             15s;
    drop_idle_publisher 15s;

    log_interval 5s; #log 模块在 access.log 中记录日志的间隔时间，对调试非常有用
    log_size     1m; #log 模块用来记录日志的缓冲区大小
   
    server {
        listen 1935;  # RTMP服务监听端口
        chunk_size 4000;
         # RTMP播放--rtmp://example.com[:port]/appname/streamname
        application myapp {
            live on;
            gop_cache on; #打开 GOP 缓存，减少首屏等待时间
        }
        # HLS播放--http://example.com[:port]/dir/streamname.m3u8
        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
        }
        # DASH播放——http://example.com[:port]/dir/streamname.mpd
        application dash {
            live on;
            dash on;
            dash_path /tmp/dash;
        }
    }
}



**剩余未解决问题**
1. 浏览器无法将http2切换成quic。
2. 将这些组件放到Mininet中。

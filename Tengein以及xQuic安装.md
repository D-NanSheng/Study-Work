Tengein以及xQuic安装过程及遇到的问题

**安装教程：**
[https://github.com/TheDarkArchmageShangYang/networkLearning/blob/main/工具的安装及使用/xquic服务器使用.md](https://github.com/alibaba/tengine/blob/master/modules/ngx_http_xquic_module/README.md)
注意，xquic版本最好按照上面的教程，如果选择最新的xquic会出现接口不一致的问题。
注意，xquic官方提供了两种ssl库，最好选择Tongsuo，选择boringssl也可能会出现接口不一致的问题，此外boringssl本身并不提倡别的应用使用它作为第三方库。

另外，注意上述文档中的这句话
'/usr/local/tengine/sbin/tengine -p /usr/local/tengine/ -c conf/nginx.conf'
'''
/usr/local/tengine/sbin/tengine：这是 Tengine 服务器的可执行文件路径。表明要执行位于 /usr/local/tengine/sbin/ 目录下的 tengine 程序。
-p /usr/local/tengine/：这个参数指定了 Tengine 的运行目录（prefix）为 /usr/local/tengine/。运行目录通常用于存放一些与服务器运行相关的文件，如日志文件、临时文件等。
-c conf/nginx.conf：参数 -c 用于指定 Tengine 的配置文件。这里指定的配置文件为 /usr/local/tengine/conf/nginx.conf。配置文件中包含了服务器的各种设置，如监听的端口、虚拟主机的配置、反向代理规则等，决定了服务器如何处理客户端的请求
'''
注意，'conf/nginx.conf'这个文件的位置是在'/usr/local/tengine/'，而不是你下载了tengine的那个目录。所以修改配置时要修改'/usr/local/tengine/conf/nginx.conf'这个文件。

tengine中.config配置时，最好用绝对路径
'''
./configure \
  --prefix=/usr/local/tengine \
  --sbin-path=sbin/tengine \
  --with-xquic-inc="/home/guopeisheng/workbench/QuicBBR/xquic-1.6.0/include" \
  --with-xquic-lib="/home/guopeisheng/workbench/QuicBBR/xquic-1.6.0/build" \
  --with-http_v2_module \
  --without-http_rewrite_module \
  --add-module=modules/ngx_http_xquic_module \
  --with-openssl="/home/guopeisheng/workbench/QuicBBR/Tongsuo-8.3.2" \
  --with-http_mp4_module(ps:传输mp4视频才需要这个)
'''

**视频传输**
参考https://nginx.org/en/docs/http/ngx_http_mp4_module.html
在生成tengine的配置时，需要增加'--with-http_mp4_module'
注意，视频文件要也要放在/usr/local/tengine/目录下，或者就指定一个绝对路径。

**nginx.conf**
'''
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
    xquic_ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.pem;
    xquic_ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.pem;
    xquic_congestion_control bbr;
    xquic_socket_rcvbuf 5242880;
    xquic_socket_sndbuf 5242880;
    xquic_anti_amplification_limit 5;
    xquic_log_level debug;
    server {
        listen 10.37.30.62:80;
        listen 10.37.30.62:8000 ssl http2;
        listen 10.37.30.62:8000 xquic reuseport;
        server_name guopeisheng;
        
        add_header Alt-Svc 'h3=":8000"; ma=2592000,h3-29=":8000"; ma=2592000;' always;
        
        ssl_protocols       TLSv1.3;
        ssl_certificate        /home/guopeisheng/workbench/QuicBBR/cert/cacert.pem;
        ssl_certificate_key    /home/guopeisheng/workbench/QuicBBR/cert/privkey.pem;

        location / {
            root   html;
            index  index.html index.htm;

            # autoindex on;
            # autoindex_exact_size   on;
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
}
'''


增加nginx-rtmp-module模块
nginx-rtmp-module 网址：https://github.com/arut/nginx-rtmp-module
在编译tengine时，    './configure --add-module=/path/nginx-rtmp-module' 我的建议是把nginx-rtmp-module放在tengine/modelues中，因为我写成绝对路径编译也会显示找不到



**剩余未解决问题**
我的视频文件位置是/usr/local/tengine/video/，输入的播放地址是https://ipipip:8000/video/bbb_360p_30fps.mp4
如果不增加'alias /usr/local/tengine/video/;'就无法正确播放视频，因为他会默认从/usr/local/tengine/html/video/里面找
令我困惑的是，这个location不是最长匹配原则吗？为什么会先匹配到第一个location里面。 或者它不是最长匹配原则？这个我得再研究研究

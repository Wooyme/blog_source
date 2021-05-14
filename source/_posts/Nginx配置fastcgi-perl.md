---
title: Nginx配置fastcgi-perl
date: 2018-09-08 14:56:59
tags:
    - Nginx
    - perl
    - 运维
---

## **前言**
廉价VPS的小内存实在是供不起JVM，所以只好找别的出路了。php呢不是很喜欢，python又不是很想用，于是选了个硬核一点的Perl作为后端语言。反正前后分离，没了渲染压力后端想怎么玩怎么玩。

## **正文**
```Shell
apt-get install nginx libfcgi-perl wget
```
改nginx配置
```
server {
  listen   80;
  root   /var/www/example.com;

  location / {
      index  index.html index.htm index.pl;
  }  

  location ~ \.pl|cgi$ {
      try_files $uri =404;
      gzip off;
      fastcgi_pass  127.0.0.1:8999;
      fastcgi_index index.pl;
      fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
      include fastcgi_params;
      }
}
```
创建目录,给权限
```Shell
mkdir /var/www/example.com
chown -R www-data:www-data /var/www/example.com
```
下载配置FastCGI
```Shell
wget http://nginxlibrary.com/downloads/perl-fcgi/fastcgi-wrapper -O /usr/bin/fastcgi-wrapper.pl
wget http://nginxlibrary.com/downloads/perl-fcgi/perl-fcgi -O /etc/init.d/perl-fcgi
chmod +x /usr/bin/fastcgi-wrapper.pl
chmod +x /etc/init.d/perl-fcgi
update-rc.d perl-fcgi defaults
/usr/lib/insserv/insserv perl-fcgi
```
启动!
```Shell
service nginx start
service perl-fcgi start
```

## **坑**
顺利的话，就启动了。不顺利的话，启动perl-fastcgi的时候会失败，报错说`this account is currently not available`。Google了一圈好像没人在配置cgi的时候遇到这样的问题，比较多的是有些脑洞大开的人想从ssh登录到www-data这个用户上。  
顺便说一下www-data这个用户，如果看Nginx的进程的话
```Shell
ps -ef | grep nginx
```
会看到nginx的几个worker-process所属的用户都是www-data，这个就是专门给web应用提供的用户。  
看了一下这些登录不了的解决方案，发现cgi的问题应该也可以用同样的方法解决。

在`/etc/passwd`里，我们可以找到www-data的那行，然后就会发现它的结尾是`/usr/sbin/nologin`，我们要改成`/bin/bash`。这样这个用户就有运行perl脚本的能力了。重新启动一下服务，就OK了。

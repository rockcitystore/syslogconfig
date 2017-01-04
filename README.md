[toc]

##生产环境
打印info级
更换IP地址





##教程
[https://access.redhat.com/solutions/54363](https://access.redhat.com/solutions/54363)
[http://blog.chinaunix.net/uid-17240700-id-2813887.html](http://blog.chinaunix.net/uid-17240700-id-2813887.html)
[https://github.com/phuesler/ain](https://github.com/phuesler/ain)
发送日志服务器的rsyslog.conf之所以要开启udp支持主要因为发送服务器的syslog需要udp到本地才能接受到日志并转发到接受日志服务器

如果直接使用如下配置，
```
//utils/logger.js
, {
            type: 'log4js-ain2',
            tag: 'yingyan.servers',
            facility: 'user',
            hostname:'10.37.86.93'//接收服务器ip
        }
```
接受日志服务器的日志不能区分日志来源,全部显示为接收服务器ip， **但是可以通过把ip加入tag变通解决**

```
Jan  1 18:49:24 10.37.86.93 yingyan.servers[5118]: app.js - yingyan_v2_server listen at 130 in PRE
Jan  1 18:49:24 10.37.86.93 yingyan.servers[5118]: app.js - load action :./actions/specAnalysis.js
```




或者按照以下配置：





##node配置
```
//utils/logger.js
var log4js = require('log4js');
var config =require('../config');
log4js.configure({
    appenders: [
        {
            type: 'console'
        }
        , {
            type: 'log4js-ain2',
            tag: 'yingyan.servers',
            facility: 'user'
           ,hostname:require('os').networkInterfaces().eth0[0].address
        }
    ],
    levels: {
        "[all]": "ALL"
    }
    ,
    replaceConsole: true
});

module.exports = log4js;
```

##接受日志服务器
####Configure server using UDP.
编辑`/etc/rsyslog.conf `，新增/取消注释以下
```
//取消注释
# Provides UDP syslog reception  
$ModLoad imudp
$UDPServerRun 514

#### RULES ####
//新增user.none 
# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;authpriv.none;cron.none;user.none                /var/log/messages

//新增
#模板
$template verbose,"%timegenerated%,%syslogseverity-text%,%HOSTNAME%,%syslogtag%%msg:::json%\n"
#yingyan  user在log4js的facility配置 记录到info级
user.*              /opt/log/yingyan.servers.log;verbose

# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
#*.* @@remote-host:514
//新增
 *.* @@10.37.86.91:514
 *.* @@10.37.86.92:514
```

####重启服务
`service rsyslog restart`


##发送日志服务器
####Configure server using UDP.
编辑`/etc/rsyslog.conf `，新增/取消注释以下
```
//取消注释
# Provides UDP syslog reception  
$ModLoad imudp
$UDPServerRun 514

#### RULES ####
//新增user.none 
# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;authpriv.none;cron.none;user.none                /var/log/messages

//新增
#yingyan  user在log4js的facility配置 记录到info级
user.info              @10.37.86.93
# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
#*.* @@remote-host:514
//新增
 *.* @@127.0.0.1:514
```

####重启服务
`service rsyslog restart`




##颜色输出
```
tail -f /opt/log/yingyan.servers.log |
sed -e 's/\(.*info,.*\)/\o033[32m\1\o033[39m/' \
    -e 's/\(.*err,.*\)/\o033[31m\1\o033[39m/'
```

##配置logrotate

教程：[http://xstarcd.github.io/wiki/Linux/rsyslog_logrotate.html](http://xstarcd.github.io/wiki/Linux/rsyslog_logrotate.html)
往`/etc/logrotate.d/`添加配置文件`nodelog`,实现按日期保存日志
```
#全局配置

#指定文件配置
/opt/log/yingyan.servers.log {
	rotate 60
	daily
	notifempty
	missingok
	size 50M
	dateext
        postrotate
	/bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
        endscript
}

```

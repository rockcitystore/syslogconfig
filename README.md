[toc]

##生产环境
打印info级
更换IP地址





##教程
[https://access.redhat.com/solutions/54363](https://access.redhat.com/solutions/54363)
[http://blog.chinaunix.net/uid-17240700-id-2813887.html](http://blog.chinaunix.net/uid-17240700-id-2813887.html)
[https://github.com/phuesler/ain](https://github.com/phuesler/ain)
发送日志服务器的rsyslog.conf之所以要开启udp支持主要因为发送服务器的syslog需要udp到本地才能接受到日志并转发到接受日志服务器

##node配置
```
//utils/logger.js
var log4js = require('log4js');
var config =require('../config');
log4js.configure({
    appenders: [
        {
            type: 'console'
            , layout: {
            type: "pattern",
            pattern: "%d %[%-5p%] %c %m"
        }
        }
        , {
            type: 'log4js-ain2',
            tag: 'yingyan.servers',
            facility: 'user'
            //兼容liunx和macos
           ,hostname: (require('os').networkInterfaces().eth0 && require('os').networkInterfaces().eth0[0].address) || (require('os').networkInterfaces().en0 && require('os').networkInterfaces().en0[1].address) || 'none IP address'
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
user.*              /opt/logs/yingyan.servers.log;verbose

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
#yingyan  user是／utils/logger里的facility配置 记录到info级
user.info              @10.37.86.93

```

####重启服务
`service rsyslog restart`




##颜色输出
```
tail -f /opt/logs/yingyan.servers.log |
sed -e 's/\(.*info,.*\)/\o033[32m\1\o033[39m/' \
    -e 's/\(.*err,.*\)/\o033[31m\1\o033[39m/'
```

##配置logrotate

教程：[http://xstarcd.github.io/wiki/Linux/rsyslog_logrotate.html](http://xstarcd.github.io/wiki/Linux/rsyslog_logrotate.html)
往`/etc/logrotate.d/`添加配置文件`nodelog`,实现按日期/小时/周/月保存日志
```
#全局配置

#指定文件配置
/opt/logs/yingyan.servers.log {
	rotate 1440# 保留两个月 
	hourly #按小时
	notifempty#日志文件为空不进行转储
	missingok#如果日志文件不存在，不报错
	size 50M#超过50MB后轮转日志
	dateext #	增加日期作为后缀，不然会是一串无意义的数字
        postrotate
	/bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
        endscript
}

```


####开启按小时轮转
>There is `/etc/cron.daily/logrotate` script for daily logrotates. However there is no such script by default in `/etc/cron.hourly/` directory. Copy this script and it should work fine.

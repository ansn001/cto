# 简介
最小（少）原则，是安全的重要原则。最小的权限，最小的用户，最少的服务，最少的进程，是最安全的。

系统安全包括：文件系统保护、用户管理安全、进程的保护以及日志的管理。
# 场景
1. 确保服务最少，每个都是有用，而且权限最小化。
2. 确保用户最少，每个都是有用，而且权限最小化。
3. 确保文件权限最小。
4. 及时更新补丁，解决漏洞。
5. 规范好人为的因素。往往这个才是最大的隐患。

# 解决方案
## 最少服务
服务越少，漏洞越少，越不容易被攻击，越安全。服务器本身越封闭越安全。
### 最小安装。
绝不安装多余的软件，需要什么安装什么。在安装系统的时候就使用`最小安装`。不要图形界面，不要其他服务。
### 取消不必要的服务
即使做了最小安装，还是有很多可能用不到的服务，建议也是关闭，除非真的有用。
```
# 查看哪些服务在运行
/sbin/chkconfig --list |grep 3:on
# 没有使用的服务都可以考虑删除。
chkconfig ip6tables off # ipv6
chkconfig auditd off #用户空间监控程序
chkconfig autofs off #光盘软盘硬盘等自动加载服务
...
```
### 禁止外来ping操作
```
[root@tp /]# vi /etc/rc.d/rc.local
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all
```
### 限制控制台的登录
特别是关机和重启的命令，太危险了。
```
rm -rf /etc/security/console.apps/
```
```
[root@tp /]# vi /etc/securetty
...
#我们注释掉
tty1
# tty2
# tty3
# tty4
# tty5
# tty6
#只留下tty1,这时，root仅可在tty1终端登录
```

### 删除历史记录
防止账号被攻破后丢失更多的信息。
```
[root@tp /]# vim ~/.bash_logout
# 在里面添加命令
rm -rf  ~/.bash_history
```

## 最小用户
使用的用户权限越小越安全。特别是有些软件的漏洞可以直接获取账号执行权限。一旦使用root启动，就相当于服务器的root直接被破解。
！千万不要用root启动软件。
### 自动注销
当我们登录到Linux服务器上操作完以后，应该退出当前用户，否则可能会出现安全问题，特别是root用户，一旦被盗用很可能造成不可挽回的损失
```
[root@tp /]# vim /etc/profile
# 在里面添加
export TMOUT=300
```
### 设置口令复杂度
(授权完修改密码会有影响吗？？？这个需要测试)
定期修改密码
```
# 一个是在/etc/login.defs文件，里面几个选项
PASS_MAX_DAYS 90 #密码最长过期天数
PASS_MIN_DAYS 80 #密码最小过期天数
PASS_MIN_LEN 10 #密码最小长度
PASS_WARN_AGE 7 #密码过期警告天数
```
### 清理没有用的账号
在想是不是注释掉，还是直接删除
```
# 需要删除的用户包括：
userdel lp
userdel sync
userdel shutdown
userdel halt
userdel news
userdel operator
userdel games
userdel ftp
userdel rpc
userdel rpcuser
userdel gopher
userdel nscd
# 需要删除的组包括：
groupdel lp
groupdel news
```
### 使用sudo来使用root权限
```
[root@tp /]#  /etc/sudoers
# 在 root ALL=(ALL) ALL 下面添加一行
username  ALL=(ALL)   ALL
# 如果不想每次都输入密码可以用这一行
username ALL=(ALL) NOPASSWD:ALL 
exit 
```
## 最小文件权限
原则：原则上不给任何权限，只有需要的时候才添加权限。能不给写和执行的权限，坚决不能给！！拒绝777的行为。
赋权限的类型：
1. 重要的系统目录不可以修改
2. 产品代码只可以读，不可以执行，不可以修改
3. 需要上传目录，否则特别需要文件读写的目录要单独规划好。
4. 通过umask设置默认生成的文件和文件目录的最小权限。

## 更新补丁
建议做法：重装系统,update，然后测试业务是否正常。不建议写成定时去更新，容易引发软件的冲突，导致业务不可用。
如果是线上的业务，可以通过集群和配置管理的方式，把部分服务器更新。但是要做好计划，不能盲目更新。
## 人为的因素
人才是系统安全最大的隐患。

1. 每个人一个账号。
2. 每个角色一个组（比如：运维，开发）。这个需要进一步思考和细化。
3. 不允许使用root，如果有需要使用sudo。（能不能粒度到组啊？）
4. 把日常的运维操作，做成命令或者别名，减少人为操作。

# 验证方法
1. 文件是否被人篡改过 。`Tripwire`
2. 密码是否安全，是否容易被破解。`John the Ripper`。当然原则上通过防火墙来隔离更好，不允许其他网段ssh。
3. 系统安全。`Lynis`是针对Unix/Linux的安全检查工具，可以发现潜在的安全威胁。这个工具覆盖可疑文件监测、漏洞、恶意程序扫描、配置错误等。
4. 其他的场景，根据能不能操作来验证。
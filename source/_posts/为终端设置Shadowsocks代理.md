---
title: 为终端设置Shadowsocks代理
tags: linux
abbrlink: 937317d6
date: 2016-11-07 22:19:41
---

Shadowsocks是我们常用的代理工具，它使用socks5协议，而终端很多工具目前只支持http和https等协议，对socks5协议支持不够好，所以我们为终端设置shadowsocks的思路就是将socks协议转换成http协议，然后为终端设置即可。仔细想想也算是适配器模式的一种现实应用吧。

想要进行转换，需要借助工具，这里我们采用比较知名的polipo来实现。polipo是一个轻量级的缓存web代理程序。闲话休叙，让我们开始动手吧。


### 准备工作

- 首先需要配置好一个可用的shadowsocks，客户端下载地址：https://github.com/shadowsocks/shadowsocks/releases
- 安装和配置ss，请自行搜索解决。

<!-- more -->
### Linux安装使用

Ubuntu安装

```bash
sudo apt-get install polipo
```

如下打开配置文件

```bash
sudo vim /etc/polipo/config
```

设置ParentProxy为Shadowsocks，通常情况下本机shadowsocks的地址如下

```bash
# Uncomment this if you want to use a parent SOCKS proxy:
socksParentProxy = "localhost:1080"
socksProxyType = socks5
```

设置日志输出文件
```bash
logFile=/var/log/polipo
logLevel=4
```


先关闭正在运行的polipo，然后再次启动

```bash
sudo service polipo stop
sudo service polipo start
```


### Mac安装使用
Mac下使用Homebrew安装

```bash
brew install polipo
```

设置每次登陆启动polipo

```bash
ln -sfv /usr/local/opt/polipo/*.plist ~/Library/LaunchAgents
```
修改文件`/usr/local/opt/polipo/homebrew.mxcl.polipo.plist`设置parentProxy

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>homebrew.mxcl.polipo</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/opt/polipo/bin/polipo</string>
        <string>socksParentProxy=localhost:1080</string>
    </array>
    <!-- Set `ulimit -n 20480`. The default OS X limit is 256, that's
         not enough for Polipo (displays 'too many files open' errors).
         It seems like you have no reason to lower this limit
         (and unlikely will want to raise it). -->
    <key>SoftResourceLimits</key>
    <dict>
      <key>NumberOfFiles</key>
      <integer>20480</integer>
    </dict>
  </dict>
</plist>
```
修改的地方是增加了`<string>socksParentProxy=localhost:1080</string>`



启动使用. 注意：请确保Shadowsocks正常工作。

```bash
launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.polipo.plist
```


### 验证是否代理成功

安装完成就需要进行验证是否work。这里展示一个最简单的验证方法，打开终端，如下执行

```bash
$ curl ip.gs
当前 IP：125.39.112.15 来自：中国天津天津 联通
$ http_proxy=http://localhost:8123 curl ip.gs
当前 IP：210.140.193.128 来自：日本日本
```
如上所示，为某个命令设置代理，前面加上http_proxy=http://localhost:8123 后接命令即可。

>注：8123是polipo的默认端口，如有需要，可以修改成其他有效端口。



### 设置别名

bash中有一个很好的东西，就是别名alias. Linux用户修改~/.bashrc，Mac用户修改~/.bash_profile文件，增加如下设置

```bash
alias hp="http_proxy=http://localhost:8123"
```

然后Linux用户执行`source ~/.bashrc`，Mac用户执行`source ~/.bash_profile`

测试使用

```bash
$ curl ip.gs
当前 IP：125.39.112.14 来自：中国天津天津 联通
$ hp curl ip.gs
当前 IP：210.140.193.128 来自：日本日本 
```

### 全局设置, 删除当前会话

如果嫌每次为每一个命令设置代理比较麻烦，可以为当前会话设置全局的代理。使用`export http_proxy=http://localhost:8123`即可。 如果想撤销当前会话的http_proxy代理，使用 `unset http_proxy` 即可。 示例效果如下

```bash
$ curl ip.gs
当前 IP：125.39.112.14 来自：中国天津天津 联通
$ export http_proxy=http://localhost:8123
$ curl ip.gs
当前 IP：210.140.193.128 来自：日本日本 
$ unset http_proxy
$ curl ip.gs
当前 IP：125.39.112.14 来自：中国天津天津 联通
```
如果想要更长久的设置代理，可以将export http_proxy=http://localhost:8123加入.bashrc或者.bash_profile文件

### 设置git代理


```bash
git clone https://android.googlesource.com/tools/repo --config http.proxy=localhost:8123
Cloning into 'repo'...
remote: Counting objects: 135, done
remote: Finding sources: 100% (135/135)
remote: Total 3483 (delta 1956), reused 3483 (delta 1956)
Receiving objects: 100% (3483/3483), 2.63 MiB | 492 KiB/s, done.
Resolving deltas: 100% (1956/1956), done.
```
其实这样还是比较复杂，因为需要记忆的东西比较多，下面是一个更简单的实现

首先，在.bashrc或者.bash_profile文件加入这一句。
```bash
gp=" --config http.proxy=localhost:8123"
```
然后执行source操作，更新当前bash配置。

更简单的使用git的方法

```bash
git clone  https://android.googlesource.com/tools/repo $gp
Cloning into 'repo'...
remote: Counting objects: 135, done
remote: Finding sources: 100% (135/135)
remote: Total 3483 (delta 1956), reused 3483 (delta 1956)
Receiving objects: 100% (3483/3483), 2.63 MiB | 483 KiB/s, done.
Resolving deltas: 100% (1956/1956), done.
```



### git全局设置代理,删除代理

```bash
git config --global http.proxy 'localhost:8123'
git config --global https.proxy 'localhost:8123'

git config --global --unset http.proxy
git config --global --unset https.proxy
```
---
draft: true
category: Spring Boot
tags:
  - Spring Boot
  - deployment
date: 2020-05-27
title: Spring Boot 作为 Linux Systemd 服务部署
vssue-title: Deploying Spring Boot Applications As Systemd
---

systemd 是一个 Linux 系统基础组件的集合，提供了一个系统和服务管理器，运行为 PID 1 并负责启动其它程序。

可执行的jar可以其他 Unix 系统程序一样运行，也可以注册到init.d或systemd。这使我们可以很方便的在生成应用环境中安装和管理SpringBoot应用程序。

## plugin 配置

在 Gradle 中添加以下plugin配置可以创建一个`完全可执行`jar

```groovy
bootJar {
    launchScript()
}
```

Maven 等价配置如下
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
</plugin>
```
## 准备

### 目录

假设项目部署在 `/srv/spring` 目录下

```
ubuntu@instance-20191110-1830:/srv/spring$ ll
total 18040
drwxr-xr-x 3 spring spring     4096 Jun 17 01:34 ./
drwxr-xr-x 8 root   root       4096 Jun 16 01:38 ../
-rw------- 1 spring spring      113 Jun 17 01:24 spring.conf
-r-x------ 1 spring spring 18455089 Jun 17 01:32 spring.jar*
```
### 安全设置

文件权限直接关系到服务器安全与否。通常情况下，我们应当遵循「最小权限原则」，即权限越小越好。

```bash
# 使用独立用户运行应用
sudo useradd -M -s /sbin/nologin spring
sudo chown -R spring:spring /srv/spring
sudo chmod 400 spring.conf
sudo chmod 500 spring.jar
```

### 配置文件

如果需要在运行时限制内存的使用率或更改日志文件的目录可以通过创建配置文件的方式来实现。具体的操作为创建一个和 jar 同名的后缀为 `.conf` 文件，应用在运行的时候会自动检测当前 jar 目录下存不存在配置文件，存在就会自动加载配置信息。
例如我们的 jar 包名称为 `spring.jar` 则对应的配置文件为 `spring.conf`

具体的配置项可以查看 [Customizing a Script When It Runs
](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html#deployment-script-customization-when-it-runs)

本例中使用的配置文件如下

```
ubuntu@instance-20191110-1830:/srv/spring$ sudo cat spring.conf
JAVA_OPTS='-Xmx1024M'
RUN_ARGS='--server.port=9191'
```

## 编写单元文件
单元文件可以从多个地方加载，`systemctl show --property=UnitPath` 可以按优先级从低到高显示加载目录：
* `/usr/lib/systemd/system/` ：软件包安装的单元
* `/etc/systemd/system/` ：系统管理员安装的单元

### 单元文件

```bash
ubuntu@instance-20191110-1830:/srv/spring$ sudo cat /etc/systemd/system/spring.service
[Unit]
Description=spring
After=syslog.target

[Service]
User=spring
ExecStart=/srv/spring/spring.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```
单元文件的语法可以参考 [systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
##


## 参考
* [Systemd](https://wiki.archlinux.org/index.php/Systemd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

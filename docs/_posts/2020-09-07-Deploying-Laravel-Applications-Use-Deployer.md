---
category: PHP
tags:
  - php
  - laravel
  - deployment
date: 2020-09-07
title: 使用 Deployer 部署 Laravel 应用
vssue-title: Deploying Laravel Applications Use Deployer
---

Deployer 是一个基于 SSH 协议的无侵入 web 项目部署工具，因为它不需要你在目标服务器上装什么服务之类的东西即可使用，它只需要在你的开发机，或者你的笔记本，就是发起部署动作的一方安装即可。

它的原理就是通过 SSH 到你的机器去创建目录，移动文件，执行指定的动作来完成项目的部署。

Deployer 到部署步骤依赖于定义的 `deploy.php` 文件，我们可以将我们部署项目的步骤在其中定义，同时 Deployer 也内置了大部分 web 项目的部署步骤，如 Laravel，Symfony，我们只需稍加修改就可以完成部署。

主要分为以下几个部分：

- 在本地使用 composer 安装 deployer
- 在 Linux 服务器添加账户与配置权限
- 项目 git 仓库允许服务器访问（clone 代码）
- 部署我们的 web 项目

我们分开一个个讲，在后面的每一步请注意区分当前逻辑所在的环境：**本地 / 目标服务器**。

## Deployer 的安装配置

Deployer 的安装方式主要有两种，全局安装及项目安装。

> 此部分在本地操作

### 全局安装

以下两种方式任选其一

#### phar 包
```bash
curl -LO https://deployer.org/deployer.phar
mv deployer.phar /usr/local/bin/dep
chmod +x /usr/local/bin/dep
dep --version 
```

#### composer 安装

```bash
composer global require deployer/deployer
# PHP composer bin
export PATH=~/.composer/vendor/bin:$PATH
dep --version
```

### 项目中安装

本文将采用此种方式，进入需要部署的项目目录执行

 ```bash
composer require deployer/deployer --dev
vendor/bin/dep --version
```

## 服务器端的配置

> 此部分在服务器操作

### 用户配置

出于安全考虑我们使用独立用户进行项目部署

```bash
# 添加一个用户名为 deployer 的用户
sudo adduser deployer
# 将 deployer 添加值 www 组中
# web 项目通常需要一些上传，或者缓存写入这样的操作，所以 deployer 还需要有权限对目录进行修改，我们这里使用的 nginx 及 php-fpm 所在的用户组
usermod -aG www deployer
# 切换到 deployer 用户
su - deployer
# 设置 deployer 创建时默认的权限掩码，使其它用户及组只具有读的权限
echo "umask 022" >> ~/.bashrc
exit
```
将 `depoloyer` 用户加到 sudoers 中：

```bash
sudo visudo -f /etc/sudoers.d/deployer
# 添加如下内容
deployer ALL=(ALL) NOPASSWD: ALL
```

### 网站目录配置

> 由于我使用的 nginx 及 php-fpm 的用户和用户组都是 www ，所以 web 的根目录配置的用户和组都是 www，这里用户需要更具自己的实际情况进行调整，比如 ubuntu 默认的用户和组是 www-data

接下来要对我们的 web 根目录授权，假设我们的 web 服务的根目录在 `/data/wwwroot` 下，那么需要将这个目录的用户设置为 `deployer` ，组设置为 www 用户 `www`:

```bash
# 修改 web 根目录的用户和用户组
sudo chown deployer:www /data/wwwroot
# 使在 web 根目录下创建的文件继承 web 根目录的权限设定
sudo chmod g+s /data/wwwroot
```

### 服务器免密码登录 deployer

> 此部分在本地操作

如果本地已经配置 ssh 密钥，直接执行如下命令即可

```bash
# XXX 为服务器 ip 或者服务器主机名
ssh-copy-id deployer@XXX.XXX.XXX.XXX
```

没有配置过 ssh 密钥，通过如下命令生成，并执行上述操作即可

```bash
# XXX 可以是邮箱地址或者是对该密钥的描述
ssh-keygen -t rsa -b 4096 -C "XXX"
```

## 项目 git 仓库允许服务器访问

> 此部分在服务器操作

```bash
su - deployer
# 创建 ssh 密钥
ssh-keygen -t rsa -b 4096 -C "deployer" 
# 显示公钥,默认存储在 ~/.ssh 目录下
cat ~/.ssh/id_rsa.pub
```

请完整的复制 cat 出来的结果，然后去代码库添加 SSH 公钥。
这里 以 GitLab 为例：
项目所在仓库 --> 设置 --> 仓库 --> Deploy Keys，按照提示填入所对应的参数即可。

完成后，可在服务器上执行 git clone 操作验证密钥是否生效。

## Deployer 的使用

> 此部分在本地操作

### dep init

进入到需要部署到项目下执行

```bash
# 初始化 deploy.php 文件
vendor/bin/dep init

```
它会让你选择项目类型，比如 Laravel，symfony 等，如果你都不是，选择 common 类型即可。

这一步操作将会在当前目录生成一个 `deploy.php` 文件，包含用于部署的配置和任务。

需要关心到配置有
```php
<?php
namespace Deployer;

require 'recipe/laravel.php';
require 'recipe/cachetool.php';

// 由于我使用到是源码编译的方式按照的 PHP，因此这里指定了 PHP 执行文件的地址，一般通过 apt 或 yum 安装的 PHP 不存在此类问题，可以不用配置
set('bin/php', '/usr/local/php/bin/php');

// 项目名称
set('application', 'deployer-sample');

// 项目所在到仓库地址
set('repository', 'git@e.coding.net:oolongd/sample/laravel.git');


// 主机地址
host('10.10.10.71')
    ->user('deployer')
    ->identityFile('~/.ssh/id_rsa')
    ->set('deploy_path', '/data/wwwroot/www.11.com')
    ->set('cachetool', '/dev/shm/php-cgi.sock'); 

// 在创建配置缓存任务执行完成后执行路由缓存创建操作
after('artisan:config:cache', 'artisan:route:cache');
// 任务部署失败后，重置 deployer 的部署状态
after('deploy:failed', 'deploy:unlock');
// 由于服务器启用了 OPcache 来对 PHP 代码进行加速，故这里需要在每次代码部署完成后重置 OPcache 
after('deploy:symlink', 'cachetool:clear:opcache');
after('rollback', 'cachetool:clear:opcache');
```
我们看到这边定义的任务并不多，其实因为我们引入了 `require 'recipe/laravel.php'` 文件，文件中定义了许多部署 Laravel 项目的配置及任务，感兴趣的读者可以自行研究下。这里不多加赘述。

这里使用到了 cachetool 这个工具在项目部署后重置 OPcache。 需要注意的是，在使用前还需要在项目中引入 Deployer 的扩展包。

```bash
composer require deployer/recipes --dev
```
关于 recipes 可以参考 [deployer/recipes](https://deployer.org/recipes.html)

正确填写完配置清单以后，我们就可以部署我们的项目了，确认你的代码已经提交到代码仓库，因为执行部署的时候并不是将当前代码部署到服务器，而是从代码库拉最新的版本。

### 执行部署

```bash
# 加上 -vvv 显示详细的部署过程
vendor/bin/dep deploy -vvv
```

由于每个人的服务器环境多少会有所差异，本文只能描述大致的一个方向，因此可能在部署过程中会有失败的情况，通常根据报错信息即可定位。

### 执行回退

```bash
vendor/bin/dep rollback
```

## 关于 Deployer 部署结构

```bash
drwxrwsr-x 5 deployer www 4096 Sep  7 10:39 ./
drwxr-sr-x 4 deployer www 4096 Sep  3 11:47 ../
lrwxrwxrwx 1 deployer www   11 Sep  7 10:39 current -> releases/12/
drwxrwsr-x 2 deployer www 4096 Sep  7 10:38 .dep/
drwxrwsr-x 7 deployer www 4096 Sep  7 10:39 releases/
drwxrwsr-x 3 deployer www 4096 Sep  3 11:47 shared/
```
- `.dep` - Deployer 部署时所需到文件，比如 当前项目是否正在部署，防止两个部署任务同时运行及一些版本相关信息
- `current` - 它是指向一个具体的版本的软链接，你的 nginx 配置中 root 应该指向它，比如 laravel 项目的话 `root` 就指向：`/data/wwwroot/demo-app/current/public`
- `releases` - 部署的历史版本文件夹，里面可能有很多个最近部署的版本，可以根据你的配置来设置保留多少个版本，建议 5 个。保留版本可以让我们在上线出问题时使用 `dep rollback` 快速回滚项目到上一个版本。
- `shared` - 共享文件夹，它的作用就是存储我们项目中版本间共享的文件，比如 Laravel 项目的 `.env` 文件，`storage` 目录，或者你项目的上传文件夹，它会以软链接的形式链接到当前版本中。

## Deployer 的实现原理 

其实通过文件到目录结构就可以对 Deployer 对运行原理了解一二，具体步骤如下：
1. 在 `releases` 文件中创建一个存储当前项目版本对文件夹，我们姑且称作为项目文件夹，文件名为数字，并按照部署的顺序递增。
2. 在项目文件夹中，执行代码 clone 操作，并安装项目依赖，此时等依赖安装在对应版本等项目文件夹中，不同版本依赖版本相对独立，互不干扰。
3. 同时创建 `.env` 文件和 `storage` 文件的软链接到当前项目文件夹中达到共享目的。
4. 然后执行额外到一些操作，比如配置缓存，数据同步等操作。
5. 上述流程走完后，将当前版本等项目文件夹软链接到 `current` 文件夹，此时 nginx 及 php-fpm 即可读到最新的代码文件
6. 回退相对部署而言更加简单，只需将上一版本的项目文件夹软链接到 `current`，即可实现快速回退到操作。


## 参考
* [又一篇 Deployer 的使用攻略](https://learnku.com/articles/13242/another-introduction-to-the-use-of-deployer)
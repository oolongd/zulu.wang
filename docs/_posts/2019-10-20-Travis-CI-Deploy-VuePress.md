---
category: DevOps
tags:
  - CI/CD
  - DevOps
  - Travis CI
  - OSS
  - VuePress
  - 阿里云
date: 2019-10-22
title: GitHub 使用 Travis CI 实现 VuePress 博客基于阿里云 OSS 的自动部署
vssue-title: Travis CI Deploy VuePress
---

# GitHub 使用 Travis CI 实现 VuePress 博客基于阿里云 OSS 的自动部署

> 对象存储服务（Object Storage Service，OSS）是一种海量、安全、低成本、高可靠的云存储服务，适合存放任意类型的文件。容量和处理能力弹性扩展，多种存储类型供选择，全面优化存储成本。

阿里云对象存储 OSS 支持绑定自定义域名和静态网站托管，您可以使用您的自有域名在 OSS 上托管您的网站。

## 一、准备

### 1.1 OSS
* 已完成自有域名的注册
* 已完成域名备案
* 已创建公共读的Bucket

### 1.2 Travis CI
* GitHub 账号

### 1.3 VuePress

## 二、流程

得益于 VuePress 博客是纯静态的。首先本地部署 VuePress，生成构建静态文件，接下来的大致步骤如下：

1. 创建阿里云 OSS Bucket，并将权限设为公有读，启用静态网站托管并设置`默认首页`和`默认404页`
2. 申请域名，实名认证，ICP 备案，申请 https 证书，并将域名绑定到 Bucket
3. 获取阿里云的 AccessKey ID 和 AccessKey Secret
4. 创建 GitHub 项目 
5. 添加 ```.travis.yml``` 文件到你项目根目录下，并在 [Travis CI](https://travis-ci.org/) 启用对应的 GitHub 项目
6. 配置 Travis CI，设置  ```.travis.yml``` 中所对应的环境变量，`ENDPOINT`、`ACCESS_KEY_ID`、 `ACCESS_KEY_SECRET`
7. Git 提交静态文件，并 Push 到 GitHub

## 三、部署

### 3.1 VuePress 
```bash

# 将 VuePress 作为一个本地依赖安装
yarn add -D vuepress 

# 新建一个 docs 文件夹
mkdir docs

# 新建一个 markdown 文件
echo '# Hello VuePress!' > docs/README.md

```
接着，在 `package.json` 里加一些脚本:
```json
{
  "scripts": {
    "docs:dev": "vuepress dev docs",
    "docs:build": "vuepress build docs"
  }
}
```
生成静态的 HTML 文件，运行：
```bash
yarn docs:build
```
默认情况下，文件将会被生成在 `.vuepress/dist`，当然，你也可以通过 `.vuepress/config.js` 中的 `dest` 字段来修改，

### 3.2 配置 Bucket
进入`控制台` - `对象储存 OSS` - `创建 Bucket`，填写名称，选择区域，储存类型选择标准储存，读写权限选择公共读：
![](https://public.zulu.wang/img/20191022165604.png)
左侧存储空间列表中，单击目标存储空间。 依次进入`基础设置` - `静态页面`，设置`默认首页`和`默认404页`
![](https://public.zulu.wang/img/20191022171505.png)
依次进入`域名管理` - `绑定用户域名`

> 中国大陆地区需将域名[备案](https://beian.aliyun.com/order/selfBaIndex.htm)后才可以绑定到Bucket上

![](https://public.zulu.wang/img/20191022172401.png)
具体步骤参考 [绑定自定义域名](https://help.aliyun.com/document_detail/31902.html)

### 3.3 配置 Travis CI
新建 GitHub 项目，并启用 Travis CI。Travis CI 相关操作可参考这篇博客[持续集成服务 Travis CI](https://zulu.wang/posts/2019/10/18/travis-ci.html)
GitHub 账号登录 [Travis CI](https://travis-ci.org/)，确认接受访问 GitHub 的权限。
登录之后，Travis CI 就会同步你 GitHub 账号的仓库。然后打开[账户页面](https://travis-ci.org/account/repositories)并给你想要构建的项目启用 Travis-CI。
![](https://public.zulu.wang/img/20191018135503.png)
设置项目构建时所需的环境变量，用于配置阿里云 OSS 工具 `ossutil64` 
进入`Travis-CI` - 选择对应项目 - `More options` - `Setting` - `Environment Variables`，
![](https://public.zulu.wang/img/20191022174759.png)
在 3.1 所建目录的根目录下添加 ```.travis.yml``` 文件，内容如下
```yaml
language: node_js

node_js:
  - lts/*

branches:
  only:
  - master

cache:
  yarn: true
  directories:
    - "node_modules"

install:
  - yarn install

script:
  - yarn docs:build

after_success:
  - curl -LO http://gosspublic.alicdn.com/ossutil/1.6.7/ossutil64
  - chmod +x ossutil64
  - ./ossutil64 config -e $ENDPOINT -i $ACCESS_KEY_ID -k $ACCESS_KEY_SECRET
  - ./ossutil64 cp -rf docs/.vuepress/dist oss://zulu-wang
```

具体作用为，通过 Travis CI 自动生成静态文件，然后通过阿里云提供的 OSS 工具，将生成完的静态文件上传到 OSS 中。
`oss://zulu-wang` 此处的 zulu-wang 需要更改为实际的 Bucket 名称
关于 ossutil 可以参考 [命令行工具 ossutil](https://help.aliyun.com/document_detail/50452.html)

### 3.4 GitHub

将项目 Push 到 GitHub 即可触发 Travis CI 的自动构建，访问自定义的域名查看是否部署成功

## 四、优化

### 4.1 给 OSS 启用 CDN 加速
您可以使用阿里云 CDN 改善网站性能。CDN 让您的网站文件（如html、图像和视频）可供全球各地的数据中心（即边缘节点）使用。当访问者从您的网站请求文件时，CDN 自动将请求重定向到最近边缘节点上的文件副本，因此下载速度要快于访问者从较远的数据中心请求内容。参考[绑定 CDN 加速域名](https://help.aliyun.com/document_detail/97687.html)


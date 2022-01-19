---
layout: post
title: Code Signature
date: 2020-05-20
Author: Xiaoye
tags: [杂项]
excerpt_separator: <!--more-->
toc: true
---

回顾 tech talk 简单总结

<!--more-->

消息和算法计算出**签名**，消息可以是文件，软件，etc.

"Hello Bob" + Encrypt(private key) = ABCDEFGH...

签名和消息放在一起就是一条被**签名的消息**

签名的消息 + Encrypt(public key) 能证明这条签名是真是假



数字证书 = public key + owner info + 证书颁发者信息 + 签名（由CA对信息签名）



Apple 会用 private key 对通过审核的 App 进行签名，签名的方式：对每个文件的 hash 进行签名



On-device debugging

* 开发者生成 CSR (certificate signing request) ，会生成一对 private key 和 public key
* 将 CSR 上传给 Apple  签名生成新的证书，和本地生成的 private key 配套使用



iOS app entitlements

包含 App 的一些信息并且可以被其他进程读取，会和 code 一起被签名



* code sign 保证 code，resources 和 entitlements 没被篡改
* code sign 在启动 App 之前将二进制文件加载到内存进行校验，校验不过杀掉进程无法启动 App
* code sign flag 用于 link 动态库，link 的动态库要和 App 是同一个人签的



Mobile provisioning profile

只有 Apple 能签，可以限制 app

* App 是谁签的
* 可以安全的证书，设备
* 允许的 entitlements
* 确认 App identifier。wildcard 表示可以给多个 app identifier 使用，com.microsoft.*，但是限制 entitlements
* 过期时间



Case 1: install an app

1. download app bundle
2. 校验 app bundle 和 provisioning profile
3. 验证 entitlements，你想用的 entitlements 是否是 Apple 允许的，是否是 provisioning profile 中允许的设备、开发者、是否过期...
4. 验证完后拷贝到目标位置安装
5. 生成沙盒目录
6. 恢复备份



Case 2: launch app

1. 验证 provisioning profile 是否是 Apple 签的
2. 创建应用进程
3. 校验 code sign，code、entitlements、info.plist...
4. 根据 entitlements 设置沙盒，keychain-access-group...
5. load、  link 动态库


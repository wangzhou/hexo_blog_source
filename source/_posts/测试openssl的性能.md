---
title: 测试openssl的性能
tags:
  - 加解密
  - openssl
description: >-
  文本介绍openssl性能测试的相关方法，包括openssl自带的speed测试，加硬件engine的测试。并且简单分析下openssl
  speed测试的代码。
abbrlink: c413143d
date: 2021-06-19 09:59:34
---
1. openssl基本命令
------------------

AES对称加解密：

openssl enc -aes-128-cbc -in data -out key_encrypt -K 12345678901234567890 -iv 12345678
openssl enc -aes-128-cbc -in key_encrypt -out key_decrypt -K 12345678901234567890 -iv 12345678 -d

公钥加密：
openssl rsautl -encrypt -in rsa_test -inkey test_pub.key -pubin -out rsa_test.en -engine uadk

公钥生成：
openssl rsa -in test.key -pubout -out test_pub.key -engine uadk

私钥生成:
openssl genrsa -out test.key -engine uadk 4096

私钥解密：
openssl rsautl -decrypt -in rsa_test.en -inkey test.key -out rsa_test.de -engine uadk

签名：
openssl rsautl -sign -in msg.txt -inkey test.key -out signed.txt -engine uadk

认证：
openssl rsautl -verify -in signed.txt -inkey test_pub.key -pubin -out verified.txt -engine uadk

哈希:
openssl md5/sha1/sha256/sm3 -engine uadk data
openssl md5/sha1/sha256/sm3 data

如上，有-engine xxx的表示用执行的硬件加解密engine做任务，没有指定就是用openssl
里提供的软件计算方法搞。

在非对称加解密的测试中，我们使用RSA算法。需要先生成私钥，然后生成公钥，然后用
秘钥进行加解密和签名、认证的测试。

2. openssl speed命令
--------------------

openssl speed aes

openssl speed rsa

openssl speed -engine uadk -async_jobs 1 -evp md5

openssl speed -engine uadk -async_jobs 1 -evp aes-128-cbc

openssl speed -engine uadk -elapsed rsa2048  // 如下例子

openssl speed -engine uadk -elapsed -async_jobs 1 rsa2048

openssl speed -engine uadk -elapsed -async_jobs 36 rsa2048

如上，加了-async_jobs使用了openssl里的异步机制，如果engine里使用过了openssl里的
异步机制，这里就会触发engine里的异步机制生效。

openssl speed的代码在openssl/apps/speed.c，拿同步rsa2048为例。
```
	main
	  /* 分配input，output buffer */
	  +-> app_malloc(buflen, "input buffer");
	  /* 只看同步，即async_jobs = 0的情况 */
	  +-> run_benchmark(async_jobs, RSA_sign_loop, loopargs);
	    /*
	     * 可以看到这里的内存使用模型是，反复的用一个buffer做sign。如果
	     * engine的实现没有另外申请内存，这个测试将反复用一块固定的buffer。
	     */
	    +-> RSA_sign_loop
	      +-> RSA_sign
	        /* 如果下面适配的是openssl engine, 可以实现如下的回调函数支持 */
	        +-> rsa->meth->rsa_sign
		+-> RSA_private_encrypt
		  +-> rsa->meth->rsa_priv_enc

```

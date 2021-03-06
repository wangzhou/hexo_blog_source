---
title: KAE笔记
tags:
  - openssl engine
description: KAE是华为KunPeng服务器上加速器模块对应的openssl engine实现。本文是KAE的学习笔记。
abbrlink: 996a9d74
date: 2021-06-20 23:19:27
---

梳理KAE实现Openssl engine的基本逻辑。本文基于的KAE代码在这个位置:
https://github.com/kunpengcompute/KAE

1. 注册为openssl engine的位置在: engine_kae.c
---------------------------------------------
```
IMPLEMENT_DYNAMIC_BIND_FN(bind_kae)                                             
	+-> kae_engine_setup() // 配置引擎的参数
		+-> cipher_module_init() // 初始化sec引擎
			初始化队列池，但是没有申请队列
			+-> wd_ciphers_init_qnode_pool()

			+-> sec_create_ciphers()

				+—> sec_ciphers_set_cipher_method(cipher_info_t cipherinfo)      

					注册一个实例的回调函数。每个实例的创建、做任务、销毁分别是：init, do_cipher, cleanup。
					EVP_CIPHER *cipher = EVP_CIPHER_meth_new(cipherinfo.nid, cipherinfo.blocksize, cipherinfo.keylen);
					EVP_CIPHER_meth_set_iv_length(cipher, cipherinfo.ivlen);             
					EVP_CIPHER_meth_set_flags(cipher, cipherinfo.flags);                 
					EVP_CIPHER_meth_set_init(cipher, sec_ciphers_init);                  
					EVP_CIPHER_meth_set_do_cipher(cipher, sec_ciphers_do_cipher);        
					EVP_CIPHER_meth_set_set_asn1_params(cipher, EVP_CIPHER_set_asn1_iv); 
					EVP_CIPHER_meth_set_get_asn1_params(cipher, EVP_CIPHER_get_asn1_iv); 
					EVP_CIPHER_meth_set_cleanup(cipher, sec_ciphers_cleanup);            
					EVP_CIPHER_meth_set_impl_ctx_size(cipher, sizeof(cipher_priv_ctx_t));

			注册sec的回调处理函数
			+-> async_register_poll_fn(ASYNC_TASK_CIPHER, sec_cipher_engine_ctx_poll)
		...

		+-> async_module_init() //起异步线程, 整个引擎里只有一个异步的轮训线程
			+-> async_polling_thread_init()

	+-> ENGINE_set_ciphers(e, sec_engine_ciphers) // 向系统里设置sec引擎
```
所以, 一个openssl engine使用wd队列的模型是：每次上层用户请求创建一个实例，每个实例
里申请一个wd队列，可以反复向这一个队列上发送任务，发送任务的时候在这个队列上创建
一个ctx，反复发送任务时, 每次都复用这个队列和ctx。

发送异步任务和以上同步任务情况是一样的。不一样的是，每一个异步任务发送后，都把队列
的信息加入到一个自定义的软件队列里，然后向异步轮寻线程发送通知，异步轮寻线程接受
到通知后轮寻对应的硬件队列:

sec_ciphers_do_cipher(EVP_CIPHER_CTX *ctx, unsigned char *out, const unsigned char *in, size_t inl)
	+-> sec_ciphers_async_do_crypto(e_cipher_ctx, &op_done)
		+-> async_add_poll_task(e_cipher_ctx, op_done, type)

总结:
-----

1. KAE openssl并没有整理规划wd队列的使用，如果是一个新请求就创建一个队列
   的话，最大并发请求个数会是2048

2. KAE在支持存储的时候，spark，ceph都会出现大量同步并发请求的情况。
   目前需要把ctx->q, 多对1的情况支持起来。

3. 异步的时候，使用少量线程就可以跑满硬件性能。所以，异步并发请求不大。
   同步会有大的并发请求

4. 一个进程里的请求大概是相同大小的。

5. KAE一个engine一个异步轮寻队列，目前可以满足他的性能需求。

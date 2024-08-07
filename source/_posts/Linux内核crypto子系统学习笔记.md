---
title: Linux内核crypto子系统学习笔记
tags:
  - Linux内核
  - crypto
description: >-
  本文分析Linux kernel里crypto子系统的大概实现，写crypto子系统下的加速器驱动的时候
  可以参考下。crypto子系统支持加解密，压缩解压缩等功能。
abbrlink: d2df7e14
date: 2021-07-05 22:33:10
categories:
---

Analysis will start from crypto test cases in crypto/testmgr.c, e.g. deflate.
上面的路径上是内核里这对crypto子系统的一个测试程序。通过分析这个程序可以大概
看出crypto子系统向外提供的API. 整个系统的情况大概是这样的：

crypto API <---> crypto core <---> crypto_register_alg

设备驱动通过crypto_register_alg把一个设备支持的算法注册到crypto系统里。
注册的时候会通过struct crypto_alg把相关的信息传递给crypto core.

struct crypto_alg的结构是这样的：
```
struct crypto_alg {
	struct list_head cra_list;
	struct list_head cra_users;

	u32 cra_flags;
	unsigned int cra_blocksize;
	unsigned int cra_ctxsize;
	unsigned int cra_alignmask;

	int cra_priority;
	atomic_t cra_refcnt;

	char cra_name[CRYPTO_MAX_ALG_NAME];
	char cra_driver_name[CRYPTO_MAX_ALG_NAME];

	const struct crypto_type *cra_type;

	union {
		struct ablkcipher_alg ablkcipher;
		struct blkcipher_alg blkcipher;
		struct cipher_alg cipher;
		struct compress_alg compress;
	} cra_u;

	int (*cra_init)(struct crypto_tfm *tfm);
	void (*cra_exit)(struct crypto_tfm *tfm);
	void (*cra_destroy)(struct crypto_alg *alg);
	
	struct module *cra_module;
} CRYPTO_MINALIGN_ATTR;
```
这个结构的几个关键的信息是: cra_ctxsize, cra_u(下面以compress_alg说明), cra_init,
			    cra_exit.
这个结构表述的是算法相关的系统，但是在执行一个请求的时候，还有维护一组上下文的信息，
这些信息记录在结构体: struct crypto_tfm.
```
struct crypto_tfm {

	u32 crt_flags;
	
	union {
		struct ablkcipher_tfm ablkcipher;
		struct blkcipher_tfm blkcipher;
		struct cipher_tfm cipher;
		struct compress_tfm compress;
                    --> cot_compress
                    --> cot_decompress
	} crt_u;

	void (*exit)(struct crypto_tfm *tfm);
	
	struct crypto_alg *__crt_alg;

	void *__crt_ctx[] CRYPTO_MINALIGN_ATTR;
};
```
其中最后一个__crt_ctx是这个上下文的私有数据。上面的cra_ctxsize就是这个私有数据的
size. cra_init是准备上下文的函数，比如，你用一个硬件设备压缩数据，实际的物理操作
发生在这个硬件的一个队列上，那么就需要准备这个队列，准备必要的缓存等等。cra_exit
是退出上下文。cra_u里是具体执行算法的函数，比如可以压缩和解压缩的函数。

从设备驱动的角度讲, 设备驱动只是看到了crypto_alg这个结构。这个结构里的crypt_tfm
即一个操作执行的上下问是从哪里知道的呢？毕竟crypto_alg这个结构里的.cra_init,
.cra_exit, .cra_u里的.coa_compress和.coa_decompress都需要这个执行上下文。
我们在下面具体看一下。


知道这些内部的数据结构对我们理解外部的API有帮助。现在假设crypto的设备驱动已经有了，
那么，其他的内核模块怎么用呢？ 其实一开头我们已经讲到crypto/testmgr.c测试程序。

测试的代码里有异步的测试和同步的测试流程，我们这里先看同步的测试:

主要的逻辑就三个函数, 第一先需要分配一个压缩的上下文(本文用压缩的例子), 其实它
就是crypto_tfm的包装，和cryto_tfm是一样的:
```
struct crypto_comp {
	struct crypto_tfm base;
};
```
struct crypto_comp = crypto_alloc_comp(driver, type, mask), 这个过程中会调用到
cra_init函数，这个函数是设备驱动实现的，完成硬件相关的配置，上面已经提到过。
调用关系如下:
```
/* 分配一个压缩解压缩的上下文, 可以看到这里的压缩解压缩的上下文完全就是crypto_tfm */
struct crypto_comp = crypto_alloc_comp(driver, type, mask);
    --> crypto_alloc_base(alg_name, type, mask)
            /* find algrithm: use alg_name, driver name */
        --> alg = crypto_alg_mod_lookup(alg_name, type, mask);
            /* 上下文是依据具体的算法去分配的 */
        --> tfm = __crypto_alloc_tfm(alg, type, mask);
	        /* 上下文中指定相关的算法 */
            --> tfm->__crt_alg = alg;
            --> crypto_init_ops
	            /* 把相应的算法中的压缩解压缩函数传递给上下文 */
                --> crypto_init_compress_ops(tfm)
                        /* ops is struct compress_tfm */
	            --> ops->cot_compress = crypto_compress;
                            /* tfm->__crt_alg->cra_u.compress.coa_compress */ 
                            /*
                             * e.g. drivers/crypto/cavium/zip/zip_main.c
                             *      struct crypto_alg zip_comp_deflate.
                             * will finally call zip_comp_compress!
                             */
                        --> tfm->__crt_alg->cra_compress.coa_compress

	            --> ops->cot_decompress = crypto_decompress;
		/*
                 * 在创建上下文的最后调用下，算法里的初始化函数，如果是和一个硬件
		 * 的驱动适配，那么这里就可以执行相应硬件初始化的内容。
		 */
            --> if (!tfm->exit && alg->cra_init && (err = alg->cra_init(tfm)))
```

第二，就是执行压缩的操作:
crypto_comp_compress(tfm, input, ilen, result, &dlen)
```
crypto_comp_compress(crypto_comp, src, slen, dst, dlen)
        /* so hardware can do compress here! */
    --> compress_tfm->cot_compress;
```

第三，就是释放这个压缩的上下文
crypto_free_comp(comp)

内核虽然现在提供了压缩的异步接口，但是貌似还没有驱动会用到。异步接口的使用要比同步
接口复杂一点。下面具体看看。

```
In alg_test_comp, async branch:
/* 和同步一样，这里也创建一个异步的上下文 */
acomp = crypto_alloc_acomp(driver, type, mask);
/*
 * 不过和同步接口不一样的是，这里又创建一个acomp_req的上下文, 后续的操作都围绕
 * 着这个req结构展开。可以看到req里面包含了异步接口需要的回调函数。
 */
req = acomp_request_alloc(tfm);
acomp_request_set_params(req, &src, &dst, ilen, dlen);
acomp_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
			   crypto_req_done, &wait);
crypto_acomp_compress(req)
```
这里需要说明的是，testmsg.c里的这个acomp的测试程序里加了wait/complete的相关
内容。这里应该是为了测试方便而加的，一般的异步接口里, 当硬件完成操作的时候，在
中断函数里直接调用异步接口的回调函数就可以了。


---
title: How to use Linux kernel crypto compression
tags:
  - Linux内核
  - crypto
description: This doc shares the crypto compression API in Linux kernel.
abbrlink: 2efef7d4
date: 2021-06-28 23:58:56
categories:
---

Linux kernel crypto subsystem supports different algorithms including compression
algorithms. A hardware or software implementation can be added to crypto
subsystem to do specific work.

If you have a hardware module to do Zlib/Gzip compressions, you can write a
driver, which can be registered into Linux kernel crypto subsystem.

To use this hardware to do compression, you need not care about hardware detail,
only thing you need to know is crypto API, here, you should know crypto compress
related API.

Firstly, we should create a crypto compression context using:
We only consider synchronized API crypto_alloc_comp here.(struct crypto_comp)
```
	crypto_alloc_comp(const char *alg_name, u32 type, u32 mask)
	crypto_alloc_acomp(const char *alg_name, u32 type, u32 mask)
```
alg_name here can be specific driver name or standard algorithm name. You can
find driver name in cra_driver_name item in struct crypto_alg, which may be
offered in hardware engine driver's crypto_alg, we do not offer this; algorithm
name is offered in cra_name in struct crypto_alg, which also can be offered in
hardware engine driver, we do this, we offer "zlib-deflate" and "gzip".

Crypto subsystem tries to find registered algorithms by driver name or standard
algorithm name. Driver name has the top priority, standard algorithm name and
priority item in struct crypto_alg also can be used to find proper algorithm.
In user space, you can use "cat /proc/crypto" to find registered crypto
algorithms, e.g.:
```
name         : crct10dif
driver       : crct10dif-pclmul
module       : crct10dif_pclmul
priority     : 200
refcnt       : 1
selftest     : passed
internal     : no
type         : shash
blocksize    : 1
digestsize   : 2

name         : crc32
driver       : crc32-pclmul
module       : crc32_pclmul
priority     : 200
refcnt       : 1
selftest     : passed
internal     : no
type         : shash
blocksize    : 1
digestsize   : 4
...
```
u32 type and u32 mask here, we use below, which are defined in[1]
```
	#define CRYPTO_ALG_TYPE_ACOMPRESS	0x0000000a
	#define CRYPTO_ALG_TYPE_COMPRESS	0x00000002
	...

	#define CRYPTO_ALG_TYPE_MASK		0x0000000f
```
Here, we only support CRYPTO_ALG_TYPE_COMPRESS currently.

After creating compression context, we can use it to do compression/decompression by:
```
	crypto_comp_compress(struct crypto_comp *tfm, const u8 *src,
			     unsigned int slen, u8 *dst, unsigned int *dlen)
	crypto_comp_decompress() (currrently we do not support)
```
src, slen, dst, dlen are input/output buffer's address and size.

After compression/decompress finished, we call below function to free related
context:
```
	crypto_free_comp()
```

Currently, if you want to use HiSilicon's Compression engine in D06, which
driver is under upstreaming.[2] You should open below kernel configures:
```
CONFIG_CRYPTO_DEV_HISILICON
CONFIG_CRYPTO_DEV_HISI_QM
CONFIG_CRYPTO_DEV_HISI_ZIP
```

NOTE:

 [1] linux/include/linux/crypto.h
 [2] https://lkml.org/lkml/2018/9/3/6

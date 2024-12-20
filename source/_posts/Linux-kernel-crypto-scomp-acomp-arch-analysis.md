---
title: Linux kernel crypto scomp/acomp arch analysis
tags:
  - Linux内核
  - crypto
  - ZIP
description: >-
  This doc shares the arch of linux crypto scomp/acomp arch. This doc is based
  on mainline kernel v5.0-rc3
abbrlink: 6fa184
date: 2021-06-28 23:57:46
categories:
---

General introduce of linux kernel crypto compression API
-----------------------------------------------------------
```
 crypto_register_alg()
 crypto_alloc_comp()
 
 crypto_register_scomp() 
 crypto_alloc_acomp()

 crypto_register_acomp() 
 crypto_alloc_acomp()
```
As showed in above, we can only use crypto_register_alg() to register a general
crypto compress alg, and use crypto_alloc_comp() to get the crypto_comp to do
compression/decompression. We can not register a new compress alg by this old
general API in mainline kernel now and in future.

Crypto system also supports to register a sync compress alg using
crypto_register_scomp, and use crypto_register_acomp to register an async
compress alg. Now we use only crypto_alloc_acomp() to get crypto_scomp/crypto_acomp.
There is no interface to get crypto_scomp, but crypto_alloc_acomp().

For the detail usage of scomp/acomp, we can have an example in kenrel/crypto/testmgr.c
alg_test_comp().

Currently all compression/decompression algorithms are registered to kernel
crypto system by crypto_register_alg() or crypto_register_scomp(). And all users
in kernel use crypto_alloc_comp() to get a compression/decompression context.

In future, as mentioned in above, all compression/decompression alg should
register to crypto by scomp/acomp.

comp
-------
```
 crypto_alloc_comp()
 crypto_comp_compress()/crypto_comp_decompress()
 crypto_free_comp()
```
We can use above APIs to do compression/decompress. Crypto subsystem has done
little about them. So When we add a hardware/software implementation for these
APIs, there is little limitation about them.

Scomp
--------
We should use acomp interface to use scomp, they are as below:
```
crypto_alloc_acomp()
acomp_request_alloc()
acomp_request_set_params()
acomp_request_set_callback()
crypto_init_wait()
crypto_wait_req()
crypto_acomp_compress()/crypto_acomp_decompress()
acomp_request_free();
crypto_free_acomp()
```
For the view of user, acomp APIs offer scatterlists to store input/output data.

crypto scomp has internal buffers: when creating a scomp context, it allocates
a input buffer and a output buffer in every cpu core, each buffer is 128KB.

scomp arch first copies data in scatterlist into above internal input buffer,
then call compression/decompression inplementations to do real work. As one
cpu core use the its own input/output buffer, so before doing real compression/
decompression work, scomp arch will call get_cpu to disable scheduling, otherwise
another scomp job may run in the same cpu core, which will break the data in
input/output buffer. This means struct scomp_alg's compress/decompress can not
do scheduling in themselves.

Acomp
--------
We also use acomp interface to use acomp, they are as above.
As it is an async interface, the difference from scomp is user can offer a
callback to software/hardware implementation, which can call the callback after
compression/decompression is done.

The callback is set by acomp_request_set_callback(), and it will be passed to
compression/decompression process by struct acomp_req.

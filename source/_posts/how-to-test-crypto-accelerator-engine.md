---
title: how to test crypto accelerator engine
tags:
  - Linux内核
  - crypto
  - 软件测试
description: >-
  Linux内核crypto子系统带有自测试的功能，它可以对注册到crypto子系统上的各种算法
  做测试。如果你自己写了一个驱动注册了一个算法到crypto子系统上可以使用crypto自带
  的测试程序做一下快速的测试。本文大概介绍下crypto下的自测试相关的东西。
abbrlink: 65923ba6
date: 2021-07-05 22:26:58
categories:
---

简单的讲，就是在你写的crypto驱动注册到crypto子系统的时候，注册函数会调用crypto_chain
注册链表上的回调函数。

如果使能了crypto/algboss.c这个驱动，这个驱动初始化的时候会向crypto_chain里注册相关
的测试代码, 这些测试代码会启动一个内核线程去测试刚刚注册的crypto算法：
```
	cryptomgr_schedule_test
		--> kthread_run(cryptomgr_test, param, "cryptomgr_test")

	static int cryptomgr_test(void *data)
	{
		struct crypto_test_param *param = data;
		u32 type = param->type;
		int err = 0;
	
	#ifdef CONFIG_CRYPTO_MANAGER_DISABLE_TESTS
		goto skiptest;
	#endif
	
		if (type & CRYPTO_ALG_TESTED)
			goto skiptest;
	
		err = alg_test(param->driver, param->alg, type, CRYPTO_ALG_TESTED);
	
	skiptest:
		crypto_alg_tested(param->driver, err);
	
		kfree(param);
		module_put_and_exit(0);
	}
```
可以看到要想要执行到测试程序alg_test, 需要把CONFIG_CRYPTO_MANAGER_DISABLE_TESTS
设置成n.

alg_test(crypto/testmgr.c)会根据算法的名字调用事先准备好的测试代码, 可以搜索
struct alg_test_desc alg_test_descs[](crypto/testmgr.c)这个数组找到所有事先放好
的测试代码。

比如，我们可以找到deflate压缩解压缩相关的测试代码：
```
	...
		.alg = "deflate",
		.test = alg_test_comp,
		.fips_allowed = 1,
		.suite = {
			.comp = {
				.comp = __VECS(deflate_comp_tv_template),
				.decomp = __VECS(deflate_decomp_tv_template)
	...
```
可以看到，alg_test_comp函数会调用crypto子系统提供的API做相应算法的测试，比如，
这里会用crypto_alloc_comp, crypto_comp_compress, crypto_free_comp(comp)
等crypto API对新注册的支持deflate算法的驱动做测试(假设你写的crypto算法的驱动支持
deflate算法), 当然测试使用的压缩解压缩的数据就是上面deflate_comp_tv_template,
deflate_decomp_tv_template中的数据。

但是，这里对于压缩解压缩算法似乎存在一个问题, 那就硬件加速器压缩解压缩得到的
数据和软件压缩解压缩可能是不一样的。也就是说，你为硬件压缩解压缩写一个crypto的
驱动，注册在crypto系统上，然后上面crypto自带的测试程序测试，由于硬件压缩，软件
压缩出来的结果是不一样的，可能你的硬件压缩的结果是对的，但是上面的测试是失败的。

对于压缩解压这个问题，可以通过用软件把硬件压缩完的数据解压一下，然后对比是否和
原来的压缩前的数据一致。具体实现的话，可以直接用crypto API申请一个基于软件的压缩
解压算法进行验证。也可以用内核lib/zlib_deflate, lib/zlib_inflate库来进行软件的
压缩和解压缩。对于第一种验证方法，我们可以修改内核的alg_test_comp函数来实现，
这个修改可以考虑upstream到内核主线。对于第二种验证方法，我们可以在硬件的crypto
驱动中直接调用zlib_deflate/zlib_inflate相关函数来做，基本的使用方法是：
```
	struct z_stream_s stream;
	char decomp_result[COMP_BUF_SIZE];

	stream.workspace  = kmalloc(zlib_inflate_workspacesize(), GFP_KERNEL);

	ret = zlib_inflateInit2(&stream, MAX_WINDOW_BIT);

	stream.next_in = dst;
	stream.avail_in = dlen;
	stream.total_in = 0;
	stream.next_out = decomp_result;
	stream.avail_out = COMP_BUF_SIZE;
	stream.total_out = 0;

	ret = zlib_inflate(&stream, Z_FINISH);

	memcmp(decomp_result, src, slen);
```

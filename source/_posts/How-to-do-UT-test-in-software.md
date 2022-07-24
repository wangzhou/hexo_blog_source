---
title: How to do UT test in software
tags:
  - 软件测试
description: >-
  UT test is one kind of basic software test. This document tries to share the
  basic idea about UT test from author. The idea is not mutual and author does
  not have much experience about UT.
abbrlink: d4f2a16
date: 2021-07-05 22:26:09
categories:
---

The idea of UT is to test the logic of your codes match with your plan, it is
in function level.

Let's see an example function:
```
int tested_function(a, b)
{
	c = function(a, b);
	return c;
}
```
Above example is very simple. We can see this tested_function inside is combined
logic. Every time you set inputs, e.g. a, b here, you get a output c. So the UT
test of this function is to find proper input combinations to test if return
value is expected.

However, we have another kind of function which likes timed-based logic. Even
"input" is same, but it can return different value. The reason of this is
because there is a memory inside of this kind of function. See blow example:
```
int tested_function(a, b)
{
	void *c = system_api();
	function1(global_d);
	function2(global_e);
	...
}
```
Everytime you call system_api, its return value is different, as there are some
memory in this system_api. Everytime you use global_a, global_b, their values
are different, as the scope of these value is outside of this function, so from
the view of this tested_function, global_d/global_e is memory part.

So for this kind of time-based logic function. Its inputs are a, b, c, global_d,
global_e. So the UT test is to prepare proper inputs combinations and test if
return value are expected.

So we should control system_api to create expected c to test our logic in
tested_function. So we create a system_api() ourself to replace real system_api(),
it will be like below, which is called a stub function of system_api().
```
void *system_api()
{
	switch (control_system_api) {
	case CONTROL_1:
		return c_1;
	case CONTROL_2:
		return c_2;
	case CONTROL_3:
		return c_3;
	}
}
```

An ut test for a C file, all outside functions should have related stub functions.
Stub functions are the preparations for after ut test.

We should write ut test case to do ut test: first setting inputs, remember if
your tested function is a time-base logic function, you should set "input points"
for all inputs and memory inputs, e.g. a, b, c, global_d, global_e; second, run
tested function; third, check return value comparing with expected ones. A test
case will be like:
```
	void prepare_input()
	{
		control_system_api = 1;
		global_d = 123;
		global_e = 456;
	}

	void check_result()
	{
		assert(c == 789);
		log();
	}

	int test_case_1()
	{
		prepare_input();
		c = tested_function(a, b);
		check_result(c);
		clear_input()；
	}
```

---
layout:     post
title:      使用fuzzer实现anti-debugging的趣味实验 XD
subtitle:   Linux平台下使用fuzzer实现anti-debugging的实验
date:       2019-01-20
author:     Weizhou
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Binary
---

## 前言

在youtube和国外论坛上看到了一种比较有趣的使用fuzzing实现反调试的思路.

方法的原理很简单，通过循环修改源文件的一个byte使得该文件正常共能能够实现，但是能够对gdb和radare2这累调试软件进行一定程度上的干扰。通过实验之后发现，该方法不一定适合实战，但是可能会在对于反调试这一部分开拓新思路上面会有所帮助。

## 正文

首先使用C语言写一个类似于Crackme的简单小程序：

```C code
#include <stdio.h>
#include <string.h>

int main(int argc, char* argv[])
{
	if(argc==2)
	{
		if (strcmp("CORRECT_SERIAL", argv[1])==0)
		{
			printf("Accessed!\n");
		}
		else
		{
			printf("Denied!\n");
		}
	}
	else
	{
		printf("Usage %s [serial_number]\n", argv[0]);
	}
}
```

然后写一个python脚本自动实现上述的通过fuzzing进行反调试的实验

<strong>python源代码：</strong>

```python3
#/bin/python3!

import os
import random

def alter_byte(bytes_in):
	i = random.randint(0, len(bytes_in))
	c = chr(random.randint(0,255))
	return bytes_in[:i] + c.encode()+ bytes_in[i+1:]

def alter_file():
	with open("secret", 'rb') as f_obj, open("secret_tmp", 'wb') as new_f:
		f_content = f_obj.read()
		new_content = alter_byte(f_content)
		new_f.write(new_content)


def cmp_out_file(f_1, f_2):
	with open(f_1, 'rb') as f_1_obj, open(f_2, 'rb') as f_2_obj:
		return f_1_obj.read() == f_2_obj.read()

def check_normal():
	cmd = "(./secret_tmp; ./secret_tmp CORRECT_SERIAL; ./secret_tmp WRONG_SERIAL) > normal_tmp.txt"
	os.system(cmd)
	return cmp_out_file("std_output.txt", "normal_tmp.txt")

def check_gdb():
	os.system("echo disassemble main | gdb secret_tmp > gdb_tmp.txt")
	return cmp_out_file("gdb.txt", "gdb_tmp.txt")

def check_r2():
	os.system("echo 'aaa\rs main\rpdf' | radare2 secret_tmp > r2_tmp.txt")
	return cmp_out_file("r2_tmp.txt", "r2.txt")

if __name__ == '__main__':
	os.system("cp secret secret_tmp")
	os.system("echo disassemble main | gdb secret_tmp > gdb.txt")
	os.system("(./secret_tmp; ./secret_tmp CORRECT_SERIAL; ./secret_tmp WRONG_SERIAL) > std_output.txt")
	os.system("echo 'aaa\rs main\rpdf' | radare2 secret_tmp > r2.txt")
	while True:
		alter_file()
		if check_normal() and not check_gdb() and not check_r2():
			print("************* Found possible solution ************")
			os.system("rm gdb.txt r2_tmp.txt r2.txt gdb_tmp.txt normal_tmp.txt std_output.txt")
exit()
```
这段代码主要实现了在程序功能正常的情况下对 radare2 和 gdb的实现反调试的功能。<br>
因为Crackme程序中只有一个main函数，所以check_r2()函数主要是要能对main函数进行一定的干扰。<br>
运行程序之后会生成一个secret_tmp的二进制文件。<br>
下面是几组在fuzzing之前和之后的对比<br>

<strong>fuzzing前后程序运行对比</strong><br>
![Github](img/fuz1.jpg)














---
layout: article

title:  filecrypt工具加解密大文件失败问题分析
date:   2020-01-01 10:39:00 +0800
 
tags: openssl
key:  file_crypt_aes
---

最近在使用[filecrypt](https://github.com/liwugang/filecrypt)加密一个大文件时发现在使用AES算法时会失败，有些机器上会异常退出，到底是哪里出现了问题，本文记录下分析和修复的过程。

<!--more-->

## 分析

由于知道是大文件才出现问题，我们使用truncate命令先创建一个5G的文件

> truncate -s 5G big_file

接着使用filecrypt执行AES加密操作
> ./filecrypt  -e -p test -a aes big_file

-e 为进行加密， -p 指定加密密码， -a aes 指定aes加密算法， big_file 即为所要加密的文件。 filecrypt可以在[此处下载](https://github.com/liwugang/filecrypt/releases)。

执行后的错误信息为
> /home/d/sources/tools/filecrypt/big_file encrypt failed!

所以接下来的从错误信息"encrypt failed!"入手进行分析。

### 代码分析

找到"encrypt failed!" 出现的函数[crypt_file](https://github.com/liwugang/filecrypt/blob/master/filecrypt.c#L151)

#### crypt_file

```c
int crypt_file(const char *file_name, int encrypt, int decrypt, const char *password, int algorithm_id) {
... ...
    crypt_operations *ops;
    file_crypt_info crypt_info;
... ...
    ops = get_crypt_ops(crypt_info.crypt_algorithm); // 3. 调用get_crypt_ops来获取ops
    if (!ops) {
        printf("the algorithm: %d is not supported!\n", crypt_info.crypt_algorithm);
        goto CLEANUP;
    }
... ...

    if (encrypt) {
        if (ops->encrypt(&crypt_info, password, source_addr, dest_addr + crypt_info.crypt_file_offset)) { // 2. ops->encrypt函数函数为false
            memcpy(dest_addr, &crypt_info, sizeof(crypt_info));
            memcpy(dest_addr + sizeof(crypt_info), file_name, strlen(file_name) + 1);
            printf("%-30sencrypt done!\n", file_name);
            delete_origin = TRUE;
        } else {
            printf("%-30sencrypt failed!\n", file_name); // 1. 错误日志地方
            delete_new = TRUE;
        }
    } else {
        if (!ops->is_right_password(&crypt_info, password)) {
            printf("%-30sthe password is not right!\n", file_name);
            delete_new = TRUE;
        } else {
            if (ops->decrypt(&crypt_info, password, source_addr + crypt_info.crypt_file_offset, dest_addr)) {
                printf("%-30sdecrypt done!\n", file_name);
                delete_origin = TRUE;
            } else {
                printf("%-30sdecrypt failed!\n", file_name);
                delete_new = TRUE;
            }
        }
    }
... ...
}
```

从日志位置 1 处逆推，可知 2 处ops->encrypt返回为false，但此处是通过函数指针方式调用，需要通过 3 处get_crypt_ops找到实际的调用函数。

#### get_crypt_ops

```c
crypt_operations *get_crypt_ops(int algs) {
    if (algs >= 0 && algs < ALGORITHM_MAX && algs_array[algs] != NULL) {
        return algs_array[algs]; // 根据 algs 索引从 algs_array 取 crypt_operations
    } else {
        return NULL;
    }
}

struct crypt_operations *algs_array[ALGORITHM_MAX];

void init_algs() {
    algs_array[ALGORITHM_XOR] = &xor_crypt_operations; // 异或算法
    algs_array[ALGORITHM_AES] = &aes_crypt_operations; // aes 加解密算法
}

```

由于我们使用的aes算法，此处肯定是aes_crypt_operations

#### aes_crypt_operations

```c
crypt_operations aes_crypt_operations = {
    .get_crypt_file_length = aes_get_crypt_file_length,
    .is_right_password = aes_is_right_password,
    .encrypt = aes_encrypt,
    .decrypt = aes_decrypt,
};
```
即可以看到实际调用的算法是aes_encrypt.

#### aes_encrypt

```c

int aes_encrypt(file_crypt_info *crypt_info, const char *user_password,
                     void *input_data, void *output_data) {
    uint64_t have_encrypted = 0;
    if (!aes_encrypt_common(input_data, crypt_info->file_length, user_password, DEFAULT_IV,
                    output_data, &have_encrypted)) {
        return FALSE; // 1. aes_encrypt_common 调用失败
    }
    if (have_encrypted != crypt_info->crypt_file_length - crypt_info->crypt_file_offset) {
        return FALSE; // 2. have_encrypted 实际加密长度和需要加密长度不符合
    }
    get_password_hash(user_password, crypt_info->key);
    crypt_info->key_length = HASH256_SIZE;
    return TRUE;
}

```
从上面的分析可知，aes_encrypt 返回 false，但是从上面代码来看，有两处都会返回false，所以不能确定那种情况，此时简单方法就是动态调试。

### 动态调试

{:.warning}
要确保编译filecrypt指定了 -g 参数，这样在调试时才能准确断点。

> gdb --args ./filecrypt -e -p test -a aes big_file

```bash
wangshuidan@localhost:/home/d/sources/tools/filecrypt$ gdb --args ./filecrypt -e -p test -a aes big_file 
GNU gdb (Debian 8.3.1-1) 8.3.1
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./filecrypt...
(gdb) b aes_encrypt
Breakpoint 1 at 0x403cf8: file algs/aes.c, line 17.
(gdb) r
Starting program: /home/d/sources/tools/filecrypt/filecrypt -e -p test -a aes big_file
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, aes_encrypt (crypt_info=0x7fffffffda88, user_password=0x409260 "test", input_data=0x7ffeb7aca000, output_data=0x7ffd77ac9161) at algs/aes.c:17
17          uint64_t have_encrypted = 0;
(gdb) n
18          if (!aes_encrypt_common(input_data, crypt_info->file_length, user_password, DEFAULT_IV,
(gdb) 
19                          output_data, &have_encrypted)) {
(gdb) 
18          if (!aes_encrypt_common(input_data, crypt_info->file_length, user_password, DEFAULT_IV,
(gdb) 
22          if (have_encrypted != crypt_info->crypt_file_length - crypt_info->crypt_file_offset) { # 第 2 中情况
(gdb) n
23              return FALSE;
(gdb) p have_encrypted 
$1 = 1073741840 # have_encrypted 值
(gdb) p crypt_info->crypt_file_length  - crypt_info->crypt_file_offset
$2 = 5368709136 # crypt_info->crypt_file_length  - crypt_info->crypt_file_offset 值

```
从上面调试信息可以看到，aes_encrypt_common 是返回true的，不然不能执行到后面。所以错误是由于实际加密长度和需要加密长度不符合导致。
要加密5368709136（5G多，即和文件大小吻合）长度，但实际上只加密了1073741840长度。

#### 调试 aes_encrypt_common

```bash
Breakpoint 1, aes_encrypt_common (input=0x7ffeb7aca000 "", length=5368709120, password=0x55555555d260 "test", iv=0x555555559627 "0123456789012345", out=0x7ffd77ac9161 "", 
    out_length=0x7fffffffda98) at algs/base.c:54
54          int result = FALSE;
(gdb) n
56          uint64_t offset = 0;
(gdb) 
57          int have_done = 0;
(gdb) 
58          int value = 0;
(gdb) 
59          int input_length = 0;
(gdb) 
61          if (!(ctx = EVP_CIPHER_CTX_new())) {
(gdb) 
65          if (1 != EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, password, iv)) {
(gdb) 
70          while (length) {
(gdb) n
71              if (length > INT32_MAX) {
(gdb) 
72                  input_length = INT32_MAX;
(gdb) p length
$1 = 5368709120  # length 长度， INT32_MAX 为2147483647
(gdb) n
76              if (1 != EVP_EncryptUpdate(ctx, out + have_done, &value, input + offset, input_length)) {
(gdb) 
80              length -= input_length;
(gdb) n
81              offset += input_length;
(gdb) 
82              have_done += value;
(gdb) p have_done
$2 = 0
(gdb) p value
$3 = 2147483632 # 第一次加密2147483647字节明文，返回的是2147483632字节的密文
(gdb) n
70          while (length) {
(gdb) 
71              if (length > INT32_MAX) {
(gdb) 
72                  input_length = INT32_MAX;
(gdb) p length
$1 = 5368709120  # length 长度， INT32_MAX 为2147483647
(gdb) 
76              if (1 != EVP_EncryptUpdate(ctx, out + have_done, &value, input + offset, input_length)) {
(gdb) 
80              length -= input_length;
(gdb) 
81              offset += input_length;
(gdb) 
82              have_done += value;
(gdb) p have_done
$4 = 2147483632
(gdb) p value
$5 = -2147483648 # 第二次加密2147483647字节明文，返回的是负数？
(gdb) p have_done 
$2 = -16 # have_done为负数，问题出现在这里

```

两次同样输入都是2147483647字节明文，为什么两次输出不一样，一次是2147483632，而后一次是-2147483648？而后一次为负数导致have_done的值异常。

## AES 加解密

### AES 加密

#### 加密所用到的三个函数

{:.warning}
int EVP_EncryptInit_ex(EVP_CIPHER_CTX *ctx, const EVP_CIPHER *type, ENGINE *impl, unsigned char *key, unsigned char *iv);

该函数是进行AES加密函数的初始化，type为加密类型，key为加密密钥，iv为初始向量。

{:.warning}
int EVP_EncryptUpdate(EVP_CIPHER_CTX *ctx, unsigned char *out, int *outl, unsigned char *in, int inl);

该函数是进行实际的加密，in为要加密的明文，长度用inl指定，out为加密后的密文，outl返回密文长度。

{:.warning}
int EVP_EncryptFinal_ex(EVP_CIPHER_CTX *ctx, unsigned char *out, int *outl);

如果padding开启时，该函数用于加密最后的data数据。

#### 密文长度确定

整体长度确定

{:.warning}
len(密文) = len(明文) + BLOCK_SIZE - len(明文) % BLOCK_SIZE

EVP_EncryptUpdate函数的密文长度确定

{:.warning}
len(密文) = len(处理的明文) = len(明文) / BLOCK_SIZE * BLOCK_SIZE

此处len(明文)是本次输入长度 + 上次未处理的长度

EVP_EncryptUpdate 处理的明文数据是BLOCK_SIZE的整数倍，且为小于等于明文长度的最大数BLOCK_SIZE的倍数。 若BLOCK_SIZE = 16，明文为20，则处理的明文和返回的密文为16，剩下的会在EVP_EncryptFinal_ex函数处理。

EVP_EncryptFinal_ex函数的密文长度确定

{:.warning}
len(密文) = BLOCK_SIZE


### AES 解密

#### 解密所用到的三个函数

{:.warning}
int EVP_DecryptInit_ex(EVP_CIPHER_CTX *ctx, const EVP_CIPHER *type, ENGINE *impl, unsigned char *key, unsigned char *iv);

该函数是进行AES解密函数的初始化，type为加密类型，key为加密密钥，iv为初始向量。

{:.warning}
int EVP_DecryptUpdate(EVP_CIPHER_CTX *ctx, unsigned char *out, int *outl, unsigned char *in, int inl);

该函数是进行实际的解密，in为要解密的密文，长度用inl指定，out为解密后的明文，outl返回明文长度。

{:.warning}
int EVP_DecryptFinal_ex(EVP_CIPHER_CTX *ctx, unsigned char *outm, int *outl);

如果padding开启时，该函数用于解密最后的data数据。

#### 明文长度确定

EVP_DecryptUpdate函数的明长度确定

{:.warning}
len(明文) = len(处理的密文) = (len(密文) - 1) / BLOCK_SIZE * BLOCK_SIZE

此处len(密文)是本次输入长度 + 上次未处理的长度

EVP_DecryptUpdate处理的密文数据是BLOCK_SIZE的整数倍，且为小于密文长度的最大数BLOCK_SIZE的倍数。 若BLOCK_SIZE = 16，密文为16，则不会处理，返回明文为0。

EVP_DecryptFinal_ex函数的密文长度确定

{:.warning}
len(明文) = 实际加密前长度 - 已解密的文件长度

## 继续分析

之前的问题： 两次同样输入都是2147483647字节明文，为什么两次输出不一样，一次是2147483632，而后一次是-2147483648？

第一次处理的明文为： 2147483647 / 16 * 16 = 2147483632，解释了第一次密文长度为2147483632。

第二次要处理的明文长度其实为： 第一次剩余15(2147483647 - 2147483632) + 第二次2147483647 = 2147483662，返回的密文长度为2147483648，32位输出刚好对应是上面结果-2147483648。所以找到问题的原因为调用EVP_EncryptUpdate函数输入最大值不能为INT32_MAX，而应该是INT32_MAX / BLOCK_SIZE * BLOCK_SIZE。

### 修改继续调试

```bash

Breakpoint 1, aes_encrypt_common (input=0x7ffeb7aca000 "", length=5368709120, password=0x55555555d260 "test", iv=0x555555559627 "0123456789012345", out=0x7ffd77ac9161 "", 
    out_length=0x7fffffffda98) at algs/base.c:54
54          int result = FALSE;
(gdb) n
56          uint64_t offset = 0;
(gdb) 
57          int have_done = 0; # int 类型
(gdb) 
58          int value = 0;
(gdb) 
59          int input_length = 0;
(gdb) 
61          if (!(ctx = EVP_CIPHER_CTX_new())) {
(gdb) 
65          if (1 != EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, password, iv)) {
(gdb) 
70          while (length) {
(gdb) 
71              if (length > AES_MAX_SIZE) {
(gdb) 
72                  input_length = AES_MAX_SIZE;
(gdb) 
76              if (1 != EVP_EncryptUpdate(ctx, out + have_done, &value, input + offset, input_length)) {
(gdb) 
80              length -= input_length;
(gdb) p have_done
$1 = 0
(gdb) n
81              offset += input_length;
(gdb) 
82              have_done += value;
(gdb) 
70          while (length) {
(gdb) p have_done
$2 = 2147483632
(gdb) p value
$3 = 2147483632  # 第一次执行完没有问题
(gdb) n
71              if (length > AES_MAX_SIZE) {
(gdb) 
72                  input_length = AES_MAX_SIZE;
(gdb) 
76              if (1 != EVP_EncryptUpdate(ctx, out + have_done, &value, input + offset, input_length)) {
(gdb) 
80              length -= input_length;
(gdb) 
81              offset += input_length;
(gdb) 
82              have_done += value;
(gdb) p have_done 
$4 = 2147483632
(gdb) n
70          while (length) {
(gdb) p have_done  # 第二次完have_done为负数？ 
$5 = -32
(gdb) p value
$6 = 2147483632 # value 没有问题，之前遇到的问题修复

```
从上面可以看到，第二次执行后，EVP_EncryptUpdate 返回的密文value没有问题，不是负数，但是have_done为负数了，这是怎么情况？

### Integer Overflow

从上面看到 have_done 被定义为int类型，第一次执行结果为2147483632， 第二次执行结果也为2147483632，两次相加已经超过INT32_MAX，出现了整数溢出漏洞，所以为负数，修改也比较简单，使用uint64_t类型代替。

## 总结

至此整个分析过程就结束，也找到问题所在，修复后验证没有问题。针对本地分析过程的总结：

* 大于INT32_MAX的文件现在很常见，所以在处理时要使用64位整数进行文件长度的处理；
* 由于加解密算法中提供的明文和密文长度都是int类型，在处理时要考虑加密算法中明文和密文长度如何确定，防止整数溢出。

## 修复

修复patch [https://github.com/liwugang/filecrypt/commit/12a9dd49326cfe1761a0299127a45a13ae89950c](https://github.com/liwugang/filecrypt/commit/12a9dd49326cfe1761a0299127a45a13ae89950c)








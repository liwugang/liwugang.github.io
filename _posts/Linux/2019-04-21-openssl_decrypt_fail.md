---
layout: article

title:  openssl AES密钥和iv长度问题分析
date:   2019-04-21 15:13:00 +0800
 
tag: openssl
key: openssl_aes
---


在做[filecrypt](https://github.com/liwugang/filecrypt)项目时花费时间最多的是AES256算法的调试上，出现的问题是：
调用完加密函数然后直接调用解密函数，这样是可以正确解密的，但是调用完加密函数后将密文保存在文件后，然后重新使用程序进行解密却是无法正常解密，本文分析下该问题的原因。

# 例子

```c

int aes_encrypt_common(uint8_t *input, uint64_t length, const unsigned char *password,
        const unsigned char *iv, uint8_t *out, uint64_t *out_length);
int aes_decrypt_common(uint8_t *input, uint64_t length, const unsigned char *password,
        const unsigned char *iv, uint8_t *out, uint64_t *out_length);

// 上述加解密函数来自于 https://github.com/liwugang/filecrypt/blob/master/algs/base.c

int main() {
    int i;
    char text[] = "test";
    char *cipher = (char *) malloc(1024);
    char *plain  = (char *) malloc(1024);

    char key[] = "1234567890"; // 密钥
    char iv[] = "1111111"; // iv

    uint64_t out_length = 0;

    // 加密
    int ret = aes_encrypt_common(text, strlen(text), key, iv, cipher, &out_length); 
    // 使用和加密一样的密钥和iv进行解密
    ret = aes_decrypt_common(cipher, out_length, key, iv, plain, &out_length);
    printf("first:%d\n", ret);

    // 再次调用解密，密钥和iv是复制过来的
    char *another_key = (char *) calloc(1, strlen(key) + 1);
    char *another_iv = (char *) calloc(1, strlen(iv) + 1);
    ret = aes_decrypt_common(cipher, out_length, another_key, another_iv, plain, &out_length);
    printf("second:%d\n", ret);
}

```
大家认为上述两次执行解密一样吗？ 

来看下执行结果

```c
first:1
139868329146176:error:0606506D:digital envelope routines:EVP_DecryptFinal_ex:wrong final block length:crypto/evp/evp_enc.c:559: // 出错日志
second:0
```
可以看到两次不一样，第一次1为成功，第二次0为失败，按道理密钥和iv的字符串完全相同，为什么会这样？下面需要深入openssl来探个究竟.

# 代码分析

## openssl 下载编译
加解密使用的是openssl，而默认情况是没有开调试的，所以需要我们单独编译debug版本的openssl来方便调试。openssl可以自己在官网下载，或者使用我下载的版本：http://artfiles.org/openssl.org/snapshot/openssl-SNAP-20190419.tar.gz， 使用下面进行编译debug版本
> ./config -d && make

然后将编译出来的静态库链接到我们程序中，由于libcrypto.a依赖于pthread和dl库，需要添加上 -pthread -ldl
> gcc -o  openssl_test openssl_test.c  ../openssl-1.1.1b/libcrypto.a -pthread -ldl -g

从[aes_decrypt_common](https://github.com/liwugang/filecrypt/blob/master/algs/base.c#L98)源码中看到，密钥和iv是通过EVP_DecryptInit_ex来传递的，将下来从EVP_DecryptInit_ex来分析密钥和iv如何被使用的：

```c
int EVP_DecryptInit_ex(EVP_CIPHER_CTX *ctx, const EVP_CIPHER *cipher,
                       ENGINE *impl, const unsigned char *key,
                       const unsigned char *iv)
{
    return EVP_CipherInit_ex(ctx, cipher, impl, key, iv, 0);
}
// 接着看EVP_CipherInit_ex

int EVP_CipherInit_ex(EVP_CIPHER_CTX *ctx, const EVP_CIPHER *cipher,
                      ENGINE *impl, const unsigned char *key,
                      const unsigned char *iv, int enc)
{
        ... ...
        ctx->cipher = cipher; // ctx->cipher是我们传的cipher
        ... ...
      
        ctx->key_len = cipher->key_len; // ctx->key_len 是来自ciphter中的key_len
        ... ...

    if (!(EVP_CIPHER_flags(EVP_CIPHER_CTX_cipher(ctx)) & EVP_CIPH_CUSTOM_IV)) {
        switch (EVP_CIPHER_CTX_mode(ctx)) {

        case EVP_CIPH_STREAM_CIPHER:
        case EVP_CIPH_ECB_MODE:
            break;

        case EVP_CIPH_CFB_MODE:
        case EVP_CIPH_OFB_MODE:

            ctx->num = 0;
            /* fall-through */

        case EVP_CIPH_CBC_MODE: // 我们是使用CBC

            OPENSSL_assert(EVP_CIPHER_CTX_iv_length(ctx) <=
                           (int)sizeof(ctx->iv));
            if (iv)
                memcpy(ctx->oiv, iv, EVP_CIPHER_CTX_iv_length(ctx)); // iv是直接拷贝相应的长度，和字符串是否'\0'无关，从名字看像是iv的字节数，后面在看EVP_CIPHER_CTX_iv_length
            memcpy(ctx->iv, ctx->oiv, EVP_CIPHER_CTX_iv_length(ctx));
            break;

        case EVP_CIPH_CTR_MODE:
            ctx->num = 0;
            /* Don't reuse IV for CTR mode */
            if (iv)
                memcpy(ctx->iv, iv, EVP_CIPHER_CTX_iv_length(ctx));
            break;

        default:
            return 0;
        }
    }

    if (key || (ctx->cipher->flags & EVP_CIPH_ALWAYS_CALL_INIT)) {
        if (!ctx->cipher->init(ctx, key, iv, enc)) // 此处对密钥key进行处理，通过调试可知实际调用aesni_init_key
            return 0;
    }
    ctx->buf_len = 0;
    ctx->final_used = 0;
    ctx->block_mask = ctx->cipher->block_size - 1;
    return 1;
}

// 接下来看密钥如何处理
static int aesni_init_key(EVP_CIPHER_CTX *ctx, const unsigned char *key,
                          const unsigned char *iv, int enc)
{
    ... ...
    if ((mode == EVP_CIPH_ECB_MODE || mode == EVP_CIPH_CBC_MODE)
        && !enc) {  // 走这里， CBC模式并且enc == 0
        // 第一个参数为我们提供的密钥，第二个参数为key的bits长度
        ret = aesni_set_decrypt_key(key, EVP_CIPHER_CTX_key_length(ctx) * 8,
                                    &dat->ks.ks);
    ... ... 
}
// aesni_set_decrypt_key是汇编实现，函数调用参数从左到右传递方式：rdi, rsi, rdx, rcx, r8d, r9d，key是第一个参数，长度是第二个参数，需要关注rdi和rsi就行

__aesni_set_encrypt_key:
.cfi_startproc	
    ... ...
	movl	$268437504,%r10d
	movups	(%rdi),%xmm0 // 将key的前16字节放到xmm0中
    ... ...
	cmpl	$256,%esi // 判断长度
	je	.L14rounds // 如果长度是256，则跳转到L14rounds
	cmpl	$192,%esi
	je	.L12rounds // 如果长度是192，则跳转到L12rounds
	cmpl	$128,%esi
	jne	.Lbad_keybits // 若长度不是128的话，则keybits是错误的，所以可以看到keybits只支持128,192和256


.L12rounds:
	movq	16(%rdi),%xmm2 // 是将key + 16的8字节放在xmm2中
	movl	$11,%esi
	cmpl	$268435456,%r10d
	je	.L12rounds_alt
    ... ...

.L14rounds:
	movups	16(%rdi),%xmm2 // 此时是将key + 16的16字节放到xmm2中
	movl	$13,%esi
	leaq	16(%rax),%rax
	cmpl	$268435456,%r10d
	je	.L14rounds_alt
    ... ...

// 该函数总结为： length只能为256, 192和128. 若length是256，取key的32(16+16)字节，若length为192,取key的24(16+8)字节，length为128,只取16字节。
```
iv是使用EVP_CIPHER_CTX_iv_length(ctx)字节数，key的使用EVP_CIPHER_CTX_key_length(ctx)字节数，接下来来看这些值怎么确定。
```c
int EVP_CIPHER_CTX_iv_length(const EVP_CIPHER_CTX *ctx)
{
    return ctx->cipher->iv_len;
}

int EVP_CIPHER_CTX_key_length(const EVP_CIPHER_CTX *ctx)
{
    return ctx->key_len;
}

// 通过上面分析得到， iv_len和key_len分别为我们传进去的cipher的中iv_len和key_len，我们是使用EVP_aes_256_cbc()来创建的cipher。而该函数是通过下面宏定义的，而该函数返回的变量也是通过宏定义的

const EVP_CIPHER *EVP_aes_##keylen##_##mode(void) \
{ return &aes_##keylen##_##mode; }

# define BLOCK_CIPHER_generic(nid,keylen,blocksize,ivlen,nmode,mode,MODE,flags) \
static const EVP_CIPHER aes_##keylen##_##mode = { \
        nid##_##keylen##_##nmode,blocksize,keylen/8,ivlen, \
        flags|EVP_CIPH_##MODE##_MODE,   \
        aes_init_key,                   \
        aes_##mode##_cipher,            \
        NULL,                           \
        sizeof(EVP_AES_KEY),            \
        NULL,NULL,NULL,NULL }; \

// 逆向找到调用的地方：
BLOCK_CIPHER_generic(nid,keylen,16,16,cbc,cbc,CBC,flags|EVP_CIPH_FLAG_DEFAULT_ASN1)
// 接着往上找
#define BLOCK_CIPHER_generic_pack(nid,keylen,flags)             \
        BLOCK_CIPHER_generic(nid,keylen,16,16,cbc,cbc,CBC,flags|EVP_CIPH_FLAG_DEFAULT_ASN1)     \
        BLOCK_CIPHER_generic(nid,keylen,16,0,ecb,ecb,ECB,flags|EVP_CIPH_FLAG_DEFAULT_ASN1)      \
        BLOCK_CIPHER_generic(nid,keylen,1,16,ofb128,ofb,OFB,flags|EVP_CIPH_FLAG_DEFAULT_ASN1)   \
        BLOCK_CIPHER_generic(nid,keylen,1,16,cfb128,cfb,CFB,flags|EVP_CIPH_FLAG_DEFAULT_ASN1)   \
        BLOCK_CIPHER_generic(nid,keylen,1,16,cfb1,cfb1,CFB,flags)       \
        BLOCK_CIPHER_generic(nid,keylen,1,16,cfb8,cfb8,CFB,flags)       \
        BLOCK_CIPHER_generic(nid,keylen,1,16,ctr,ctr,CTR,flags)

// 最终找到
BLOCK_CIPHER_generic_pack(NID_aes, 256, 0)

// EVP_aes_256_cbc() 为 LOCK_CIPHER_generic_pack(NID_aes, 256, 0) ==> BLOCK_CIPHER_generic(nid,256,16,16,cbc,cbc,CBC,EVP_CIPH_FLAG_DEFAULT_ASN1) ==> aes_256_cbc {nid_256_cbc, 16, 256/8, 16, EVP_CIPH_FLAG_DEFAULT_ASN1|EVP_CIPH_cbc_MODE, aes_init_key, aes_cbc_cipher, NULL, sizeof(EVP_AES_KEY), NULL,NULL,NULL,NULL }

// 从EVP_CIPHER结构体中看到第三个和第四个变量分别为key_len和iv_len，针对EVP_aes_256_cbc() key_len和iv_len分别为32字节和16字节。

```
所以，可以看到上面加密时密钥和iv分别取32字节和16字节，不管字符串是否有'\0'，上面例子中的第一次解密使用和加密同样的密钥和iv，所以是相同的，而第二次解密使用的密钥和iv只是前面strlen(key) + 1和strlen(iv) + 1相同，所以解密失败。

# 总结

* openssl针对不同模式加密和解密密钥和iv是固定的，所以加密和解密提供的固定长度的密钥和iv都要一致，而不是部分一致，如上述例子中的。
* 超过密钥和iv的部分将不参与到运算中去， 即256的密钥是32位，若两个密钥长度大于32字节，并且前32位相同都为正确密钥，那么这两个密钥都可以正确解密密文。
* 在遇到该问题后，可能有的人觉得openssl库很复杂，就不敢去调试或继续跟踪问题，但其实只有我们有耐心并且勇于尝试，最后发现可能很容易就找到关键地方并解决问题。


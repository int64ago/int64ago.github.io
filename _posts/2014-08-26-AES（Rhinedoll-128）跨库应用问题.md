---
layout:     post
title:      AES（Rhinedoll-128）跨库应用问题
date:       2014-08-26 09:00:00
summary:    AES算是标准加密算法了，跟计算机有关的应该都接触过，但是在跨库应用的时候还是很可能出问题
---

高级加密标准（英语：Advanced Encryption Standard，缩写：AES），在密码学中又称Rijndael（发音近于 "Rhine doll"）加密法，是美国联邦政府采用的一种区块加密标准。这个标准用来替代原先的DES，已经被多方分析且广为全世界所使用。2006年，高级加密标准已然成为对称密钥加密中最流行的算法之一。

以上修改自[wiki][1]

----------

对于这个算法的底层的细节我也没能力研究，不过框架可以用一个图来说明：

![](https://dn-getlink.qbox.me/9119f97a-2cd1-11e4-a160-31fb6a356569.png)

因为这里主要不是讲这个算法本身，所以就没啥理论的东西了，现就遇到的问题总结下

**存储形式**
----

原文/密文/密钥 都是unsigned char类型（hex），很多时候这些都不是hex类型，因为很多应用需要以文本的形式传输，通常有两种"文本"类型：

 - 按照hex字节码存放，如，`0x0a771bcf`——>`0A771BCF`
 - 转为base64，如，`0x0a771bcf`——>`Cncbzw==`

**真实密钥**
----

以openssl为例

`echo "helloworld" | openssl enc -aes-128-cbc -nosalt -pass pass:password`

得到的结果为：

`0000000: fa7d d88c 8f9d d990 ffac 557a 69ae 0a82`

注意这里的pass字段，很多实现都是限制密码长度为16bytes(=128bits)的，但是这里却没有限制，其实，这里的pass并不是真正的密钥，真正的密钥是key，而且还要结合初始化向量iv，这两个会由pass生成：

```
[root@localhost AES-C]$ openssl enc -aes-128-cbc -nosalt -pass pass:passwordpassword -P
key=9DBB300E28BC21C8DAB41B01883918EB
iv =6E63301487DFD4F0EE00A3D12D502D30
```

当然openssl可以只指定key和iv，而不需要pass，pass生成key和iv的例子程序如下（无salt版）：

```c
//gcc -lcrypto
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <openssl/evp.h>

int main(int argc, char *argv[])
{
        const EVP_CIPHER *cipher;
        const EVP_MD *dgst = NULL;
        unsigned char key[EVP_MAX_KEY_LENGTH], iv[EVP_MAX_IV_LENGTH];
        const char *password = "passwordpassword";
        const unsigned char *salt = NULL;
        int i;
        OpenSSL_add_all_algorithms();
        cipher = EVP_get_cipherbyname("aes-128-cbc");
        if(!cipher) { fprintf(stderr, "no such cipher\n"); return 1; }
        dgst=EVP_get_digestbyname("md5");
        if(!dgst) { fprintf(stderr, "no such digest\n"); return 1; }
        if(!EVP_BytesToKey(cipher, dgst, salt,
                                (unsigned char *) password,
                                strlen(password), 1, key, iv)){
                fprintf(stderr, "EVP_BytesToKey failed\n");
                return 1;
        }
        printf("Key: "); for(i=0; i<cipher->key_len; ++i) { printf("%02x", key[i]); } printf("\n");
        printf("IV: "); for(i=0; i<cipher->iv_len; ++i) { printf("%02x", iv[i]); } printf("\n");
        return 0;
}
```

**IV及padding**
------------

默认情况下，AES的IV为全零，padding为PKCS5Padding，如果在不同库之间进行加解密，可能不一样，比如[mcrypt][2]就是NoPadding，而javax.crypto.Cipher和mcrypt之间交互就需要指定为`Cipher.getInstance("AES/CBC/NoPadding")`
[这里][3]是Cipher(java)和mcrypt(c)加解密的例子

特殊行为
有的库不知道出于什么原因，没有按照正常的规则实现AES，比如[gwt-crypto][4]，当需要在GWT的client端实现加密，C写的客户端解密，由于GWT本身的限制，只能用此库，可是它的行为很诡异，生成的密文和其它库实现的都不一样，C当然解不开了……
调试的过程中发现doFinal上有这样一句注释：

> Process the last block in the buffer. If the buffer is currently
> full and padding needs to be added a call to doFinal will produce
> 2 * getBlockSize() bytes.

**而且**，代码里还有个这个过程（这里的cbcV为iv）：

```c
 /*
   * XOR the cbcV and the input,
   * then encrypt the cbcV
 */
for (int i = 0; i < blockSize; i++)
{
      cbcV[i] ^= in[inOff + i];
}
```

这种情况基本打消了用现有c库实现解密的念头，于是找了个openssl剥离出来的c自己改了，下面是一个主要文件，其它见[github][5]：

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "ex_aes_decrypt.h"

int char2int(char ch)
{
        if(ch >= 'a' && ch <= 'z') return ch - 'a' + 10;
        else return ch - '0';
}
void hex2bin(char *hex, int len, u8 *bin)
{
        int i;
        for(i = 0; i < len; ++i){
                bin[i] = (u8)((char2int(hex[i*2])<<4) + char2int(hex[i*2 + 1]));
        }
}

void char2bin(char *ch, int len, u8 *bin)
{
        int i;
        for(i = 0; i < len; ++i){
                bin[i] = (u8)ch[i];
        }
}
void bin2char(char *ch, int len, u8 *bin)
{
        int i;
        for(i = 0; i < len; ++i){
                ch[i] = (char)bin[i];
        }
}

int getPandding(u8 *bin)
{
        int index = 0;
        while(index < AES_BLOCK_SIZE){
                int i, flag = 0;
                for(i = index; i < AES_BLOCK_SIZE; ++i){
                        if(AES_BLOCK_SIZE - index != bin[i]){
                                flag = 1;
                                break;
                        }
                }
                if(flag == 0) return index;
                ++index;
        }
        return index;
}
/*
 *   By Cody Chan<int64ago@gmail.com>
 *   iv[] = {0}
 *   PKCS5Padding
 *   key must be 16 bytes
 *   in_len%16 == 0
 *   return: out
 */
char *ex_aes_decrypt(char *in, int in_len, char *key_)
{
        in_len /= 2;
        AES_KEY key;
        u8 key_bin[AES_BLOCK_SIZE];
        u8 *in_bin = (u8*)malloc(sizeof(u8)*in_len);
        u8 *out_bin = (u8*)malloc(sizeof(u8)*in_len + 1);
        char *out = (char *)malloc(sizeof(char)*in_len + 1);
        char iv[AES_BLOCK_SIZE] = {0};

        hex2bin(in, in_len, in_bin);
        char2bin(key_, AES_BLOCK_SIZE, key_bin);
        AES_set_decrypt_key(key_bin, 128, &key);

        int blocks = in_len / AES_BLOCK_SIZE;
        int i;
        for(i = 0; i < blocks; ++i){
                AES_decrypt(in_bin + i*AES_BLOCK_SIZE,
                                out_bin + i*AES_BLOCK_SIZE, &key);
                int j;
                for(j = 0; j < AES_BLOCK_SIZE; ++j){
                        out_bin[i*AES_BLOCK_SIZE + j] ^= iv[j];
                        iv[j] = in_bin[i*AES_BLOCK_SIZE + j];
                }
        }
        int padding = getPandding(out_bin + (blocks-1)*AES_BLOCK_SIZE);
        out_bin[(blocks-1)*AES_BLOCK_SIZE + padding] = '\0';
        bin2char(out, in_len, out_bin);
        free(in_bin);
        free(out_bin);
        return out;
}
```

**Tips**
----

当然上面的注意事项只有在跨库使用的时候才可能会发生，如果用的同一个库，一般都有接口非常统一的encrypt和decrypt，放心用即可


  [1]: https://zh.wikipedia.org/wiki/Advanced_Encryption_Standard
  [2]: http://sourceforge.net/projects/mcrypt/files/Libmcrypt/
  [3]: https://gist.github.com/int64Ago/bc816bd950b179e04955
  [4]: https://code.google.com/p/gwt-crypto/
  [5]: https://github.com/int64Ago/AES-C

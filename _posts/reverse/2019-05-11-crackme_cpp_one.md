---
layout: post

title:  简单C++ crackme分析
date:   2019-05-11 15:20:00 +0800
categories: reverse
tag: crackme
---

* content
{:toc}



该crackme来自于[这里](https://www.root-me.org/en/Challenges/Cracking/ELF-C-0-protection),也可以从[这里]({{ "/assets/crackme/cpp_001" | absolute_url }})下载，是Linux平台上的C++程序，没加壳，比较简单，故将分析记录下来。

工具选择
===============
Linux静态分析工具使用[cutter](https://github.com/radareorg/cutter), ，但由于cutter对C++的函数识别不好，故也使用IDA一起进行分析。

静态分析
==============

先静态分析，了解程序的大概轮廓，并找到关键部分，在结合动态调试，可以达到事半功倍的效果。

### 搜索关键字符串
```c
.rodata:08048DFC	00000037	C	Bravo, tu peux valider en utilisant ce mot de passe... 
.rodata:08048E34	00000031	C	Congratz. You can validate with this password... // 可以推断出来这是成功的提示
.rodata:08048E65	00000014	C	Password incorrect. // 这是失败的提示
```

### 根据字符串来看流程
![graph]({{"/assets/pictures/crackme_cpp_one/graph.jpg" | prepend:site.baseurl}})

图中蓝色部分为上面成功的提示的地方，所以需要执行的代码路径为1--> 2 --> 3。

## 模块1
```asm
  int main (int argc, char **argv, char **envp);
|           ; var int var_16h @ ebp-0x16
|           ; var int var_15h @ ebp-0x15
|           ; var int var_14h @ ebp-0x14
|           ; var int var_10h @ ebp-0x10
|           ; var int var_ch @ ebp-0xc
|           ; var int var_8h_2 @ ebp-0x8
|           ; var char *var_4h @ esp+0x4
|           ; var int var_8h @ esp+0x8
|           ; DATA XREF from entry0 (0x80488a7)
|           0x08048a86      lea       ecx, [var_4h] ; 为argc的地址
|           0x08048a8a      and       esp, 0xfffffff0
|           0x08048a8d      push      dword [ecx - 4]
|           0x08048a90      push      ebp
|           0x08048a91      mov       ebp, esp
|           0x08048a93      push      ebx
|           0x08048a94      push      ecx
|           0x08048a95      sub       esp, 0x20
|           0x08048a98      mov       ebx, ecx
|           0x08048a9a      cmp       dword [ebx], 1；对argc即参数的个数和1对比
|       ,=< 0x08048a9d      jg        0x8048aee；如果大于1, 跳转到0x8048aee,也即我们的第2部分

```

需要我们至少password作为第2个参数。

## 模块2
```asm
text:08048AEE loc_8048AEE:                            ; CODE XREF: main+17↑j
.text:08048AEE                 lea     eax, [ebp+var_15]
.text:08048AF1                 mov     [esp], eax
.text:08048AF4                 call    __ZNSaIcEC1Ev   ; std::allocator<char>::allocator(void)
.text:08048AF9                 lea     eax, [ebp+var_15]
.text:08048AFC                 mov     [esp+8], eax ; 参数3
.text:08048B00                 mov     dword ptr [esp+4], offset unk_8048DC4 ;参数2
.text:08048B08                 lea     eax, [ebp+var_C]
.text:08048B0B                 mov     [esp], eax ;参数1
.text:08048B0E                 call    __ZNSsC1EPKcRKSaIcE ; std::string::string(char const*,std::allocator<char> const&) ;创建string1，是以unk_8048DC4作为字符串，保存在[ebp+var_C]
.text:08048B13                 lea     eax, [ebp+var_16]
.text:08048B16                 mov     [esp], eax
.text:08048B19                 call    __ZNSaIcEC1Ev   ; std::allocator<char>::allocator(void)
.text:08048B1E                 lea     eax, [ebp+var_16]
.text:08048B21                 mov     [esp+8], eax ; 参数3
.text:08048B25                 mov     dword ptr [esp+4], offset unk_8048DCC ;参数2
.text:08048B2D                 lea     eax, [ebp+var_10]
.text:08048B30                 mov     [esp], eax ;参数1
.text:08048B33                 call    __ZNSsC1EPKcRKSaIcE ; std::string::string(char const*,std::allocator<char> const&) ;创建string2，是以unk_8048DCC作为字符串，保存在[ebp+var_10]
.text:08048B38                 lea     eax, [ebp+var_14]
.text:08048B3B                 lea     edx, [ebp+var_C]
.text:08048B3E                 mov     [esp+8], edx ;参数3 上面string1
.text:08048B42                 lea     edx, [ebp+var_10]
.text:08048B45                 mov     [esp+4], edx ;参数2 上面string2
.text:08048B49                 mov     [esp], eax ;参数1， 保存在[ebp+var_14]
.text:08048B4C                 call    _Z5ploufSsSs    ; plouf(std::string,std::string)
.text:08048B51                 sub     esp, 4
.text:08048B54                 lea     eax, [ebp+var_10]
.text:08048B57                 mov     [esp], eax      ; this
.text:08048B5A                 call    __ZNSsD1Ev      ; std::string::~string() ;string2 回收
.text:08048B5F                 lea     eax, [ebp+var_16]
.text:08048B62                 mov     [esp], eax
.text:08048B65                 call    __ZNSaIcED1Ev   ; std::allocator<char>::~allocator()
.text:08048B6A                 lea     eax, [ebp+var_C]
.text:08048B6D                 mov     [esp], eax      ; this
.text:08048B70                 call    __ZNSsD1Ev      ; std::string::~string() ;string1 回收
.text:08048B75                 lea     eax, [ebp+var_15]
.text:08048B78                 mov     [esp], eax
.text:08048B7B                 call    __ZNSaIcED1Ev   ; std::allocator<char>::~allocator()
.text:08048B80                 mov     eax, [ebx+4] ; argv
.text:08048B83                 add     eax, 4 ; argv + 4
.text:08048B86                 mov     eax, [eax] ; 我们传入的参数
.text:08048B88                 mov     [esp+4], eax    ; char * ;参数2 我们传入的参数
.text:08048B8C                 lea     eax, [ebp+var_14]
.text:08048B8F                 mov     [esp], eax      ; std::string * ; 参数1， 上述plouf的参数1
.text:08048B92                 call    _ZSteqIcSt11char_traitsIcESaIcEEbRKSbIT_T0_T1_EPKS3_ ; std::operator==<char,std::char_traits<char>,std::allocator<char>>(std::basic_string<char,std::char_traits<char>,std::allocator<char>> const&,char const*) ; 进行对比
.text:08048B97                 test    al, al
.text:08048B99                 jz      short loc_8048BE5 ;对比结果为0,则跳转到loc_8048BE5
模块3
.text:08048B9B                 mov     dword ptr [esp+4], offset aBravoTuPeuxVal ; "Bravo, tu peux valider en utilisant ce "...
.text:08048BA3                 mov     dword ptr [esp], offset _ZSt4cout@@GLIBCXX_3_4
.text:08048BAA                 call    __ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc ; std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &,char const*)
.text:08048BAF                 mov     dword ptr [esp+4], offset __ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_ ; std::endl<char,std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &)
.text:08048BB7                 mov     [esp], eax
.text:08048BBA                 call    __ZNSolsEPFRSoS_E ; std::ostream::operator<<(std::ostream & (*)(std::ostream &))
.text:08048BBF                 mov     dword ptr [esp+4], offset aCongratzYouCan ; "Congratz. You can validate with this pa"... ; 成功的字符串
.text:08048BC7                 mov     dword ptr [esp], offset _ZSt4cout@@GLIBCXX_3_4
.text:08048BCE                 call    __ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc ; std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &,char const*)
.text:08048BD3                 mov     dword ptr [esp+4], offset __ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_ ; std::endl<char,std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &)
.text:08048BDB                 mov     [esp], eax
.text:08048BDE                 call    __ZNSolsEPFRSoS_E ; std::ostream::operator<<(std::ostream & (*)(std::ostream &))
.text:08048BE3                 jmp     short loc_8048C09
```

总结下：

* 以unk_8048DC4处的字符串构造string1；
* 以unk_8048DCC处的字符串构造string2；
* 执行plouf(addr, string2, string1)；
* 将addr和我们传入的密码进行对比，若相同则成功。

## plouf 函数分析

```asm
.text:0804898D ; plouf(std::string, std::string)
.text:0804898D                 public _Z5ploufSsSs
.text:0804898D _Z5ploufSsSs    proc near               ; CODE XREF: main+C6↓p
.text:0804898D
.text:0804898D var_1D          = byte ptr -1Dh
.text:0804898D var_1C          = dword ptr -1Ch
.text:0804898D arg_0           = dword ptr  8
.text:0804898D arg_4           = dword ptr  0Ch
.text:0804898D arg_8           = dword ptr  10h
.text:0804898D
.text:0804898D                 push    ebp
.text:0804898E                 mov     ebp, esp
.text:08048990                 push    edi
.text:08048991                 push    esi
.text:08048992                 push    ebx
.text:08048993                 sub     esp, 2Ch
.text:08048996                 lea     eax, [ebp+var_1D]
.text:08048999                 mov     [esp], eax
.text:0804899C                 call    __ZNSaIcEC1Ev   ; std::allocator<char>::allocator(void)
.text:080489A1                 lea     eax, [ebp+var_1D]
.text:080489A4                 mov     [esp+8], eax ;参数3
.text:080489A8                 mov     dword ptr [esp+4], offset unk_8048DB0 ;参数2
.text:080489B0                 mov     eax, [ebp+arg_0]
.text:080489B3                 mov     [esp], eax ;我们传入的参数1
.text:080489B6                 call    __ZNSsC1EPKcRKSaIcE ; std::string::string(char const*,std::allocator<char> const&) ;创建string1，是以unk_8048DB0作为字符串，保存在传入的参数1
.text:080489BB                 lea     eax, [ebp+var_1D]
.text:080489BE                 mov     [esp], eax
.text:080489C1                 call    __ZNSaIcED1Ev   ; std::allocator<char>::~allocator()
.text:080489C6                 mov     [ebp+var_1C], 0 ; index
.text:080489CD                 jmp     short loc_8048A2B
.text:080489CF ; ---------------------------------------------------------------------------
.text:080489CF
.text:080489CF loc_80489CF:                            ; CODE XREF: plouf(std::string,std::string)+BA↓j
.text:080489CF                 mov     eax, [ebp+var_1C]
.text:080489D2                 mov     [esp+4], eax ; 作为下标
.text:080489D6                 mov     eax, [ebp+arg_4] ; 传入的参数2 - string2
.text:080489D9                 mov     [esp], eax
.text:080489DC                 call    __ZNSsixEj      ; std::string::operator[](uint)
.text:080489E1                 movzx   esi, byte ptr [eax] string2[index]
.text:080489E4                 mov     ebx, [ebp+var_1C]
.text:080489E7                 mov     eax, [ebp+arg_8] ;参数的参数3 - string1
.text:080489EA                 mov     [esp], eax      ; this
.text:080489ED                 call    __ZNKSs6lengthEv ; std::string::length(void)
.text:080489F2                 mov     edi, eax ; edi保存 string1的长度
.text:080489F4                 mov     eax, ebx ; index
.text:080489F6                 mov     edx, 0
.text:080489FB                 div     edi
.text:080489FD                 mov     ecx, edx ; = index % string1的长度
.text:080489FF                 mov     eax, ecx
.text:08048A01                 mov     [esp+4], eax
.text:08048A05                 mov     eax, [ebp+arg_8]
.text:08048A08                 mov     [esp], eax
.text:08048A0B                 call    __ZNSsixEj      ; std::string::operator[](uint)
.text:08048A10                 movzx   eax, byte ptr [eax] ; string1[index % string1长度]
.text:08048A13                 xor     eax, esi  ;string[index % string1长度] xor string2[index]
.text:08048A15                 movsx   eax, al
.text:08048A18                 mov     [esp+4], eax
.text:08048A1C                 mov     eax, [ebp+arg_0] ;传入的参数1
.text:08048A1F                 mov     [esp], eax
.text:08048A22                 call    __ZNSspLEc      ; std::string::operator+=(char)
.text:08048A27                 add     [ebp+var_1C], 1 ; 将结果相加到参数1中
.text:08048A2B
.text:08048A2B loc_8048A2B:                            ; CODE XREF: plouf(std::string,std::string)+40↑j
.text:08048A2B                 mov     eax, [ebp+var_1C]
.text:08048A2E                 mov     [esp+4], eax
.text:08048A32                 mov     eax, [ebp+arg_4] ;参数的参数2-string2
.text:08048A35                 mov     [esp], eax
.text:08048A38                 call    __ZNSsixEj      ; std::string::operator[](uint)
.text:08048A3D                 movzx   eax, byte ptr [eax]; string2[index]
.text:08048A40                 test    al, al
.text:08048A42                 setnz   al
.text:08048A45                 test    al, al
.text:08048A47                 jnz     short loc_80489CF ;如果非0,跳转到loc_80489CF
.text:08048A49                 jmp     short loc_8048A79
.text:08048A4B ; ---------------------------------------------------------------------------
.text:08048A4B                 mov     ebx, eax
.text:08048A4D                 lea     eax, [ebp+var_1D]
.text:08048A50                 mov     [esp], eax
.text:08048A53                 call    __ZNSaIcED1Ev   ; std::allocator<char>::~allocator()
.text:08048A58                 mov     eax, ebx
.text:08048A5A                 mov     [esp], eax
.text:08048A5D                 call    __Unwind_Resume
.text:08048A62                 mov     ebx, eax
.text:08048A64                 mov     eax, [ebp+arg_0]
.text:08048A67                 mov     [esp], eax      ; this
.text:08048A6A                 call    __ZNSsD1Ev      ; std::string::~string()
.text:08048A6F                 mov     eax, ebx
.text:08048A71                 mov     [esp], eax
.text:08048A74                 call    __Unwind_Resume
.text:08048A79
.text:08048A79 loc_8048A79:                            ; CODE XREF: plouf(std::string,std::string)+BC↑j
.text:08048A79                 mov     eax, [ebp+arg_0]
.text:08048A7C                 add     esp, 2Ch
.text:08048A7F                 pop     ebx
.text:08048A80                 pop     esi
.text:08048A81                 pop     edi
.text:08048A82                 pop     ebp
.text:08048A83                 retn    4
.text:08048A83 _Z5ploufSsSs    endp
.text:08048A83
```

用伪代码来描述下上述函数：

```c
    plouf(addr, string2, string1) {
        string result = &addr; // 第1个参数地址
        for (int i = 0; i < len(string2); i++) { // 遍历string2
            temp = string2[i] ^ string1[i % len(string1)]; // 将string2的每个元素和 string1 相关地址进行异或
            result += temp;
        }
    }
    
```

## 求出password

### string1内容
```asm
.rodata:08048DC4 unk_8048DC4     db  18h                 ; DATA XREF: main+7A↑o
.rodata:08048DC5                 db 0D6h
.rodata:08048DC6                 db  15h
.rodata:08048DC7                 db 0CAh
.rodata:08048DC8                 db 0FAh
.rodata:08048DC9                 db  77h ; w
.rodata:08048DCA                 db    0
```
### string2 内容
```asm
.rodata:08048DCC unk_8048DCC     db  50h ; P             ; DATA XREF: main+9F↑o
.rodata:08048DCD                 db 0B3h
.rodata:08048DCE                 db  67h ; g
.rodata:08048DCF                 db 0AFh
.rodata:08048DD0                 db 0A5h
.rodata:08048DD1                 db  0Eh
.rodata:08048DD2                 db  77h ; w
.rodata:08048DD3                 db 0A3h
.rodata:08048DD4                 db  4Ah ; J
.rodata:08048DD5                 db 0A2h
.rodata:08048DD6                 db  9Bh
.rodata:08048DD7                 db    1
.rodata:08048DD8                 db  7Dh ; }
.rodata:08048DD9                 db  89h
.rodata:08048DDA                 db  61h ; a
.rodata:08048DDB                 db 0A5h
.rodata:08048DDC                 db 0A5h
.rodata:08048DDD                 db    2
.rodata:08048DDE                 db  76h ; v
.rodata:08048DDF                 db 0B2h
.rodata:08048DE0                 db  70h ; p
.rodata:08048DE1                 db 0B8h
.rodata:08048DE2                 db  89h
.rodata:08048DE3                 db    3
.rodata:08048DE4                 db  79h ; y
.rodata:08048DE5                 db 0B8h
.rodata:08048DE6                 db  71h ; q
.rodata:08048DE7                 db  95h
.rodata:08048DE8                 db  9Bh
.rodata:08048DE9                 db  28h ; (
.rodata:08048DEA                 db  74h ; t
.rodata:08048DEB                 db 0BFh
.rodata:08048DEC                 db  61h ; a
.rodata:08048DED                 db 0BEh
.rodata:08048DEE                 db  96h
.rodata:08048DEF                 db  12h
.rodata:08048DF0                 db  47h ; G
.rodata:08048DF1                 db  95h
.rodata:08048DF2                 db  3Eh ; >
.rodata:08048DF3                 db 0E1h
.rodata:08048DF4                 db 0A5h
.rodata:08048DF5                 db    4
.rodata:08048DF6                 db  6Ch ; l
.rodata:08048DF7                 db 0A3h
.rodata:08048DF8                 db  73h ; s
.rodata:08048DF9                 db 0ACh
.rodata:08048DFA                 db  89h
.rodata:08048DFB                 db    0
```

### 程序求password
```c
#include <stdio.h>
#include <string.h>

int string1[] = {0x18, 0xd6, 0x15, 0xca, 0xfa, 0x77}; // 为上面string1
int string2[] = {0x50, 0xB3, 0x67, 0xAF, 0xA5, 0x0E, 0x77, 0xA3, 0x4A, 0xA2, 0x9B, 0x01, 0x7D, 0x89, 0x61, 0xA5, 0xA5, 0x02, 0x76, 0xB2, 0x70, 0xB8, 0x89, 0x03, 0x79, 0xB8, 0x71, 0x95, 0x9B, 0x28, 0x74, 0xBF, 0x61, 0xBE, 0x96, 0x12, 0x47, 0x95, 0x3E, 0xE1, 0xA5, 0x04, 0x6C, 0xA3, 0x73, 0xAC, 0x89}; // 上面string2


int main()
{
    int num = sizeof(string2) / sizeof(string2[0]);
    int i = 0;
    for (; i < num; i++) {
        string2[i] ^= string1[i % (sizeof(string1) / sizeof(string1[0]))];
        printf("%c", string2[i]);
    }
}

```

> 执行结果:Here_you_have_to_understand_a_little_C++_stuffs

总结
============

* C++反编译后函数名会比较复杂，识别函数名难度会比较大，所以需要挑选多个工具一起使用；
* 此题相对简单，只用静态分析就可以，但遇到难度较大的，可以采用静态和动态结合方式。
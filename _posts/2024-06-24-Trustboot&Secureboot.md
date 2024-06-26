---
layout: post
title:  Trustboot Secureboot
category: 开发 
description: 记录一下hostboot上支持trustboot和secureboot的开发过程，从中学习了一下SPI去驱动TPM这种低速接口driver开发和调试，以及安全相关的数字签名，SM2/SM3等相关内容
---

算是第一次从头到尾的开发一个底层的低速接口的驱动了 :frog: ，正好借此机会学习一下“读手册<->写程序”这个流程……，手册用的是st的TPM的datasheet，我们项目中用的TPM没找到手册，而且st的手册比spec要好用得多

## SPI&TPM通信
硬件连接

![](/assets/img/2024-06-24-16-51-44.png)


SPI是一种通信协议，放到硬件上，他应该是一种总线，通信过程中，CPU只负责写入特定格式的**数据流**，具体的行为是由连接的设备来完成的，比如连接了TPM，或者ADC，它们接收了数据流之后会执行操作，例如ADC会自动切换channel，轮流发送数据。  
直白的理解：
- SPI是一种全双工的通信协议，与I2C、UART类似，SPI支持全双工或者半双工的同步传输  
- TPM是一个用来保证image是“可信的”的设备，简单来说通过记录固件启动过程中，对image的measure(hash)操作通过PCR extend来存到TPM，以及事件日志eventlog，利用**TPM的硬件不可更改性**，在OS中通过日志来对image做同样的操作，并比较记录的结果
- Trustboot就是由TPM硬件来保证的

TPM spec上支持SPI/I2C，在hostboot上原本是有基于I2C的版本的，但是由于硬件连接已经变成了SPI，所以根据DD framework，似乎只需要写个SPI driver去驱动TPM就好了，上层的TPM通信已经有了

### TPM command
太长了而且重点也不突出，比如有个很重要的问题，**TPM是半双工通信**，也没有强调，导致看了半天也不知道CS有啥作用，参考了st的TPM datasheet
![](/assets/img/2024-06-24-16-17-28.png)

这里面回答了这个问题，为了实现同步，即告诉TPM通信开始了，TPM的CS信号需要在通信开始时被拉低

![](/assets/img/2024-06-24-16-21-59.png)
接下来就是TPM的命令格式
![](/assets/img/2024-06-24-16-23-14.png)

TPM的命令总共4个字节，格式为：    
cmd[0] : read(1)/wirte(0)  
cmd[1-7] : size of data  
cmd[8-15] : 0xD4似乎是固定是这个  
cmd[16-31] : addr, 要操作的TPM的地址  

### TPM reg
这些reg在data sheet中都有描述，比较重要的是这个TPM_ACCESS，在执行TPM startup之前需要先执行这个TPM_ACCESS_0  
![](/assets/img/2024-06-24-16-44-01.png)

hostboot的初始化TPM的流程是没有这个的，不知道是不是I2C版本和SPI版本的区别  
也就是通过上面的command来发送读写命令，向这个reg来写值，代码如下：
![](/assets/img/2024-06-24-16-47-42.png)
这里面设计了一个polling操作，因为TPM执行写寄存器这个执行需要一定的时间，这个polling的时间设置为10ms，一共polling20次，实际上是用不了这么长时间，根据具体的datasheet来，没找到我这个项目使用的TPM的datasheet所以就多延时一下……

## SPI driver
SPI跟qspi类似，都是需要注册到DD framwork上
### read & write
read操作
![](/assets/img/2024-06-24-16-55-36.png)


同样write操作：
![](/assets/img/2024-06-24-16-59-36.png)

用volatile来直接读写寄存器，这个关键字的作用就是：编译器不做优化，CPU读写内存的时候，CPU不会从cache中去读，而是直接去从内存中读，保证了对寄存器操作的正确性
![](/assets/img/2024-06-24-16-56-46.png)
### SPI reg init
SPI的初始化需要对SPI参数进行配置，并且上面的read write操作中也需要依赖SPI状态寄存器，之前觉得最头疼的就是这些寄存器……描述很抽象，如果配置不对的话程序运行结果也不对，下面整理一下我印象比较深刻的几个SPI寄存器，积累一下“经验”：

#### 1.SSIENR和SER
- SSIENR: ssi_enable, 当这个被disable的时候，**SPI的FIFO会被清空**，整个通信流程会立刻被halt,每次通信完成后需要执行这个来清除一下buffer  
- SER: slave enable register, 这个是用来enable slave的通信的，当SSIENR是1的时候，SER的置0是无效的  
简单理解就是这两个寄存器是用来enable和disable SPI通信的，SSIENR是控制整个串口通信的，SER是用来enable slave的

#### 2.参数相关
通信过程中的tmod，frame，baudr，scph，scpol
![](/assets/img/2024-06-24-17-20-42.png)
- SCPH和SCPOL: 时钟极性和相位，SCPOL是决定没有数据的时候的CLK的高低，SCPH决定在第0个cycle还是第1个cycle去读数据
- dfs_32: data frame size, 每个数据帧的大小

#### 3.状态相关
由于SPI的slave设备的运行是没有CPU指令执行快的，所以需要有一个状态寄存器来让CPU去polling，确定当前读写状态，这里看寄存器的描述，还是觉得有很多冗余和混乱……
- busy位：当前SPI操作的状态，如果busy=1，表示当前的SPI还没有完成，所以要等SPI完成之后再去读写
- TFNF位：transmit fifo not full， 发送FIFO**非满**，此时可以发送东西
- RFNE位：receive fifo not empty， 接收FIFO**非空**，此时可以读东西
类似的还有TFF,RFE这两个是不是就是冗余了？

## Secureboot
secureboot是防止未经签名的固件被启动，属于“不可抵赖性”， trustboot相当于“完整性”
Secureboot是通过在编译或者打包阶段给flash中image进行签名，来保证image没有被篡改，签名的验签需要在各个阶段进行，签名根据使用的秘钥分为两种，一种是使用root key，用来对fw public key进行签名，fw key用来对fw image进行签名。  
![](/assets/img/2024-06-25-10-45-48.png)
在secureboot的最初的阶段，比较OTPROM中的root public key hash与读取的是否一致
使用root public key来对fw public key进行验签，通过后，使用fw public key来对image进行验签，什么是验签呢？ok, callback回硕士阶段研究过的密码学……  
密码学是为了保证机密性、完整性、可用性，签名算是密码学的应用，是为了保证不可抵赖性，也就是保证经过签名的文件，不会被篡改，签名和验签的流程： 


- 签名： 对源文件进行摘要，目的是生成定长的摘要，用私钥加密，这个就是签名
![](/assets/img/2024-06-25-14-27-52.png)


- 验签： 验签就是使用公钥解密得到哈希值，然后用同样的算法对源文件进行摘要，比较二者的值是否一致  
![](/assets/img/2024-06-25-14-37-10.png)


遗憾的是，这个加密流程可以用openssl提供的库函数来完成，但是解密就不能了，并且在大核上还不能用加解密硬件，因此需要移植一个软件加解密的库，考虑到当前的代码框架，找到了另一个库-mbedtls，只是不支持SM2/SM3这些国密算法，还好在gitee上找到了开源的实现，下面就是移植了，这也算是第一次移植开源代码了……

### mbedtls移植

为什么选择mbedtls呢，因为它的模块之间依赖少……，观察代码目录，这些算法都是单独一个文件，并且它的功能都是可以通过宏定义的配置文件来开启/关闭的，这样设计应该就是为了让它更容易被移植吧  :frog:  

![](/assets/img/2024-06-25-15-15-11.png)

这样就可以把需要的文件copy到hostboot目录下了，这里没有选择lib目录，因为lib目录是放到base image的，会导致hbicore.bin太大了（500k->2M），虽然C2000的OCM足够大……，但是这个库只在securboot当中使用，逻辑上不应该放在lib这种基础的“库”当中，因此考虑单独生成一个.so，用的时候加载进来，不用的时候换出去……  
需要用到加解密的地方有targeting使用HB_DATA.bin，load_payload加载skiboot.lid，vfs加载hb_extend.bin，如果只验证skiboot.lid完全可以放到extend image当中，像libxz.so那样，但是考虑到vfs是在base image当中的，所以还是需要把libmbedtls放到base当中  
编译报错：   
![](/assets/img/2024-06-25-15-47-58.png)

这是链接阶段找不到symbol了，这个链接不是生成.so的链接，而是.elf生成.bin时候的linker，这个是hostboot自己实现的，很明显一个.so当中的symbol是可以重定向到别的.so当中的，但是发生这个错误就是在其他.so找不到了  
![](/assets/img/2024-06-25-15-56-10.png)

再看这个symbol的名字__udivti3，也不是C++/C的函数名，上网查了一下发现是gcc用于计算__uint128_t除法的函数，所以问题出在：  
```C
     /* mbedtls_t_udbl defined as 128-bit unsigned int */
     typedef unsigned int mbedtls_t_udbl __attribute__((mode(TI)));
```

在编译时gcc发现之后把对应的操作编译成了__udivti3，但是hostboot当中默认是不链接libgcc的，因为它要完全控制链接过程，并且也没有stdlib需要的操作系统提供的底层调用，比如printf/malloc之类的底层实现，hostboot跟其他裸机一样，实现了一系列底层的调用，然后实现自己的std**.h，
![](/assets/img/2024-06-25-16-08-27.png)

而mbedtls编译出来的.so是运行在有OS的环境的，或者说是希望运行在有libgcc环境的，所以，解决的办法就是在链接生成libmbedtls.so的时候去链接libgcc，这样对其他模块没有影响  
![](/assets/img/2024-06-25-17-00-26.png)
这里用了固定的路径，因为编译服务器上的buildroot的路径配置的是小端的libgcc，所以使用了gnu目录下的，正常来说应该是用命令获取
```sh
$(shell $(CC) -print-libgcc-file-name)
```

这里面显式链接了一个libgcc.a这个库，可以发现链接之后libmbedtls.so当中找不到_udivti3这个symbol了，**链接器的操作**是：  
找到_udivti3这个symbol在libgcc.a这个静态库当中的实现_udivti3.o，然后替换进来，libgcc.a当中其他的实现不会被放到最终的.so当中。这样hbicore.bin也不会变大太多
![](/assets/img/2024-06-26-15-14-34.png)

编译完之后还有个问题是mbedtls是用宏定义mbedtls_printf，不能把这个定义为printk…看似提供了一些宏定义留了特定的接口，但是这个.c是不能去调用.C的……如果inluce kernel/console.H会导致无法识别C++的关键字，因为.c是用gcc编译，.C是用g++编译  

考虑用g++去编译libmbedtls，从代码中的
```c++
#ifdef __cplusplus
extern C{

}
#endif
```
这个是支持C++编译器的，但是类型检查更为严格，所以需要修改代码，整体来看没必要，因为这个库是不需要暴露运行细节来学习的（因为都是数学运算），也不需要输出log来debug（正确性已经验证并且跟其它模块没有耦合）  


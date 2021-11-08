# 概述

在 RT-Thread 的内核代码中，有一种查找最低非零位的算法，主要采用查表的方法，通过时间换空间的方式，为系统寻找线程的最高优先级提供了一种高效的方法。

通过查看内核代码里面的 `kservice.c` 文件，我们可以看到如下源代码：

```c
const rt_uint8_t __lowest_bit_bitmap[] =
{
    /* 00 */ 0, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 10 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 20 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 30 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 40 */ 6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 50 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 60 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 70 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 80 */ 7, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 90 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* A0 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* B0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* C0 */ 6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* D0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* E0 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* F0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0
};

/**
 * This function finds the first bit set (beginning with the least significant bit)
 * in value and return the index of that bit.
 *
 * Bits are numbered starting at 1 (the least significant bit).  A return value of
 * zero from any of these functions means that the argument was zero.
 *
 * @return return the index of the first bit set. If value is 0, then this function
 * shall return 0.
 */
int __rt_ffs(int value)
{
    if (value == 0) 
        return 0;

    if (value & 0xff)
        return __lowest_bit_bitmap[value & 0xff] + 1;

    if (value & 0xff00)
        return __lowest_bit_bitmap[(value & 0xff00) >> 8] + 9;

    if (value & 0xff0000)
        return __lowest_bit_bitmap[(value & 0xff0000) >> 16] + 17;

    return __lowest_bit_bitmap[(value & 0xff000000) >> 24] + 25;
}
```

通过源代码，我们可以看到，该算法需要一个蛮大的数组(上文是 `256Byte`)，查找 `32bit` 的变量也需要使用判断语句将情况拆分。

如果是查找 `64bit` 大小的变量，需要更多的判断，还会更耗时间。

所以一直在思考是否会有更好的方法，能够更省空间，也能获得非常好的效率。

最近，想到一种算法，与原算法相比，能够兼具时间效率和空间效率优势，本文验证该算法的有效性。

（**PS：感觉还会有更高效的方法，目前个人就只想到这个方法了。**）

下面称 RT_Thread 默认的查找算法为主线版本，而新的查找算法为精简版本。

下面是精简版本算法的源代码：

```c
const rt_uint8_t __lowest_bit_bitmap[] =
{
    /*  0 - 7  */  0,  1,  2, 27,  3, 24, 28, 32,
    /*  8 - 15 */  4, 17, 25, 31, 29, 12, 32, 14,
    /* 16 - 23 */  5,  8, 18, 32, 26, 23, 32, 16,
    /* 24 - 31 */ 30, 11, 13,  7, 32, 22, 15, 10,
    /* 32 - 36 */  6, 21,  9, 20, 19
};

/**
 * This function finds the first bit set (beginning with the least significant bit)
 * in value and return the index of that bit.
 *
 * Bits are numbered starting at 1 (the least significant bit).  A return value of
 * zero from any of these functions means that the argument was zero.
 *
 * @return return the index of the first bit set. If value is 0, then this function
 * shall return 0.
 */
int __rt_ffs(int value)
{
    return __lowest_bit_bitmap_new[(rt_uint32_t)(value & (value - 1) ^ value) % 37];
}
```

该算法的实现原理分析，可以看该篇文章：[ 一种新的寻找字节最低非0位的算法原理分析](https://github.com/Eureka1024/An-effective-method-for-finding-the-lowest-non-0bit-bit-of-byte/blob/main/README.md)。

要验证该算法的存在价值，无非是验证该算法的 `正确性` 和 `高效性`。

本文就从这两个方面验证该算法。



# 验证算法的正确性

其实从原理已经能够得出该算法是准确的，因为证明毕竟不是非常严密，但是为了保守起见，这里采用暴力法证明，即：

**给实现该算法的函数和RT-thread的对应函数输入所有可能的值，两个函数的返回值结果相同，则说明该算法的结果是正确的**。

为了避免编译错误，我们将要验证的算法的名字改为 `__rt_ffs_new`，查找表的名字改为 `__lowest_bit_bitmap_new`。

编写了如下 C 代码：

```c
#include <stdint.h>
#include <stdio.h>

int main(void)
{
    for(uint32_t i = 0; i< 0xFFFFFFFF; i++)
    {
        if(__rt_ffs(i) != __rt_ffs_new(i))
        {
            //如果出现结果不匹配则报错
            printf("error1 = %u\n", i);
            printf("__rt_ffs(i) = %d\n", __rt_ffs(i));
            printf("__rt_ffs_new(i) = %d\n", __rt_ffs_new(i));
        } 
    }
    
    if(__rt_ffs(0xFFFFFFFF) != __rt_ffs_new(0xFFFFFFFF))
    {
        printf("error2\n");
    }
            
    printf("End of verification\n"); //提示验证结束

  return 0;  
}
```

该代码在 `Windows10` 上运行时得到的结果如下：

```bash
End of verification
```

由最后的输出可知，并没有报错，所以该算法与 RT-`Thread` 内核原查找算法的输出结果是一致的。

由于 `RTOS` 一般是在 `MCU` 中使用，我们在 MCU上 继续进行同样的测试。

新建 `STM32F103C8T6` 的工程，该工程只实现了最简单的运行配置，我们在工程中的 main 函数上进行测试。

测试代码如下：

```C
int main(void)
{
	HAL_Init();
  	SystemClock_Config();
  	MX_GPIO_Init();
  	MX_USART1_UART_Init();

	for(unsigned int i= 0x00; i< 0xFFFFFFFF; i++)
	{
	     if(__rt_ffs(i) != __rt_ffs_new(i)) //报错
            {
            	while(1)
            	{
		    printf("error1 = %u   ", i);
                    printf("__rt_ffs(i) = %d  ", __rt_ffs(i));
                    printf("__rt_ffs_new(i) = %d  ", __rt_ffs_new(i));
                }
            }
	}
    
	if(__rt_ffs(0xFFFFFFFF) != __rt_ffs_new(0xFFFFFFFF)) 
        {
            while(1)
            {
                printf("error2"); //报错
            }
        }
    
        while(1) //匹配结果正确，提示结束
        {
           printf("End of verification");
        };
}
```

输出结果为：

```
End of verification
```

由结果可知，该算法是符合正确性这个指标的。



# 验证算法的高效性

评判一个算法是否高效，通常从 `时间复杂度` 和 `空间复杂度` 这两个方面来分析。

## 更优的空间复杂度

从源代码就可以很清楚的计算出这两个算法的所需要的内存空间，RT-Thread 采用的查找算法使用到的查找表的大小为 `256Byte`，而新的查找算法只需要 `32byte`。

所以，新的算法在空间复杂度上的性能更优，可以将要使用的空间缩小为原来的 `15%` 左右。

同时，新算法的代码实现也更加精简，具体到汇编指令，语句也更加少（具体汇编指令见下文）。

## 更优的时间复杂度

首先，用`大O表示法`，两个算法的时间复杂度都是 `O(1)`。

具体到更细致的对比，这个需要从底层的汇编代码分析，简单来说，谁使用的指令更少且对应指令的执行时间更短，谁的时间复杂度更优。

我们以常用的 `ARM Cortex-M3` 架构为例分析。

先看主线版本的查找算法的汇编指令：

```assembly
  1286: int __rt_ffs(int value) 
  1287: { 
  ;进入函数时，r0用于存储函数的第一个参数
0x08006F78 4601      MOV      r1,r0  ;将寄存器r0的值存储到r1

  1288:     if (value == 0)  
0x08006F7A B909      CBNZ     r1,0x08006F80 ;比较（r1）结果不为零时,跳转到分支(0x08006F80位置)
				return 0;
0x08006F7C 2000      MOVS     r0,#0x00      ;将r0置为0，r0的值默认为函数返回值

  1300: } ;__rt_ffs函数末尾
0x08006F7E 4770      BX       lr  ;函数返回
; lr为连接寄存器R14,用于函数返回。BX用于转移到由寄存器lr给出的地址。

  1290:     if (value & 0xff)
;UXTB是将除最低位字节外的值全部清零
0x08006F80 B2C8      UXTB     r0,r1 	;将r1的值取出，将该值除最低字节外清零的结果赋值给r0
0x08006F82 B120      CBZ      r0,0x08006F8E ; r0的值为0，则跳转到0x08006F8E
  1291:         return __lowest_bit_bitmap[value & 0xff] + 1; 
0x08006F84 B2C8      UXTB     r0,r1 ;将r1的值取出，将该值除最低字节外清零的结果赋值给r0
0x08006F86 4A43      LDR      r2,[pc,#268]  ; 加载r2的值为 @0x08007094地址指向的值 （就是查找表的位置）
0x08006F88 5C10      LDRB     r0,[r2,r0] ;从查找表的r0位置找到对应值，将其赋值给r0
0x08006F8A 1C40      ADDS     r0,r0,#1	 ;r0加1并复制到r0
0x08006F8C E7F7      B        0x08006F7E ;直接到函数末尾

  1293:     if (value & 0xff00) 
0x08006F8E F401407F  AND      r0,r1,#0xFF00 //与操作
0x08006F92 B128      CBZ      r0,0x08006FA0
  1294:         return __lowest_bit_bitmap[(value & 0xff00) >> 8] + 9; 
0x08006F94 483F      LDR      r0,[pc,#252]  ; 将 @0x08007094地址指向的内容赋值给r0
0x08006F96 F3C12207  UBFX     r2,r1,#8,#8 ;从r1寄存器的第8(前面的#8)位开始，提取8(后面的#8)位到r2寄存器，剩余高位用0填充
0x08006F9A 5C80      LDRB     r0,[r0,r2]   ;找到对应位置
0x08006F9C 3009      ADDS     r0,r0,#0x09  ;r0 += 9
0x08006F9E E7EE      B        0x08006F7E

  1296:     if (value & 0xff0000) 
0x08006FA0 F401007F  AND      r0,r1,#0xFF0000
0x08006FA4 B128      CBZ      r0,0x08006FB2
  1297:         return __lowest_bit_bitmap[(value & 0xff0000) >> 16] + 17; 
0x08006FA6 483B      LDR      r0,[pc,#236]  ; @0x08007094
0x08006FA8 F3C14207  UBFX     r2,r1,#16,#8
0x08006FAC 5C80      LDRB     r0,[r0,r2]
0x08006FAE 3011      ADDS     r0,r0,#0x11
0x08006FB0 E7E5      B        0x08006F7E

  1299:     return __lowest_bit_bitmap[(value & 0xff000000) >> 24] + 25; 
0x08006FB2 4838      LDR      r0,[pc,#224]  ;将存储器地址为@0x08007094的字数据读入寄存器r0。
0x08006FB4 EB006011  ADD      r0,r0,r1,LSR #24 ;r0加上r1右移24位后的值赋值给r0,也就是得到查找表的位置
0x08006FB8 7800      LDRB     r0,[r0,#0x00] ;将存储器地址为r0＋0的字节数据读入寄存器r0，并将R0的高24位清零。
0x08006FBA 3019      ADDS     r0,r0,#0x19
0x08006FBC E7DF      B        0x08006F7E

```

精简版本算法实现的汇编指令：

```assembly
127: int __rt_ffs(int value) 
128: { 
0x0800122C 4601      MOV      r1,r0 ;将r0的值（函数参数值）放入r1中
   130:   return __lowest_bit_bitmap_new[(uint32_t)( value & (value - 1) ^ value ) % 37]; 
0x0800122E 1E48      SUBS     r0,r1,#1 ; (value-1)结果放入r0
0x08001230 4008      ANDS     r0,r0,r1 ;(value & (value-1) 结果放入r0
0x08001232 4048      EORS     r0,r0,r1 ; (value & (value-1) ^ value))实现异或的结果放入r0
0x08001234 2225      MOVS     r2,#0x25 ; 将37放入r2
0x08001236 FBB0F3F2  UDIV     r3,r0,r2 ;r3为商
0x0800123A FB020013  MLS      r0,r2,r3,r0; 乘加，r0 = r0 - r2*r3
0x0800123E 4A01      LDR      r2,[pc,#4] ; @0x08001244 ;找到查找表的位置
0x08001240 5C10      LDRB     r0,[r2,r0] ;得到查找表中的值
   131: } 
0x08001242 4770      BX       lr ;函数返回
```

由代码数量来说，对比之下，新的算法的实现就非常简洁，需要的汇编指令明显减少了很多。

但是由于新的算法存在除法等指令，这些指令的执行周期跟 MCU 的架构等有关，一般执行的时间比一般的位运算更长。功能强大的芯片执行除法等指令的效率也越高。

实际跑跑才能得出真正的结果，下面同样以常见的且性能相对一般的 `STM32F103C8T6` 来验证，让两个算法在该芯片上比比，看谁在执行一样的任务下，谁使用的时间更少，那么它就是更优的。

给 `STM32F103C8T6` 建立一个简单的工程，系统时钟频率为 `72MHz`，该工程配置了`UART`（用来发送信息方便解读）、`Systick 1ms` 中断（用来给执行任务计时）。

然后代码的整体架构就是一个简单的循环语句，通过查看执行时间得到两个算法的运行效率孰优孰劣。

#### 先测量 `0x00 - 0x0FFFFFFF` 

代码如下：

```c
int main(void)
{
  	HAL_Init();
  	SystemClock_Config();
  	MX_GPIO_Init();
	MX_USART1_UART_Init();
    
    printf("Start time = %dms\n",HAL_GetTick()); 
    for(uint32_t i= 0; i<= 0x0FFFFFFF; i++)
    {
        __rt_ffs(i);
    }
     
    int buf = HAL_GetTick();
    while(1)
    {
        printf("End time = %dms",buf);
    }
}
```

使用主线版本的时间

> Start time = 0ms
>
> End time = 141982ms

使用新版本的算法 

>Start time = 0ms
>
>End time = 138299ms

#### 对输入所有可能的结果进行测量（`0x00 - 0xFFFFFFFF` ）

```C
int main(void)
{
  	HAL_Init();
  	SystemClock_Config();
  	MX_GPIO_Init();
	MX_USART1_UART_Init();

	printf("Start time = %dms\n",HAL_GetTick()); 
    for(uint32_t i= 0; i< 0xFFFFFFFF; i++)
    {
        __rt_ffs(i);
    }
     
    __rt_ffs_new(0xFFFFFFFF);
    int buf = HAL_GetTick();
    while(1)
    {
        printf("End time = %dms",buf);
    }
}
```

主线版本

>Start time = 0ms
>
>End time = 2510470ms

精简版本

>Start time = 0ms
>
>End time = 2272475ms



从上述结果中，我们可以知道，精简版本的查找算法执行时间更短一些。

具体到 `STM32F103` 芯片，每次查找能够比主线版本的查找算法缩短: `（2510470ms - 2272475ms) / (2^32) = 55.412ns`。

按百分比算，快了近 `55.412ns / （2510470ms /  (2^32)）=  55.412ns / 584.514ns = 9.48%`

# 总结

综上所述，通过目前的验证（可能验证的方式存在瑕疵），精简版本的算法无论是在时间效率还是空间效率上，都比主线版本好一些。



# 相关讨论




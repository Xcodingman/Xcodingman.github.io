---
title: FreeRTOS Implementation
catalog: true
date: 2020.9.13
subtitle: 
header-img: 
top: false
tags: OS
categories: MCU 
---

# Code Architecture

<img style="display: block; margin: 0 auto;" src="images/code_architecture.jpg" alt="" />
FreeRTOS的代码可以分为两个部分，一个是与硬件无关的代码，一般放在source目录下，是用户程序可以调用的代码。其中，最为主要的是task.c,list.c和queue.c三个文件
<img style="display: block; margin: 0 auto;" src="images/source_diretory.jpg" alt="" />

## list.c
定义双向链表和列表项的结构，以及操作链表的函数，列表和列表项的结构如下
<img style="display: block; margin: 0 auto;" src="images/list_list_structure.jpg" alt="" />
<img style="display: block; margin: 0 auto;" src="images/list_item_structure.jpg" alt="" />

## tasks.c
tasks.c 主要包括task的结构定义以及用户可以调用的task操作
### TCB
<img style="display: block; margin: 0 auto;" src="images/TCB.jpg"
alt="" />
*ListItem  State*: task状态(Ready，blocked，suspended，running) \      
*ListItem  Event* \
*Priority* \
*pxStack* \
*TaskName

## Queue.c
用于进程间通信

# TASK
## task functions
`void ATaskFunction( void *pvParameters ); `

### 每个task含有单独stack
```
void ATaskFunction( void *pvParameters ) { /* Variables can be declared just as per a normal function.  Each instance of a task created using this example function will have its own copy of the lVariableExample variable.  This would not be true if the variable was declared static – in which case only one copy of the variable would exist, and this copy would be shared by each created instance of the task. (The prefixes added to variable names are described in section 1.5, Data Types and Coding Style Guide.) */ 

int32_t lVariableExample = 0; 
 
/* A task will normally be implemented as an infinite loop. */     

for( ;; ) {/* The code to implement the task functionality will go here. */     

} 
 
/* Should the task implementation ever break out of the above loop, then the task      must be deleted before reaching the end of its implementing function.  The NULL      parameter passed to the vTaskDelete() API function indicates that the task to be      deleted is the calling (this) task.  The convention used to name API functions is      described in section 0, Projects that use a FreeRTOS version older than V9.0.0 


vTaskDelete( NULL );
```

## 创建 task
```
BaseType_t xTaskCreate( TaskFunction_t pvTaskCode, const char * const pcName,    uint16_t usStackDepth,void *pvParameters,UBaseType_t uxPriority,
TaskHandle_t *pxCreatedTask );
```
**pvTaskCode**      function name\
**pcName**          only for debug use,freertos does not use it\
**usStackDepth**    specify the size of stack\
**pvParameters**     function parameters


## state transition
### blocked state
**Tasks可以通过两种类别的event事件进入blocked模式**\
**Temporal (time-related) events**：Task等待一段固定的延时，比如taskDelay\
**Synchronization events**：事件起源于其他task或者中断，比如IO等待数据
### The Suspended State
suspended state 由应用程序控制（与中断无关）
### The Ready State
### the Running State
![avatar](./images/state_transition.jpg)


###  以下两个函数会将当前任务放到blocked列表中
`vTaskDelayUntil()`   : run task at a fixed frequency\
` vTaskDelay(). `



## idle task
### 当scheduler启动时，会启动idle task  
`vTaskStartScheduler()`
###  Idle task 负责清理被删除的task
### Idle task hook 
用户可以定义idle task的应用程序，一般用作低功耗模式\
***priority*** alwasy 0

**configIDLE_SHOULD_YIELD** 

## Scheduling Algorithms 
### Round Robin Scheduling
Fixed Priority **Pre-emptive Scheduling** with Time Slicing\
configUSE_PREEMPTION =1 \
configUSE_TIME_SLICING =1 \
**pre-emptive:** pre-emptive the lower priority task at each end of a time slice

### Co-operative Scheduling
`configUSE_PREEMPTION =0`\
`configUSE_TIME_SLICING = Any Value`

# scheduler

Copyright (c) 2016 Corerain Technologies. All rights reserved.  
No part of this document, either material or conceptual may be 
copied or distributed, transmitted, transcribed, stored in a retrieval 
system or translated into any human or computer language in
any form by any means, electronic, mechanical, manual
or otherwise, or disclosed to third parties without
the express written permission of Corerain Technologies. 
  
# bias设计规格

## 概述
这个模块主要功能是把输入的不同的filter下卷积的结果叠加不同的offset，并且根据bias_en控制，选择是否增加bias输出或者bypass输出。

key | value 
--- | --- 
Top level name : | bias 
Version : |	2.0.0
GitHub Path GitHub : | https://github.com/corerain/rainman/tree/master/engine/conv/hdl
Design Owner : | yanxu.yang@corerain.com
Verification by : | yanxu.yang@corerain.com


如下时bias模块内部结构图。
![bias Structure](bias_architecture.png)

## 用例

经过filter卷积之后的图像为了能够更好的被分割区分，需要增加一个bias偏移，使图像能够更好的达到预期的分类值，不同的filter需要不同的bias值来提供偏移，
所以bias模块主要功能是为filter之后的数据叠加一个偏移量。

系统侧直接通过dma总线输出bias参数到bias模块，并且通过apb接口配置bias_en控制寄存器，实现控制bias模块bypass输出或加bias输出，bias模块内置加法器
和选择器，输入的像素数据到来时，读出缓存的bias数据与之累加，然后通过选择器控制输出，从而实现系统测配置bias参数及配置功能开关等。

![bias Integration](bias_integration.png)

## 输入输出信号

|Name|Signal|Width|Description|
| --- | --- | --- | --- |
| bias_vld | input | 1 | 输入bias参数有效信号 |
| bias_data | input | dma_width | 输入的bias参数 |
| up_vld | input | 1 | 输入数据有效信号 |
| up_data | input | pv*data_width | 按filter依次输出的图像数据 |
| dn_vld | output | 1 | 输出数据的有效信号 |
| dn_data | output | pv*data_width | 输出的加bias或bypass图像数据 |

## 内部信号

|Name|Width|Description|
| --- | --- | --- |
| up_dat_pixel_counter | data_width | 输入的图像像素点计数 |
| dn_bias_rdy | 1 | 控制bias_fifo读是能有效准备信号，有效时bias_fifo读出一个bias参数 |
| A_dat | data_width | 加法器的A端输入数据，等于bias模块输入的像素数据 |
| B_dat | data_width | 加法器的B端输入数据，等于从bias_fifo读出的bias参数 |
| S_dat | data_width | 输入的像素值与bias偏移叠加后的数据 |
| mux_dn_dat | data_width | 输入的原始像素值与叠加完bias后的像素值经过bias_mux选择之后输出的像素值，bias_en=1时输出s_dat，bias_en=0时输出up_data |

## 参数
| parameter | defaul | function |
| --- | --- | --- | 
| dma_width  | 32 | dma数据位宽 	  		|
| data_width | 32 | 像素数据位宽	  		|
| k			 | 3  | kernel 位宽 	  		|
| pv  		 | 2  | 像素点传输的并行数据个数|

## 功能

bias模块首先将接收的bias参数数据缓存在bias_fifo中，当输入的第一个filter的像素数据到来时，读出fifo中的一个filter的bias数据与像素数据进行叠加，
后续filter数据到来时依次读取不同filter的bias参数与之叠加，然后把叠加后的结果和原始输入数据输出给mux_bias模块进行选择输出，bias_en=1时，输出叠加结果数据，bias_en=0时，
输出原始像素数据，选择输出之后的数据经过一个fifo缓存后传输给下一级模块。

* bias参数必须在像素数据到来之前完成输入缓存，并且需要把所有filter数据从低到高依次输入到缓存区。
* 当一个filter数据接收完成后，需要立马读出下一个filter的参数为下一个filter像素叠加做准备，即dn_bias_rdy信号在up_vld最后一个时钟周期有效。
* 加法器的A、B两端必须同时有效输入filter像素数据与该filter的对应的bias值。
* 通过bias_en来控制选择器输出结果，bias_en=0时，bias模块输出原始输入像素数据，bias_en=1时，模块输出增加bias之后的数据。

可验证功能列表：

**FN** feature name 功能名称  
**FD** feature description 功能说明  
**ID** input data 输入数据  
**IS** initial (internal) state 初始（内部）状态  
**CS** control sequence 控制序列  
**FS** finial (internal) state 最终（内部）状态  
**OD** output data 输出数据  
**ES** external status observability 外部状态可观察性 

**FN** feature name 功能名称  
**FD** feature description 功能说明  
**ID** input data 输入数据  
**IS** initial (internal) state 初始（内部）状态  
**CS** control sequence 控制序列  
**FS** finial (internal) state 最终（内部）状态  
**OD** output data 输出数据  
**ES** external status observability 外部状态可观察性

## 性能

### 带宽
对于pv=8,data_width=64,100M时钟情况下，单个filter输入的最大数据带宽为8*64*100Mbps=51.2Gbps。
### 延迟
输入的数据经过bias_add模块有一个时钟周期延时，由于bias_mux模块时组合逻辑，没有时钟延时，所以当bias_en=1时，数据输出会有一个时钟延时，
当bias_en=0时，数据没有时钟延时，mux选择之后的数据传输给一级sif_fifo输出，sif_fifo有两个时钟周期延时，所以模块总的延时子啊bias_en=1的有3个时钟周期，
bias_en=0时有两个时钟周期。
### 启动时间
由于模块使用异步复位，所以模块的启动时间就等于复位时间;

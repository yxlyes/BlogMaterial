---
title: SCMI(System Control and Management Interface)
date: 2024-1-7
---

[toc]

# 介绍

SCMI是一套**独立于操作系统**的**软件接口**，用于系统管理。SCMI是**可扩展的**，目前提供了以下接口：

- 所支持的接口的**发现**以及**接口的描述**。
- **Power domain管理**。将给定的device或domain设定到支持的各种power-saving state。
- **Performance管理**。控制不同计算引擎(CPU、GPU等)domain的performance。
- **Clock管理**。设置或者查看平台所支持的clocks。
- **Sensor管理**。读sensor的数据以及当数据改变时收到通知。
- **Reset domain管理**。将给定的device或domain设定到各种reset state。
- **Voltage domain管理**。配置和管理组件的电压。
- **Power capping and monitoring**。配置和调整power caps并且监控power capping domains的power消耗。
- **Pin control**。控制和配置pins。

:question: 为什么要出现SCMI

> 提供微控制器(microcontrollers)来将power或系统管理任务从APs中抽离出来是工业界的大势所趋。这些控制器通常通常有相似的接口，无论是在它们提供的功能方面，还是在请求如何与它们通信方面。通过SCMI接口可以将任务处理从APs转移到控制器上。

SCMI所定义的接口提供了两个抽象层次：

* 协议(Protocol)

  每组功能相关的函数被归类成一个协议。因为SCMI接口是可扩展的，所以协议也会增加。

* 传输(Transport)

  协议通过传输来进行通信。传输规范描述了agent和platform component之间如何进行通信，其中agent是协议的使用者，platform component对协议信息进行处理。

所使用的接口在firmware(FDT或ACPI)中进行描述。

# SCMI结构

SCMI旨在允许agents管理硬件platform提供的各种功能，比如功耗和性能的管理。

:label: 名词解释

> **Protocols**: 不同的协议分别定义了一组不同的系统控制和管理。
>
> **Transport**: 描述了Agent和Platform之间通信的方法。
>
> Agent: 系统控制和管理的调用者(caller)。
>
> Platform: 硬件组件的集合，用来对通信信息进行解析并且执行相应的功能(callee)。
>
> Resource: 硬件平台的组件，可以通过SCMI信息来被控制。

每个transport可以提供多个channels。每个agent在和platform通信时必须有一套专属的channels。换而言之，不同的agent之间不能共享channels。此要求消除了在运行完全不同软件堆栈的agent之间创建锁原语的需要。

下面图1展示了使用SCMI接口的示例系统。在此示例中，Platform包括了一个 SCP，用来处理从AP发出的命令。AP和SCP进行通信使用的channel分别Secure/root和Non-Secure channel。

![image-20240108153805315](C:\Users\xinglong.yang\AppData\Roaming\Typora\typora-user-images\image-20240108153805315.png)

<center>图1 SCMI 概述</center>



:grey_question: None-secure和Secure/Root channel的区别是什么？

> 使用ARM TrustZone技术可以拥有Secure和None-secure channel。Agent可以处于Secure和None-Secure state。Non-Secure channel不可以访问或修改Secure Platform resources。Secure channel只能被处于Secure state的agent使用。
>
> ARM RME(Realm Management Extension)引入了两种新的security states: Realm和Root。除了已经有的Secure和Non-secure PAS(Physical Address Spaces)，使用SCMI的系统增加了Root PAS。表1说明了SCMI agents，channel和resource的访问规则。

<Table>
    <tr>
        <th rowspan="2">Agent Security State
        </th>
        <th colspan="3">SCMI Resource Assignment and Channel PAS</th>
    </tr>
	<tr>
        <th>Non-Secure</th><th>Secure</th><th>Root</th>
	</tr>
	<tr>
        <th>None-Secure</th>
        <td>Allow</td>
        <td>Deny</td>
        <td>Deny</td>
	</tr>
	<tr>
        <th>Secure</th>
        <td>Allow</td>
        <td>Allow</td>
        <td>Deny</td>
	</tr>
	<tr>
        <th>Root</th>
        <td>Allow</td>
        <td>Allow</td>
        <td>Allow</td>
	</tr>
</table>

<center>表1 SCMI Channel and Resource Access Rules</center>

# 协议(Protocols)

如前文描述，协议是一组消息。下面将分别描述信息通信流程和信息的结构。

## 通信流程

Agents和Platform之间通过transport channel进行通信。Channel分为两种类型：SCMI Fastchannel和标准的SCMI channel。

* Fastchannel：是一个轻量级单向channel，专用于控制特定的Platform resource且拥有特定的SCMI信息类型。即一个Fastchannel只能作用于特定的一个任务。不需要提供多余的header，因此latency较低。
* standard channel：用于承载多种消息类型，且可以显式的控制多个Platform resource。需要header来声明信息类型。

下面的图2描述了agents和platform之间如何通过channel进行通信。

![image-20240108200445398](C:\Users\xinglong.yang\AppData\Roaming\Typora\typora-user-images\image-20240108200445398.png)

<center>图2 Message and Channels</center>

每个Agent都有专属的channels，用来**发送信息**或者从platform**接受信息**。除了Fastchannel，每个channel都是双向通信的。根据方向不一样，channel可以分为两种类型：

* A2P(Agent to Platform)channel，agent是请求者。
* P2A(Platform to Agent)channel，platform是请求者。

Transport支持两种通信方式：

* interrupt-driven communication：当完成任务时，通过中断来表明任务已经完成。
* polling-based communications：需要请求方通过polling的方式来判断任务是否完成。

Agent向Platform发的命令分别两种类型：

* 同步

  命令始终block channel，直到被请求的工作完成。Platform通过A2P channel进行回应。所以channel只有在上次任务完成后才可以进行下一次任务。

* 异步

  对于异步请求，platform对于请求的任务会稍后完成。因此，platform会立马对caller进行回应并free channel，使得下条命令可以发送。当异步任务完成后，platform通过P2Achannel发送类型为dealyed response的回应消息。

Platform通过P2A channel发送给agent的信息分别两种类型：

* Delayed response

  表明完成了异步通信的任务。

* Notifications

  当platform发生了某些事件后，可以主动向agent发送notification。

Fastchannel不支持异步请求，delayed response和notification。

## 信息(Message)的格式

Message类似于远程过程调用(Remote procedure call)，因此必须可代表所请求的操作及其任何参数或返回值。

每个信息都携带了message header，用于标识所请求的操作。每条信息属于一个协议，因此header中包含了8-bit的协议标识，称为protocol_id。同时每条信息有一个唯一的8-bit的表示，称为message_id。

每个信息可有32-bit的参数以及32-bit的返回值。

protocol_id值的描述如下表2所示。

| protocol_id | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| 0x0 - 0xF   | Reserved                                                     |
| 0x10        | Base protocol                                                |
| 0x11        | Power domain management protocol                             |
| 0x12        | System power management protocol                             |
| 0x13        | Performance domain management protocol                       |
| 0x14        | Clock management protocol                                    |
| 0x15        | Sensor management protocol                                   |
| 0x16        | Reset domain management protocol                             |
| 0x17        | Voltage domain management protocol                           |
| 0x18        | Power capping and monitoring protocol                        |
| 0x19        | Pin Control protocol                                         |
| 0x1A-0x7F   | Reserved for future use by this specification                |
| 0x80-0xFF   | Reserved for vendor or platform-specific extensions to this interface |

<center>表2 Protocol identifiers</center>

对所有使用标准channel的protocol和transports，Message通过32-bit的message header发送给platform。header格式如下表3所示。Fastchannel不适用message header，因为它只作用于独特的信息类型。

| Field       | Mnemonic     | Description                                            |
| ----------- | ------------ | ------------------------------------------------------ |
| Bits[31:28] | -            | Reserved, must be zero.                                |
| Bits[27:18] | token        | Token.                                                 |
| Bits[17:10] | protocol_id  | Protocol identifier.                                   |
| Bits[9:8]   | message_type | Message type. Value of 0x1 is reserved for future use. |
| Bits[7:0]   | message_id   | Message identifier.                                    |

Message type分为三类：

* Commands， message type = 0。
* Delayed response，message type = 2.
* Notifications，message type = 3.

## Example for Commands

CLOCK_TARE_GET，用于获取某个clk的频率。

<table>
    <tr>
        <td colspan="2">message_id: 0x6 protocol_id: 0x14</td>
    </tr>
    <tr>
        <th>Parameters Name</td>
        <th>Description</th>
    </tr>
	<tr>
        <td>uint32 clock_id</td>
        <td>Identifier for the clock device.</td>
	</tr>
	<tr>
        <th>Return values Name</td>
        <th>Description</th>
    </tr>
	<tr>
        <td>int32 status</td>
        <td>SUCESS if the current clock rate was successfully returned.</td>
	</tr>
	<tr>
        <td>uint32 rate[2]</td>
        <td>Lower 32 bits and Upper 32bits of the physical rate in Hertz</td>
	</tr>
</table>



# 传输(Transport)

Transport描述了agent和platform之间如何进行通信。

## 基于shared memory的传输

这种形式的传输依赖于在平台和代理之间使用shared memory。即支持基于中断的通信，还支持基于polling的传输。每个channel包含：

* shared memory area

  这是caller和callee之间共享的shared memory。shared memory被caller或callee所有。所有权由shared memory中的**channel status**字段决定。当shared memory被caller所有时，channel处于free状态；被callee所有时，channel处于busy状态。

  当channel处于free时，caller可以向shared memory写信息以及相关的payload。然后caller更新channel status状态，将channel改为busy状态，此时，shared memory的所有权归callee。callee此时可以使用shared memory来填写请求的return value。当callee完成任务后，将channel重新设为free状态并通知caller任务已经完成，可以发送新的请求。

* Doorbell

  这是caller用于提醒callee消息存在的一种机制。

* Completion interrupts

  该传输支持polling和interrupt driven两种不同的通信模式。在interrupt driven模式下，当callee完成对消息的处理时，它会向caller发出中断。

### Interrupt-driven Communications flow

![image-20240108223940381](C:\Users\xinglong.yang\AppData\Roaming\Typora\typora-user-images\image-20240108223940381.png)

### Polling based Communication Flow

![image-20240108223955897](C:\Users\xinglong.yang\AppData\Roaming\Typora\typora-user-images\image-20240108223955897.png)

## 基于ACPI的传输

略。详见SCMI spec.

## 基于shared memory或MMIO的Fastchannel传输

略。详见SCMI spec.

## Virtio-SCMI

略。详见SCMI spec.

# 经典问题

:question: 当callee将channel set free，但此时caller对该请求出现了超时，此时caller开始发送下一条命令，此时caller写shared memory覆盖了callee的答复，在覆盖之后caller收到了callee的完成中断，此时将出现overlay问题。

问题详细描述已经解见patch: [**firmware: arm_scmi: Check mailbox/SMT channel for consistency**](https://git.kernel.org/pub/scm/linux/kernel/git/sudeep.holla/linux.git/commit/?h=for-linux-next&id=0904a71cb58bfc8f2cc542c896ea81cee5e6dfc2)

# 参考

[Arm System Control and Management Interface](https://developer.arm.com/documentation/den0056/e/?lang=en)

[drivers/firmware/arm_scmi](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/firmware/arm_scmi)
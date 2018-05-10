<!-- GFM-TOC -->
* [分布式事务介绍](#分布式事务介绍)
    * [1. 分布式事务简介](#1-分布式事务简介)
    * [2. 应用场景](#2-应用场景)
      * [2.1 假设没有分布式事务](#21-假设没有分布式事务)
    * [3. 分布式事务的实现](#3-分布式事务的实现)
      * [3.1 XA二阶段提交协议](#31-xa二阶段提交协议)
        * [3.1.1 魔兽世界组队模式](#311-魔兽世界组队模式)
        * [3.1.2 二阶段提交协议](#312-二阶段提交协议)
          * [3.1.2.1 二阶段提交协议的算法思想](#3121-二阶段提交协议的算法思想)
          * [3.1.2.2 2PC正向流程](#3122-2pc成功流程)
          * [3.1.2.3 2PC逆向流程](#3123-2pc失败流程)
          * [3.1.2.4 XA两阶段提交的不足](#3124-xa两阶段提交的不足) 
      * [3.2 XA三阶段提交协议](#32-xa三阶段提交协议)
        * [3.2.1 相比2PC的改动](#321-相比2pc的改动)
          * [3.2.2.1 CanCommit阶段](#3221-cancommit阶段)
          * [3.2.2.2 PreCommit阶段](#3222-precommit阶段)
          * [3.2.2.3 DoCommit阶段](#3223-docommit阶段)
<!-- GFM-TOC -->

# 分布式事务介绍
## 1. 分布式事务简介
```
  分布式事务用于在分布式系统中保证不同节点之间的数据一致性。
  指事务的操作位于不同的节点上，需要保证事务的 AICD 特性。
```
 - 产生原因
   - 数据库分库分表
   - SOA 架构，比如一个电商网站将订单业务和库存业务分离出来放到不同的节点上;
## 2. 应用场景
### 2.1 假设没有分布式事务
```
  我们以电商交易业务为例子:
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp5gfMpBib3Vm2IsPAICBYZCpyRYV1X8KXicVPKibeyjfqMUZzpFCkC6GXM528Kic56xliagulfmuUANzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
 上图中包含了库存和订单两个独立的微服务，每个微服务维护了自己的数据库。
 在交易系统的业务逻辑中，一个商品在下单之前需要先调用库存服务，进行扣除库存，再调用订单服务，创建订单记录。
```
```
  正常情况下，两个数据库各自更新成功，两边数据维持着一致性。
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp5gfMpBib3Vm2IsPAICBYZCJ9n7WP2dVhskCIpB7J2I9OWII0YDDfOC62W205W6xHglkGFLWmDHhg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
但是，在非正常情况下，有可能库存的扣减完成了，随后的订单记录却因为某些原因插入失败。
这个时候，两边数据就失去了应有的一致性。
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp5gfMpBib3Vm2IsPAICBYZCf0tkbqJsM1iakUQFYy48KWWvUovlAb4G1UgGaD8LMhxwViaxGPckWVzg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  单数据源的一致性依靠单机事务来保证,多数据源的一致性就要依靠分布式事务;
```
## 3. 分布式事务的实现
```
  分布式事务的实现有很多种，最具有代表性的是由Oracle Tuxedo系统提出的XA分布式事务协议;
```
```
  XA协议包含两阶段提交（2PC）和三阶段提交（3PC）两种实现,
  这里我们重点介绍两阶段提交的具体过程;
```
### 3.1 XA二阶段提交协议
#### 3.1.1 魔兽世界组队模式
```
 在魔兽世界这款游戏中，副本组团打BOSS的时候，为了更方便队长与队员们之间的协作，
 队长可以发起一个“就位确认”的操作：
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofVRL815UNR3mnXpYf81U5Lv5WtNiamohdu792UPtCuHhNLkg7FGMvicFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  当队员收到就位确认提示后，如果已经就位，就选择“是”，如果还没就位，就选择“否”。
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofqPopLXT6ALzBz6elibzNxT8XoQSaEgXdJjYuRbkKV65HtVDLFibeWvVw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  当队长收到了所有人的就位确认，就会向所有队员们发布消息，告诉他们开始打BOSS。
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofgsicDP1lt3zSlNj0DNgFaf1o5F0uOn6oJd5sngZzqZy01ZVXBBcSQ4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  相应的，在队长发起就位确认的时候，有可能某些队员还并没有就位
```
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofVRL815UNR3mnXpYf81U5Lv5WtNiamohdu792UPtCuHhNLkg7FGMvicFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofYaNSnxNnZfsXwxhdicfIrx0bD8BY5GiaVBqxphFcdsuJgrdPX1iaetuOg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofqxmWafL4tcIcMFcHhAcR1AX3QvS9Fw5JCC0dPTOYvtlUSJic4uibuZHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  以上就是魔兽世界当中组团打BOSS的确认流程。这个流程和XA分布式事务协议的两阶段提交非常相似。
    打魔兽,要么一起上,要么都不上!!!
```
#### 3.1.2 二阶段提交协议
##### 3.1.2.1 二阶段提交协议的算法思想
```
     在XA协议中包含着两个角色：事务协调者(部落首领)和事务参与者(部落小兵)。
     
     二阶段(Two-phaseCommit,简称2PC)提交协议的算法思想概括:
         参与者将操作成败通知协调者,再有协调者根据所有参与者的反馈情况决定各参与者是否要提交操作还是终止操作;
```
   - **2PC提交协议**:
   主要保证了分布式事务的原子性,即所有节点要么全做,要么全不做;  
##### 3.1.2.2 2PC正向流程
  - 第一阶段
  
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofektAk1LqqTkgjlFicuYE55XHon5yUguGBSk97Ec7vY62wTibVia7iaTNvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
  在XA分布式事务的第一阶段，作为事务协调者的节点会首先向所有的参与者节点发送Prepare请求。
  
```
```
 在接到Prepare请求之后，每一个参与者节点会各自执行与事务有关的数据更新，写入Undo Log和Redo Log。
 如果参与者执行成功，暂时不提交事务，而是向事务协调节点返回“完成”消息。
 当事务协调者接到了所有参与者的返回消息，整个分布式事务将会进入第二阶段。
```
 - 第二阶段
 
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqof9zeDNDYh1qjyYTo9ib4wVCu2KrtqIyJBffhkAvLNybmibEMiaSoKGqFKg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
```
 在XA分布式事务的第二阶段，如果事务协调节点在之前所收到都是正向返回，那么它将会向所有事务参与者发出Commit请求;
```
```
  接到Commit请求之后，事务参与者节点会各自进行本地的事务提交，并释放锁资源。
  当本地事务完成提交后，将会向事务协调者返回“完成”消息。
  当事务协调者接收到所有事务参与者的“完成”反馈，整个分布式事务完成。
```
```
  以上所描述的是XA两阶段提交的正向流程，接下来我们看一看失败情况的处理流程：
```
##### 3.1.2.3 2PC逆向流程
 - 第一阶段:
  
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofr3Qjn25OskkZ0Hd1ibMicWpQgTJShGSyAsthibicgNeZHUOx5Sy2Mlwsrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)
 
 - 第二阶段:
 
![](http://mmbiz.qpic.cn/mmbiz_png/NtO5sialJZGp2Ny2lbXKGqaNjy4cbhqofMklXcDS3cVJdWjw4vgibtBiaolQia9NMsT4ibMiaJyHPwwNjr9Db7ljEBug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

```
  在XA的第一阶段，如果某个事务参与者反馈失败消息，说明该节点的本地事务执行不成功，必须回滚。
```
```
  在XA的第一阶段，如果某个事务参与者反馈失败消息，说明该节点的本地事务执行不成功，必须回滚。
  于是在第二阶段，事务协调节点向所有的事务参与者发送Abort请求。接收到Abort请求之后，
  各个事务参与者节点需要在本地进行事务的回滚操作，回滚操作依照Undo Log来进行。
  以上就是XA两阶段提交协议的详细过程。
```
##### 3.1.2.4 XA两阶段提交的不足
```
  2PC提交协议确实可以保证原子性操作,但是存在很多缺点:
```
 - **同步阻塞问题**: 执行过程中，所有参与节点都是事务阻塞型的。当参与者占有公共资源时，其他第三方节点访问公共资源不得不处于阻塞状态。
 - **单点故障**:
 由于协调者的重要性，一旦协调者发生故障。参与者会一直阻塞下去。尤其在第二阶段，协调者发生故障，那么所有的参与者还都处于锁定事务资源的状态中，而无法继续完成事务操作。（如果是协调者挂掉，可以重新选举一个协调者，但是无法解决因为协调者宕机导致的参与者处于阻塞状态的问题）
 - **数据不一致**:在二阶段提交的阶段二中，当协调者向参与者发送commit请求之后，发生了局部网络异常或者在发送commit请求过程中协调者发生了故障，这回导致只有一部分参与者接受到了commit请求。而在这部分参与者接到commit请求之后就会执行commit操作。但是其他部分未接到commit请求的机器则无法执行事务提交。于是整个分布式系统便出现了数据部一致性的现象;
 - **事务状态不明确**:
 协调者再发出commit消息之后宕机，而唯一接收到这条消息的参与者同时也宕机了。那么即使协调者通过选举协议产生了新的协调者，这条事务的状态也是不确定的，没人知道事务是否被已经提交。

### 3.2 XA三阶段提交协议
#### 3.2.1 相比2PC的改动
 - 引入超时机制
```
  在协调者和参与者中同时引入超时机制;
```
 - 添加PreCommit阶段
```
  在第一阶段和第二阶段中插入一个准备阶段.保证了在最后提交阶段之前各参与节点的状态是一致的;
```
#### 3.2.2 3PC阶段分析
##### 3.2.2.1 CanCommit阶段
```
  3PC的PreCommit阶段其实和2PC的准备阶段相似.协调者向参与者发送commit请求,参与者如果可以提交就返回Yes响应,否则返回No响应;
```
  - 事务询问
```
  协调者向参与者发送CanCommit请求(可以理解为是否可以提交的意愿),询问是否可以执行事务提交操作,然后开始等待参与者的响应;
```
  - 响应反馈
```
  参与者接收到CanCommit请求之后,正常情况下,如果其自身认为可以顺利执行事务,则返回Yes响应,并进入预备状态,否则反馈No;
```
##### 3.2.2.2 PreCommit阶段
```
  协调者根据参与者的反应情况来决定是否可以记性事务的PreCommit操作.根据响应情况,有以下两种可能。
```
  - 1.假如协调者从所有的参与者获得的反馈都是Yes响应,那么会执行事务的预执行.
```
  [1]发送预提交请求: 协调者向参与者发送PreCommit请求,并进入Prepared阶段;
  [2]事务预提交:参与者接收到PreCommit请求之后,会执行事务操作,并将undo和redo信息记录到事务日志中;
  [3]响应反馈:如果参与者成功地执行了事务操作,则返回ACK响应,同时开始等待最终命令;
```
  - 2.假如有任何一个参与者向协调者发送了No响应,或者等待超时之后,协调者都没有接到参与者的响应,那么就执行事务的中断
```
  [1]发送中断请求:协调者向所有参与者发送abort请求;
  [2]中断事务:参与者收到来自协调者的abort请求之后(或超时之后,仍未收到协调者的请求),执行事务的中断;
```
##### 3.2.2.3 DoCommit阶段
该阶段进行真正的事务提交,也可以分为以下两种情况。
 - 1.**执行提交**
```
  [1].发送提交请求:协调接收到参与者发送的ACK响应,那么他将从预提交状态进入到提交状态;
  [2]事务提交:参与者接收到doCommit请求之后,执行正式的事务提交.并在完成事务之后释放所有事务资源.
  [3]响应反馈:事务提交之后,向协调者发送ACK响应;
  [4]完成事务:协调者接收到所有参与者的ACK响应之后,完成事务;
```
 - 2.**中断事务**
  协调者没有接收到参与者发送的ACK请求(可能是接受者发送的不是ACK响应,也可能响应超时),那么就会执行中断事务;
```
  [1]发送中断请求:协调者向所有参与者发送abort请求;
  
  [2](TODO)事务回滚:参与者接收到abort请求之后,利用其在阶段二记录的undo信息来执行事务的回滚操作,并在完成回滚之后释放所有的事务资源;
  
  [3]反馈结果:参与者完成事务回滚之后,向协调者发送ACK消息;
  [4]中断事务:协调者接收到参与者反馈的ACK消息之后,执行消息的中断;
```
```
  在doCommit阶段,如果参与者无法及时接收到来自协调者的doCommit或者rebort请求时,会等待超时之后,会继续进行事务的提交。
  (其实这个应该是基于概率来决定的,当进入第三阶段时,说明参与者在第二阶段已经收到了PreCommit请求,
  那么协调者产生PreCommit请求的前提条件是他在第二阶段开始之前,收到所有参与者的CanCommit响应都是Yes.
  (一旦参与者收到了PreCommit,意味着他知道大家都同意修改了)所以,一句话概括,当进入第三阶段时,由于网络超时等原因,虽然参与者没有收到commit或者abort响应,但是他有理由相信:成功提交的几率很大.)
```




---
title: paxos算法总结
date: 2019-07-31 18:58:01
tag: paxos
layout: post
---
## 资料

* http://www.leviathan.vip/2018/05/24/Paxos%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0/
* https://www.the-paper-trail.org/post/2009-02-03-consensus-protocols-paxos/
* http://rystsov.info/2015/09/16/how-paxos-works.html

## paxos作用
paxos帮助系统在出现网络错误（partition-tolerance）和节点故障（avaliabbilty）能够继续工作。能够阻止在不同的客户端看到矛盾的结果（一致性）。

**与CAP定理矛盾？**

并不矛盾，CAP中的A是很严格的，从CAP的角度来看，两阶段提交存在单点故障，Paxos的系统系统也是unavailable。Paxos的系统是CP系统，之不够其能容忍n个节点失败。

## 关键点

而Paxos只讨论了单个Var的共识。 Paxos 算法解决的问题是一个分布式系统如何就某个值（决议）达成一致。

**角色**

一个节点可能不只担任一个角色

* Proposers 提案提交者
* Acceptors 提案接受者
* Learners 广播确定的提案

Safety前提

* 一个Var只有一个Value
* 只有被确定的Value才会被Learn

**为什么要大多数节点同意**

> any two majority sets of acceptors will have at least one acceptor in common. Therefore if two proposals are agreed to by a majority, there must be at least one acceptor that agreed to both. This means that when another proposal, for example, is made, a third majority is guaranteed to include either the acceptor that saw both previous proposals or two acceptors that saw one each.


**为什么paxos决定的值是合法的value**

考虑如下的场景：

> Consider the case where a proposer has committed his proposal to the smallest possible majority of acceptors, at which point a single acceptor fails. A majority of accept confirmation messages will not reach the proposer, and therefore the protocol will no terminate. A second proposer might then try to propose a value - which is accepted by a majority since the proposer orders its request after the first. The second proposer then commits its proposal, and a majority of acceptors respond. The second proposer considers the protocol completed. At this point, the failed acceptor can recover and send the final accept message to the original proposer, which then considers the protocol completed. If the first and second proposer both propose different values, correctness is violated. This is a problem.

这种情况在异步的网络环境下是正常的。为了解决这个问题是让proposer来提议同一个值，这是为了避免在每个accpetor提交的值都是同样的带来的复杂性（This avoids the complications above by ensuring that all committed values are the same at every acceptor - no matter which proposer proposed them. ），让all proposer同意相同的值是简单的。怎么做？？

> . When acceptors respond to a prepare request, they reply with the value of the highest numbered proposal that they have already accepted.

此时，proposer只要询问这个值是否能commit。这样，proposer提交的值，很可能是其它已经完成的提议，而不是其原始的值，这有没有问题？？

答案是没有问题，这没有违反我们的需求，proposer和acceptor不在乎哪个值最终被接受，只要它是proposer发起的，但是一旦有绝大多数同意接受这个值，那么这个值将会被accpted。

> Acceptors, when they agree to a proposal, promise not to accept a value that is proposed as part of any proposal numbered less than the current proposal。

原因是：

**大多数同意意味着什么**：

> By having a majority of acceptors respond to every prepare request, Paxos ensures that every majority reply to a prepare request will contain a response from an acceptor that has seen each previously agreed proposal，

所以当proposer开始提交的时候，保证了其目前最大以前提交的协议号是多少（因为以前提交的也是大多数节点同意的，）

Paxos主要包括三个组件：Proposer，Acceptor和Learner，其中Proposer主动发起投票，Acceptor被动接收投票，并存储提案值，Learner负责将确定的提案广播出去。

**推倒过程**

* 最简单的方法，只有一个Acceptor，即所有的Proposer通过抢占式的方式来提交自己的value，**最先被提交的提案会最先被接收**，但这样如果这个唯一的Acceptor发生宕机或其他错误，后面的操作就无法继续。

* 推翻1的思路，我们采用多个Acceptor的方法，每个Acceptor接受一个Value, 只有当一个提案被超过半数的Accepots接受，这个提案才被确认。这里提出一个条件:
    * An acceptor must accept the first proposal that it receive.
    > 这是就需要允许Acceptor可以接受多个提案，即多个Proposer都可以向Acceptors提交自己的提案，最后再通过确认哪一个提案被超过半数的Acceptors选中来达到需求。

* 现在允许多个提案被同时提交，为了满足一致性条件，即Safety，我们需要保证被选择的提案里的Value是一致的。所以这里引入了一个提案的Number，为了遵守P1约束且保证只有一个提案被选中，这里引出了提案P2:
  
> If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.
    一个提案想被选中，必须先由Accetors所接受，

所以在继续在P2的基础上可以 推导出P2a:

> P2a. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.即如果一个提案已经被选中了，则后面的具有更高Number的提案必须具有相同的value。

这里有一个问题，假如一些Acceptors一开始没有接受过提案，根据P1的约束，它们必须接受自己所收到的第一个提案，而此时它们之前没有接受过任何提案，无法确定这个提案是否符合P2a的约束，所以继续衍生P2a的约束:

> P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v.

因为我们无法控制Accetptors之间对某一个提案Value的限定，所以我们转而要求Proposer对提案的Value做限定, 即假如一个提案已经被Acceptor所接受，后面每一个Proposer提交的提案的value都必须和被接受的提案的value保持一致。

> P2c. For any v and n, if a proposal with value v and number n is issued, then there is a set S consisting of a majority of acceptors such that either (a) no acceptor in S has accepted any proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S.


* 为了满足P2c的约束，一个Proposer如果想提交一个Number为n的提案，它必须知道目前Acceptor所接受的提案里是否存在提案号Number小于n的提案，如果Acceptor接受所有的提案都小于n，那么它这个提案号为n的提案将会被大多数Acceptor接受。所以为了简单，Proposer要求Acceptors不得再接受Number小于n的提案，所以Paxos将一个提案的提交分为两个过程:

    * Prepare阶段
        * A proposer chooses a new proposal number n and sends a request to each member of some set of acceptors, asking it to respond with:
            *  A promise never again to accept a proposal numbered less than n, and
            *  The proposal with the highest number less than n that it has accepted, if any
        Proposer选择一个新的提案号n发给每一个Acceptor，期待得到Acceptor的以下回应
            * 承诺不会再接受小于n的提案。
            * 如果存在已经被Accept的提案里提案号最大且小于n的提案，返回给Proposer
    * Accept协议
        * 如果Proposer收到了大多数Acceptor的回复，那么它就可以给Acceptors发送这个提案，假如prepare请求的回复里表示Acceptor之前没有收到过提案，那么Value就是自己本身这个提案自带的，如果Acceptor之前已经收到过提案了，那么Value必须是那个提案里附带的Value, 这个要求是为了满足约束P2b，即后面的提案必须认同前面已经被Accept的Value。



## 算法概述

其中Prepare阶段用于**block当前未被选中的旧的提案**，并发现**已经被选中的提案内容**, Accept过程用于真正的提案提交:

phase 1:

* Proposer选择一个提案号为n的提案发送prepare的请求给Acceptors
* Acceptor收到了一个prepare请求且提案号n大于之前回复过的prepare请求，Acceptor就会承诺不会再接受提案号小于n的任何提案，假如之前已经存在Accept的提案，那么Acceptor还会返回自己Accept的最新的提案Value。
* 假如一个Acceptor收到了一个prepare请求，但这个请求里附带的提案号Number小于之前prepare阶段承诺的，那么它将忽略这个prepare请求。

phase 2:

* 如果Proposer**收到了回复**，那么它可以给Acceptors发送这个提案，假如prepare的回复里表示Acceptor之前没有收到过提案，那么**Value就是自己本身这个提案自带的**，如果Acceptor之前已经收到过提案了，那么**value必须是那个提案里附带的Value**。
* 如果一个Acceptor收到了accept请求带有一个提案，假如这个提案号n不小于之前Acceptor在prepare回复中承诺的提案号，那么这个提案将会被Accept。


## 按角色来看算法

**PROPOSERS**
* Submit a proposal numbered nn to a majority of acceptors. Wait for a majority of acceptors to reply
* If the majority reply ‘agree’, they will also send back the value of any proposals they have already accepted. Pick one of these values, and send a ‘commit’ message with the proposal number and the value. If no values have already been accepted, use your own. If instead a majority reply ‘reject’, or fail to reply, abandon the proposal and start again.
* If a majority reply to your commit request with an ‘accepted’ message, consider the protocol terminated. Otherwise, abandon the proposal and start again.


**ACCEPTORS**
* Once a proposal is received, compare its number to the highest numbered proposal you have already agreed to. If the new proposal is higher, reply ‘agree’ with the value of any proposals you have already accepted. If it is lower, reply ‘reject’, along with the sequence number of the highest proposal.
* When a ‘commit’ message is received, accept it if a) the value is the same as any previously accepted proposal and b) its sequence number is the highest proposal number you have agreed to. Otherwise, reject it.

## 算法的容错
如果我们想要容忍f台accpter节点的错误，我们需要提供2f + 1 的accpetor。如果proposer挂掉了，我们可以安排其他的节点来接管协议，来发布它自己的proposer，假如proposer又恢复了，那么根据paxos的算法，只允许序列化号高的提交能保证正确性。


## 可能的问题
* 当两个proposer同时提交活跃时，们可以通过交替发布比先前提案“领先”的提案来“争夺”最高提案数。在这种情况得到解决，并就单一领导人达成一致之前，paxos算法可能不会终止。
* Accpeptor需要将最高的proposer id记录存储，如果acceptor失败呢，那么就会陷入Byzantine failure错误，

## 代价

在paxos执行的理想环境，a proposer将会发送 f+1, 接受f+1 回复，在第二阶段同样，这样两阶段总共又4f + 4条消息，延时时是4条消息，如果在f + 1个节点中有一个nodes reject或者时fail，那么协议又要重头开始。我们还要考虑acceptor将会将数据落盘，这个延迟比网络延迟要高，也要考虑进去。

---
title: 区块链共识算法
published: 2024-05-10
tags: ["BlockChain"]
category: BlockChain
description: "区块链共识算法"
draft: false
---

## 两条主要路线

1. 基于证明：POW、POS、PoStorage and Po What Ever!
2. 基于Committee，PBFT

## 区块链的网络模型分为

1. 同步网络：有时间上界
2. 部分同步网络：有时间上界，但不知道
3. 异步网络：无上界

## 概览

1. 基于证明：通过提供某种证明证明自己是leader，然后添加新的区块；添加的区块在被追加多个区块后达到确认状态
2. 基于Committee：通过投票决定下一轮的leader
3. 改进方案：
1. 安全性
2. 规模
3. 去中心化

## POW

![Pasted image 20240517184314](assets/Pasted%20image%2020240517184314.png)

### 中本聪协议

略

### 改进方案

#### Decoupling Blockchain Functions

![Pasted image 20240517184336](assets/Pasted%20image%2020240517184336.png)

通过分割中本聪协议中，证明leadership、打包交易块、通过追加区块进行确认，这样多合一方案造成的低效，实现性能的提升。

Bitcoin-NG：分离证明leadership区块与交易区块，在一个时间区间内，先挖掘证明区块，然后多次打包交易区块，提高性能，但引发别的问题——自私挖矿攻击、双花攻击等等。

Prism：分离 交易链、核心链以及确认链条，核心链中的区块指向多个交易区块，在核心链上进行POW工作，Voter链上则对Proposal进行确认。（这位更是重量级，值得细细品味）

NC-Max：分离交易同步与区块的确认，使用紧致区块机制来加快区块的验证。

#### 平行链

OHIE：同时展开k条链，每个区块包含所有链最后的区块的哈希，新的区块被计算出来后根据哈希的值按照某种算法被指定到某条链上。

ChainWeb & CliqueChain：每个区块的头部包含其他链的默克尔根的哈希。但要达到一样的一致性，其性能与中本聪协议无异。

Monoxide：考虑到将工作量分割开，避免无意义的工作，同时运行一次解谜提交多个区块，避免攻击者针对算力薄弱的子链。

Wang的研究，采用核心链的难度来动态调整其他子链。~~~

#### DAG 区块链

![Pasted image 20240517184350](assets/Pasted%20image%2020240517184350.png)

GHOST Greedy Heaviest-observed Sub-tree 协议，不选择最长链，而选择最大权重链，考虑到一个子链的“叔块”的作用，以太坊实现了一种加强版的GHOST协议。

在DAG中，全局序列化交易是一个复杂的工作，平行链虽然也有多个链，但总是按照相同的epoch行动，epoch内可以按照链的序号进行排序，但DAG中不存在这样的编号。

Spectre：一个子块可以有多个父块，节点使用其所知的所有末尾块作为前序块。

Phatom：将全局交易序列化问题转化为一个Maximum K-Cluster SubDAG 问题，NP难，其使用贪心算法解决，面对活性攻击存在隐患。

Conflux：引入Parent Edge与Reference Edge，以及GHOST原则，区分主链，并将DAG切分为多个Epoch，实现全局序列化。其还引入妥协策略用于抵御活性攻击与保持高交易量与快速取人，并在一定的权重机制下进行切换。

Occam：考虑Transaction的需求，动态调整难度。

### 区中心化的改进方案

矿池加重了区块链网络的中心化，引入自私挖矿攻击等问题。一般有两种解决思路：1. 避免节点形成矿池 2. 鼓励去中心化节点加入

#### 避免矿池

1. 设计算法使得矿池面面临收益被Worker窃取的风险

#### 鼓励去中心化

1. 混合工作量证明，对低算力设备更友好
2. 采用高内存占用的算法，避免ASIC矿机的竞争——以太坊ETHash

### 安全性改进方案

1. FruitChain：分离出Fruit与Block两个概念，强调Fruit对实时性的要求，使得挖矿者不得不及时提交区块，避免自私挖矿攻击。 ？？？
2. Bobtail：当最低的7个挖出来的随机数的均值低于某个阈值，即可提交新区块，稳定了块生成时间。
3. StrongChain：在采用正确答案时也考虑弱答案，可以进一步强调正确Branch上的算力水平，使得攻击者进行分叉攻击更为艰难。

## POS

![Pasted image 20240517184406](assets/Pasted%20image%2020240517184406.png)

PoS有两种路子：基于链的PoS与基于BFT0Based PoS

### Chain-based PoS

使用某种基于链的算法，能够决定下一个挖矿人是不是自己。

Peercoin：与比特币类似，PoW进行挖矿；但在验证方面，稍有不同，其提出Coin Age概念，持有人具有的Coin 与 持有时间的乘积，在进行质押时将成为权重。验证者将获得自己资产1%的奖励。

Nxt：第一个完全基于PoS的加密货币，一个节点持有的货币越多，越有可能有资格挖下一个矿。

SnowWhite：限制了加密货币的流动能力，抵御双花。每一轮基于上一个提交的区块进行哈希计算，选出一个Committee，然后再哈希计算从中选出leader，提交新块。

Ouroboros：PoS中，不可预测与不可操纵的领导人选择极为关键，一般使用伪随机算法基于现有区块链状态选出leader；但Ouroboros则不同，每一轮持币者需要执行一个多方掷硬币算法，选出领导人。

Ouroboros Praos：Ouroboros的改进版本，使用Verifiable Random Function进行领导人选举，更具安全性。

Ouroboros Genesis：希望达成full dynamic availabilituy，即只要诚实节点的算力占大多数，就能不在意任何节点上下线。使用本地的chain selection 规则而非全局。允许节点加入，仅仅根据创世区块，而无需检查点。

Ouroboros crypsinous：结合Zerocash与Ouroboros系列，保护隐私。但研究证明，只要敌对方能操控网络延迟，可用性与隐私不可兼得。

Pos 侧链：为了改进Ouroboros系列的互操作性、规模性与可升级性，引入侧链，使用双向铆钉，能够实现安全的跨链价值传输。

PoSAT：使用Verifiable Delay Function，避免额外的安全假设。为了生成新区块，节点需要迭代地计算先驱节点与公钥的VDF，直到发现一个抵御阈值的值。攻击者更难分叉链。

### BFT-based Pos

BFT-based需要先选出几个验证者与一个leader组成委员会。一般来说质押，然后基于密码学的选举，leader提出新区块，委员会实现BFT算法，决定新区块是否有效——经过投票。

Capser：以太坊的叠加层，用于规范以太坊的正规链条——指明主链，其基于投票生成检查点。

Tendermint：质押，使用两阶段提交，提交者提出区块，等待超过三分之二的投票，再进入下一个阶段，同时采用了锁定机制，避免一个节点同时发起投票，引发分叉。

Algorand：使用VRF选择leader与验证者。VRF是一个非交互算法，节点本地运行，获取运行结果。leader与validator实现Byzantine agreement算法，BA*来达成共识。BA* 运行多轮，不停得选出validator进行验证与投票，一轮又一轮，直到达成共识。

LaKSA：采用密码学采样在质押列表上采样，质押数越多采样概率越大。偏好可用性，节点将自己决定是否采用某个区块——其中记录了其得到的投票数量。

## PoStorage And POX-BASED

### PoStorage

![Pasted image 20240517184426](assets/Pasted%20image%2020240517184426.png)

存储服务器通过证明文件的完整性。

#### Proof of Replication and Proof of Spacetime

Filecoin：Proof of Replication 表征文件被接受并被存储在良好的物理设备上；而Proof of Spacetime表征数据被存储了一个特定长度的时间段。

PoRep：PoRep是一个挑战/回答协议。挖矿者根据自己的公钥存储一个伪随机的排列，并给用户返回一些信息。用户根据这些信息生成挑战，挖矿者根据数据进行零知识证明。（并不存储有价值信息） Filecoin递归计算PoRep，每一轮的输出作为下一轮输入，从而实现proof of Spacetime。

#### Proof of Space

SpaceMint：使用pebbling游戏，要求用户投入大量磁盘空间，磁盘空间越大，proof的质量越高，最高者成为新leader。

Chia：也采用PoSpace，并采用了VDF。

#### Proof of Retrievability

Permacoin：在数据中插入纠错码以及随机数，作为哨兵。通过质询哨兵证明数据可取回。

### Proof of X

Proof of Whatever

#### Proof of Elapsed Time

Hyperledger Sawtooth：**Trusted Execution Environment** 。即证明执行环境，每个节点从自己的围地中生成等待时间，最短者提出新块，拥有越多处理器越能够被选中。但容易被攻击，所以又提出了PoETA。

#### Proof of Meaningful Work

Primecoin：将puzzle替换为寻找巨大的素数。

Proof of eXercise：求解基于矩阵的计算问题，用于图像识别与数据挖掘。

## COMMITTEE-BASED CONSENSUS PROTOCAL

主题思想在于通过投票达到对某个提案的共识，且一旦达成，这个提案的区块将不会被代替。

![Pasted image 20240517221143](assets/Pasted%20image%2020240517221143.png)

### Permissioned

Raft与Paxos是基于有限的且已知的参与者的情况下的，是CFT共识协议，不支持拜占庭容错。BFT算法本来是用于确保无损节点能够对客户端产生的指令顺序达成一致，但其也很适合用于构建Permissioned区块链。

#### PBFT

![Pasted image 20240517221300](assets/Pasted%20image%2020240517221300.png)

PBFT的一致性不依赖于网络同步，但活性依赖于网络同步（CAP倾向于一致性与分区容错性）

PBFT执行Primary&Backup理论，即某个节点是Primary(leader)，而其他节点是Backup(validator)。其流程如图3：

1. Request：客户端发送请求——**客户端消息**(操作、时间戳、身份)
2. Pre-prepare：leader对view number、sequence number与请求摘要签名，并广播**Pre-prepare消息**(客户端请求与签名）
3. Prepare Phase：validator检查签名、view number与sequence number，如果全部正确，对Pre-parepare消息、view number 与sequence number与自己的身份签名，然后将其作为**Prepare消息**，本地保存一份，并广播到其他节点。
4. Commit Phase：如果验证者（与leader）收到2f份Prepare消息，匹配其自己的Pre-prepare消息，就进行Commit Phase。leader与验证者将Commit Message(view number、sequence number 与身份 的签名)进行广播，如果收到多余2f+1份Commit Message，就认为Commit Phase完成，执行指令，并将结果返回客户端。
5. Replay phase：如果客户端收到多余f+1份reply message 且结果相同，就认为达成共识了。

PBFT能够在含有3f+1个节点的情境下保证拜占庭容错，但无法处理恶意的leader。其引入了一个view 更替机制，validator带有一个计时器，当等待时间达到一定程度，就发起view change 消息，如果下一个view的领导人受到了2f+1份view change 消息，就广播 new-view消息，转为正常模式。

Byzcoin：在PBFT上引入collective signature 协议改进通信消耗，以及一个POW链，用于处理Permissionless模式。

许多区块链项目引入分片与PBFT达成大规模网络，Tendermint通过投票达成最终一致性。

#### HoneyBadgerBFT

PBFT依赖于同步或部分同步网络模型，但FPL的存在，要求设计者作出妥协。

>FPL impossibility指出，在消息延迟但不会丢失的网络中，如果至少一个节点失败、停止，则不存在保证在所有启动条件下都能终止的共识算法。

HoneyBadgerBFT基于完全异步的环境，其活性不依赖于对消息延迟上界的假设。所有节点采用Asyncronous Common Subset(ACS)来发送消息。如果N个节点都发出一个值，那么ACS保证每个节点发出一个向量，含有至少N-2f个正确输入值。然而ACS导致缓慢，所以节点随机选择交易，广播最无交集的交易集。……

其步骤：

1. 每一轮，每个参与节点随机选择一批交易，发送前进行Threshold加密。
2. 在第二阶段，节点使用ACS广播交易。ACS包括Reliable Broadcast(RBC)阶段与Asynchronous Binary Byzantine Agreement(ABA)阶段。交易通过RBC广播，然后ABA生成一个bit向量，指示哪些交易发送成功。一致合集通过ABA生成。
3. 在第三阶段，每个节点解密他的部分并广播之。交易集可以在接收到至少f+1个解密消息后解码出来。最后，这些交易将形成新区块。

#### Improvements of Asynchronous BFT Consensus Protocol

BEAT：改进了HoneyBadgerBFT的效率，利用了更高效的threshold 投硬币算法，代替原来的threshold 加密算法，包含五套不同的方案可以适时使用。

DispersedLedger：将共识算法解耦为数据可用性约定与基于HoneyBadgerBFT的区块检索。高带宽节点可以不用等待慢速节点。

Dumbo：涵盖两个原子广播协议Dumbo1与Dumbo2。HoneyBadger要求每个节点运行N个ABA实例，非常慢，但采用Dumbo1减少ABA实例数量，Dumbo2 采用multi-valued validated Byzantine agreement(MVBA)来优化ACS。Dumbo-NG与Speeding Dumbo也朝着改进效率前进。

#### Other Improvements of Committee-based BFT Consensus Protocol

BFT要求多轮交互实现一致。

PILI与PALA采用流水线BFT协议，基于同步(PILI)/部分同步(PALA)网络假设。

HotStuff 目标在于简化领导人替换的复杂度，基于部分同步网络假设。其添加了一个decide phase在commit phase后，新领导人能够选择最高法定人数证书来趋向一致。而基于同步网络假设的Sync HotStuff不再要求所有节点在同一时刻开启、结束一轮。

Flexibale BFT：Committee-based的BFT协议要求预设网络环境、Byzantine节点比例，一旦变换，活性与一致性就难以保障。Flexible Byzatine协议结构BFT协议，并设计Flexibale Byzantine Quorums。其运行一个账本容纳具有不同假设的节点，以及不同比例的拜占庭节点。其通过客户端选择不同的commit规则实现。

Momose：提出了一种多thresholdBFT协议，适配多种网络环境假设，且仅有一个提交规则，其安全性可以忍受同步网络情况下2/3的节点故障，活性可以忍受同步网络条件下1/3的节点故障以；异步与部分同步下则可以忍受1/3的节点故障。

Pompe：Leader节点可以通过操纵交易的顺序实现牟利，而Validator仅能验证交易的正确性，却无法验证交易顺序。Pompe解耦了交易顺序与添加记录，通过将交易添加顺序标识，leader无法再控制交易顺序。

Aequitas：transaction order-fairness 成为safety与liveness的又一属性。Aequitas中，交易基于FIFO-broadcast规则进行广播，节点通过Byzantine Agreement(Set-BA)对交易顺序达成一致。交易的最终顺序可以被基于leader的模型与无leader模型验证。

BFT取证，基于最大拜占庭节点数量、证明有罪的诚实节点的交易数量以及在一致性被违背的情况下可以被定为有罪的拜占庭节点数量。

### Permissionless

![Pasted image 20240518005659](assets/Pasted%20image%2020240518005659.png)

由于传统的基于委员会的BFT协议在无限制的网络环境下无法适应：

1. 需要预先固定的参与数量来保证liveness与safety
2. 没有身份控制机制，会遭遇女巫攻击
3. 需要多轮交互，网络数量大的时候通信开销很大

方案1：先使用基于证明的算法从环境中选出节点组成committee，然后执行PBFT算法
方案2：使用分片，将节点分配到不同sharding中。

挑战：

1. 需要合理的分配方法，将节点公平随机地分配到不同分片
2. 需要避免跨链交易，否则将带来高开销与安全隐患
3. 分片内算法需要安全与高效

Elastico是第一个采用分片的，其要求节点进行PoW避免女巫攻击，然后按照哈希计算结果分配到对应分片中。哈希结果最低的组成final committee，其他节点组成各自committee，交换身份，实现PBFT；而final committee负责验证其他committee的交易。

OmniLedger指出Elastico不安全，因为攻击者可以选择性公布哈希结果，从而进入同一个committee，而且final committee 很容易成为性能瓶颈。OmniLedger采用VRF-based领导人选举，与无偏的随机算法RandHound来讲validator分配到不同分片，且设计了Byzantine Shard Atomic Commit 协议（上锁开锁）。首先，用户需要从输入的交易所属的shard的leader处取得proof-of-acceptance，然后输入交易将被上锁。然后，用户**流言**一份unlock-to-commit交易，其包含所有所有proofs与上锁的交易，负责用户将会流言一份unlock-to-abort交易终端跨链交易流程。

这个过程需要用户自己做很多事情，不利于轻量级客户端的实现；齐次，validator需要对每个用户的证明签名，造成非常高的通信与计算开销。

Rapid Chain：对整个网络的流言造成极大延迟，RapidChain实现了一种新型的流言协议与一个内部委员会，极大降低了延迟。遗憾的是，Rapidchain无法实现跨链交易的原子化与隔离。Dang通过使用two-phase locking与two phase commit实现了跨链交易的原子化与隔离。

跨链交易是分片区块链面对的一个主要挑战。OptChain与BrokerChain尝试通过合理分配transaction到shard中。OptChain通过学习过去transaction模式，但仅适用于UTXO模型；BrokerChain应用于Model模型，但依赖于Broker，且不支持多输入多输出交易，因此非常慢。Pyramid尝试构建层次分片，某一个分片存储了其他几个分片的数据来实现，但失败了。

### Other Miscellaneous Protocol

#### Tx-based Chain

Tx-based Chain基于trasaction而非block。

![Pasted image 20240518014133](assets/Pasted%20image%2020240518014133.png)

IOTA中的Tangle就是一个DAG区块链，其直接关联交易，略过区块。交易的添加不需要挖矿与费用。添加交易时，一个节点需要验证两个已有交易，然后关联自己的新交易上去，者使得其没有严格的confirm时间，越多后续交易关联到前序交易，前序交易可信度越高。Byteball是一个类似的项目。

#### Hashgraph

Swirlds hashgraph 一致算法是一个完全基于异步网络模型的BFT协议，采用DAG结构。其独特地依赖于gossip about gossip 以及 virtual vote 来实现一致。Hashgraph采用event结构而非block，每个event含有两个指向父event的指针，一个是其自己的最后一个event，另一个则是从别的节点接收到的event。节点对他们接收到的event签名，然后通过gossip about gossip 转给别人。在此过程中，节点可以更新其自己的hashgraph。如果所有节点具有一样的event，那么那个event的祖先以及对应的边也将存在。event的传递代表着节点对那个event的投票，故可以从hashgraph计算出总的票数，实现virtual vote。 这是完全异步以及节省带宽的。

#### Hyperledger Fabric

大多数BFT共识协议基于排序-指向模型，然而Fabric采用指向-排序-验证模型。

1. 执行阶段：客户端发送交易提案到特定节点——endorsers。endorsers模拟执行，并给出结果（writeset 与 readset），然后生成endorsement消息到客户端，包含其对执行结果的签名与其他信息。
2. 排序阶段：交易被送至排序服务节点，如果客户端收集到了足够的endorsement，排序服务讲打包交易，并生成特定的区块序列，并广播到peer节点，通过gossip协议。
3. 验证阶段：peer节点验证endorsement，如果无效，transaction将被丢弃，否则，进行读写检查，最后，更新阶段，讲区块添加到本地区块链上。

#### Delegated Proof of Stake(DPoS)

DPoS不随机选择leader与validator，而是通过投票决定谁来生成下一个区块。

Bitshars：持币人通过投票选择节点作为 目击者。 目击者负责生成、验证与广播区块。目击者需要质押，并可以在成功生成区块后获得奖励。

EOS：得到足够投票的节点可以成为区块生产者，每一轮21个节点被选举，合作生成新区块。EOS融合了asynchrounous Byzantine Fault Tolerance 技术来达到快速交易确认。

在DPoS中，节点间需要建立信任关系，降低了去中心化程度。

#### Non-anonymous Proof-based Protocols

Ethereum Proof-of-Authority就是为企业准备的permissioned区块链。每个节点都需要一个公开的身份，所有区块被其生产者签名。节点生成有意义区块的收益与坏行为的惩罚都是透明的，其中添加新成员、删除节点、选择管理员与验证者都通过投票进行。

GoChain 采用proof of repuation。只有具备一定公信力的节点可以作为认证节点，只有认证节点可以生成、签署新区块以及验证新区块。

#### Redactable Blockchain

可编辑区块链用于

部分研究采用变色龙哈希算法代替传统的单向算法，变色龙哈希算法具备陷门，如果没有钥匙，变色龙哈希算法无碰撞，而如果掌握钥匙，就能简易地生成一样哈希的区块，从而修改内容。

另一部分研究则采用投票实现可逆。

## 比较

![Pasted image 20240518163510](assets/Pasted%20image%2020240518163510.png)

基于证明的方法与基于委员会的方法。
![Pasted image 20240518154245](assets/Pasted%20image%2020240518154245.png)

前者基于中本聪的共识协议，允许任意节点在任意时刻加入协议，除了一些特殊的设计——proof-of-authority 与proof-of-reputation。优势在于无主权完全去重心化，有助于达到强匿名性。然而这往往导致低吞吐量与更长的确认延迟。同时，基于证明的协议也进能够达成概率一致性，无法保证达成100%参与者的一致。每一轮都是概率的，但多轮的执行可以达成更高的确信度，这又进一步延长了确认延迟。

基于委员会的协议则从传统的分布式一致性算法，如PFBT演进而来。一个固定数量的参与方组成委员会，达成一致。参与方的数量是固定的，身份是明确的，匿名是不可能的。通常来说，能够保证结果得到多数同意，得到确定的一致性。一旦一个交易被添加到区块链上，他就被确认了。相比于基于证明的方法，具有更高吞吐量与更低的确认延迟。

为了达成去中心化与高吞吐的统一，出现了三条演进路线：

1. 保持基于证明的领导人选举，但采用新的结构来实现并发，如DAG、平行链。
2. 扩展基于委员会的协议来允许节点加入，如分片。
3. 混合模型，基于BFT的PoS模型，节点的加入无需验证，但需要证明持有足够的stake。

PoW与PoS是两种主要的基于证明的机制，PoS不需要大量计算，而通过持有加密货币来参与共识协议，但相比于PoW丧失了部分去中心性质。
![Pasted image 20240518163521](assets/Pasted%20image%2020240518163521.png)

基于对网络模型的假设，基于委员会的共识协议可以分为三类：

1. 同步：允许n/2的拜占庭节点
2. 部分同步：n/3的拜占庭节点
3. 异步：n/3的拜占庭节点
同时，异步网络将无法达成决定性的协议，因为FLP impossibility的存在。

## POTENTIAL FUTURE RESAERCH

### Efficient Transactions Processing

最小化冲突、重复的交易，同时允许并发区块，避免并发区块中冲突、重复交易对效率的损害。

提高跨片交易的处理。想办法避免跨片交易的发生。

### Performance Improvement of Committee-based BFT Protocols

采用阈值签名、集体签名，避免all to all gossip。

将Committee方案移植到permissionless环境，非Committe组成节点的资源将被浪费，可以考虑将其应用到解耦功能与异步处理上。

### Cross-chain Interoperability

区块链在未来连成一体是大趋势。互操作性将是一个重要的指标。

1. 侧链：侧脸允许多个区块链通过two-way pag实现跨链操作。一个区块链可以成为另一个区块链的侧脸。数据与资产可以安全高效地在不同区块链间交换。
2. 异构实现多个分片的互操作性。比如，以太坊中除了多方分片链，存在一个信标链，用于童鞋不同的分片中的链。

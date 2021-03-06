---
title: 游戏中的经典AI设计：MOBA英雄AI设计分享
date: 2018-07-23 20:44:38
tags:
 - MOBA
 - 游戏策划
categories: 其他游戏分析
---
# 设计概要

## 设计原则和目的

英雄AI的目的主要有：


> 1.新手过渡局，让玩家刚进入到游戏时，和较弱电脑对战，培养成就感，避免尚未熟悉游戏导致的挫折流失。

> 2.人机对战，给玩家练习新英雄或者挑战高难度电脑的机会。

> 3.温暖局，对连败玩家，匹配机器人去补偿一场胜利，舒缓连败挫折。

> 4.掉线托管，用强度合理的AI来补位掉线玩家，减少其他在线玩家的掉线局有损体验。


英雄AI的设计原则是：**优秀的AI并不要求是尽量的和人表现一致，也不是多么的精准和无懈可击，而是能够和玩家进行很好的交互，提升游戏体验。**


## 设计思路

我们的AI实现分为四个阶段，正好类似于玩家的成长。

* **第一阶段**是基本战术AI，主要包括：混线，买装备，逃避危险，回城，补兵。是一种单兵作战AI。模仿新手玩家的刚刚开始学习操作。

* **第二阶段**是增加一些事件响应用来控制英雄的走位和换线，包括敌塔下撤退，救援己方塔，包括抱团。模仿玩家已经开始渐渐了解塔的属性，初步开始与其他玩家合作。

* **第三阶段**是协同战术AI，该AI周期性的判断是否应该果断出击打出一波局部进攻。它会在比较短的时间内控制局部范围内的单位一起行动，会有走位，配合使用技能等较细致的行为，是一种小团队AI。模仿玩家已经开始熟悉所有英雄，微操提升，对Gank略有心得。

* **第四阶段**是战略AI，整体协调全部玩家在地图上的分布，野区，兵线。模仿玩家已经有较强的团队意识，会分工和配合了。

#  名词解释

* 1．**单体战术AI**：每个英雄都会配备自己独特的战术AI，此AI将实现战斗细节，比如英雄何时该释放技能，对谁释放;如何走位规避风险或者形成Gank优势站位;怎么补兵;购买贩卖何种道具;何时追击何时逃跑等等。

* 2．**全局AI**：全局AI是一种综合考虑场上所有战斗因素之后对单体发布指令的控制器。全局AI所关注的事情主要有：兵线英雄的分布，Gank发动时机，逃避危险，救援建筑。全局AI是通过给单位添加指令buff和修改单体战术AI的参数来实现的。

* 3． **AI参数**：我们将尽可能的暴露出AI的各种行为参数，并通过AI参数来控制电脑的AI难度强度。高难度AI，意味着它优先使用较高收益的战略。而低难度AI则可以选择比较低收益的战略。我们的不同难度AI是通过修改AI的一系列参数来实现的。

* 4．**行为树**：树形结构的行为流程处理，每个Tick到来时，行为树按照一定的规则进行搜索和执行相应节点，直到到达某个返回true的叶节点，之后结束当前Tick。

* 5． **Gank 小组**：Gank小组是一个动态的局部的概念，当我方英雄A周边有敌对英雄时，英雄A就是属于某个Gank小组的，Gank小组的其他成员必须和A距离很近。

* 6． **Gank 行为**：Gank行为是一种对集体行为的模仿，其本质仍然是单体AI，但Gank发动时机是通过全局AI来控制的。处于Gank状态的机器人会表现出与单体行动很不一样的行为，比如坦克可能宁死也不撤退，ADC优先释放控制技能。

# 行为树实现

## 行为树脑图

行为树脑图是一个多叉树，各个父节点的所有子节点节点按照从左到右、从上到下的顺序逐个检测，只要返回True了，之下的节点都不再执行。灰色注释为节点执行的先决条件，灰色节点不满足则直接返回False。脑图中的

对应着行为树中的Selector节点。

行为树工具基本思想都一致，但使用起来还是有较大差别的。常见的是Unity3D的BehaviorDesigner插件，虚幻4自带的行为树组件，公司内部的Behaviac。我最喜欢的是BehaviorDesigner，学习时还是推荐Behaviac，传送门：http://www.behaviac.com/language/zh/%E9%A6%96%E9%A1%B5/

原因比较简单，只有它是中文。

![英雄AI行为树脑图](https://github.com/CGNz/blogimage/raw/master/MOBAAI/01treehead.jpg)

这是一个尚未展开的行为树，每个超链接都对应一个子树，会逐个展开来讲解。

### 购买道具

![购买道具](https://github.com/CGNz/blogimage/raw/master/MOBAAI/02buy.jpg)



英雄购买道具需要提前写好英雄对应的阶段道具设置。

比如：

![出装流程](https://github.com/CGNz/blogimage/raw/master/MOBAAI/03chuzhuang.jpg)

每隔一段时间检测一次金钱是否可以买卖下阶段的道具。

### 濒死逃亡

![濒死逃亡](https://github.com/CGNz/blogimage/raw/master/MOBAAI/04taowang.jpg)

### Gank战术行为

![Gank战术行为](https://github.com/CGNz/blogimage/raw/master/MOBAAI/05gankact.jpg)

每个英雄都需要单独编写此子树。首先搜寻最优攻击目标，而后检测是否能用技能组合一次秒之。

* **最优技能释放目标搜索**

满足以下条件的单位应该优先被锁定：

1.HP较低

2.AP或者MP较高

3.物理或魔法护甲较低

4.处在友方其他英雄攻击范围内

我们可以使用如下计算公式(本文里面的任何公式都不一定是最优解的，但都满足定性的设计要求)：

![最优技能释放目标搜索](https://github.com/CGNz/blogimage/raw/master/MOBAAI/06gongshi.png)

其中a,b为参数，AllyNearBy为敌方英雄600码内我方英雄数量，每增加一个盟友，敌人的诱惑程度增加b。推荐参数值a=0.7, b=0.3

技能是否使用只对最优释放目标进行考虑。

### 推兵线

![推兵线](https://github.com/CGNz/blogimage/raw/master/MOBAAI/07tui.jpg)

英雄磨血节点需要考虑收益，计算公式：

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/08gongshi.jpg)

收益值要考虑率较多因素，包括敌我双方血量，敌方英雄的同盟单位，收益值可能为负值。

### 执行AI行动指令 

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/09zhixing.jpg)

AI行动指令一般都是通过行为树之外的全局AI脚本来产生，并通知给AI行为树。常见的使用方式是，用一个全局AI脚本来产生各种指令，将指令传递给行为树，实现全局AI控制单位。

## AI事件响应

### 英雄躲避塔的攻击

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/10duobi.jpg)

避免英雄冲塔行为。

### 全局GankAI

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/11gankai.jpg)

周期计算Gank形势。通知AI是否该Gank或者集体逃亡。

### 救援塔

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/12jiuyuan.jpg)

当塔受到攻击时触发，用来产生AI指令，控制AI行为。

### 兵线分布调整

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/13bingxian.jpg)

当游戏运行时间超过6分钟时，AI要开始抱团，强推一路，之后每三分钟都要进行一次抱团检测。

兵线危机值计算：

兵线局势需要考察的因素：英雄数量，士兵数量，塔的数量，前塔的HP，推荐公式：

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/14gongshi.jpg)

其中a,b,c为参数，Lane表示兵线1,2,3。对应10v10游戏推荐参数设置：a=8, b=2, c=6, d=0.2,e=20

兵线危机值可以是负值，危机值越高则兵线越危险，值越低则兵线越安全。我们每10秒计算一次兵线危机值，根据兵线的状况来决定是否援助和抱团。

抱团是一个较为稳定的行为，我们设定每次防守抱团之后都要锁定切换兵线行为3分钟，进攻抱团锁定2分钟。

从另外两条兵线抽调英雄到最危险兵线。派遣数量服从规律：抽调后兵线上 我方英雄数目/敌方英雄数目>0.65(参数)，尽可能多抽调英雄，但也确保不会让被抽调的兵线变得很不安全。派遣数目可以是0，表示全线吃紧，每条兵线都无法抽调英雄去支援其他兵线。初期，每条兵线最少也要保留一个英雄。

## Gank详解

### Gank行为基本设定

* 首先要明确几个设计前提：

1.Gank行为优先级要高于单体行为优先级，或者说，Gank行为执行期间会屏蔽掉大多数单体AI行为。

2.Gank行为需要考虑到局部范围内(比如说整个屏幕)所有单位(包括敌方)，而后控制所有我方英雄一起行动。

3.Gank AI控制下的机器人可能会表现出和单体AI完全不一致的行为，比如肉可能直接冲到敌人人堆中，吸收仇恨，至死方休;ADC和APC最优先的策略可能不是输出，而是控制;部分机器人输出伤害优先级要高于逃避危险。

4.Gank行为并非常态。达成一定条件之后才会触发。比如某个时刻敌我力量对比呈现一边倒

* Gank小队的生成

Gank是局部小团队行为。必须考察周边敌我英雄和塔的个数，英雄和塔的潜在杀伤。Gank是个局部行为，只有距离很近的那些单位才会被认为是处于同一个Gank小组内。Gank小组是个动态变化的单位组。需要每隔一段时间重新生成一次。

生成方案：

寻找Gank中心英雄，Gank中心英雄只是根据位置搜索产生的，并不意味着它们会在Gank中处于核心地位。每隔一个周期(2秒，参数)先遍历某阵营场上全部英雄，统计这些英雄身边敌对英雄的数目。并按照递减顺序排列。身边敌对英雄越多，该英雄越可能处于Gank中心位置。按顺序遍历己方英雄(只遍历身边有敌对英雄的)，如果它们还未参与Gank，则以该英雄为中心，在一定半径(2000，参数)内搜索敌我未参与Gank的英雄，将盟友英雄写入Gank小队，并标记它们已经参与Gank了，将敌方英雄写入Gank目标小队(目标小队并不是敌方的实际Gank小队，敌方的实际Gank小队生成方式和我方一致)。如此，所有可能正处于交战状态的英雄就按照区域划分到了不同的Gank小组。

* Gank的发起和结束

Gank小队是动态生成的，每一时刻Gank小队都是存在的，但发起Gank行为是需要条件的。

每隔一段时间要检测一下Gank小队的实力对比.

1. 如果我放Gank小队实力明显强于目标敌方小队，则发动Gank，并锁定5(参数)秒。Gank期间英雄优先执行Gank AI，屏蔽掉单体行为。Gank结束锁定后。重新生成Gank小组，重新判断形势，决定是否发起新的Gank。

2. 当我方Gank小队实力明显弱于敌方时，集体执行撤退到己方前沿塔。但并不进入Gank行为。

3. 均衡局面，如果有敌方单位可秒(可秒的含义是，gank小组的输出期望是目标单位hp的1.6(参数)倍)，则立刻发动Gank。否则调整我方站位，综合防御最强的英雄位置保持不变，脆皮远离敌小队中心，但不能离开坦克超过(1000参数)。调整站位是单体AI行为，战略AI通过参数来控制单体行为(发送指令buff，发送目标位置)。

### 技能伤害量化

如果希望AI精准的释放技能，量化技能伤害是至关重要的。并不是所有技能都是直接立即伤害的，AI要怎么理解自己的被动技能和buff技能?

我们做的处理是：

l 默认在一次Gank周期中AI可以普通攻击三次，或者5秒。

l 将被动技能，比如暴击和加速之类的，直接量化为三次攻击或5秒攻击中的伤害收益。

l 晕眩技能根据晕眩时间量化成额外伤害百分比。

l 辅助技能仅仅起加强队友作用的，伤害量化为0

当技能全部量化成具体数字之后，就能计算每个英雄在单次Gank中的伤害输出期望值了。

* **英雄威胁值**

我们用英雄威胁值来表征英雄在单次Gank中的伤害输出期望值。

* 威胁值的计算：

首先遍历场上所有英雄，根据英雄技能等级和CD状态预估出来技能的三种伤害(物理，魔法，真实)数据。

对峙双方如果威胁值总和差别很大(参数60%)，则认为非均衡局面出现。优势一方会立刻发起Gank，进入团战模式。而劣势一方会立刻进入集体撤退状态。

威胁值相差不是很大时，英雄表现为单兵行动。此时威胁值的主要作用是敌对目标选择。

### GankTarget选择

GankTarget的选择方式——寻找最具吸引力的敌方单位，改进版的吸引力公式：

![](https://github.com/CGNz/blogimage/raw/master/MOBAAI/15gongshi.jpg)

这个公式综合考虑的因素有：敌人是否高AD或者高AP?物理护甲和魔法护甲如何?当前血量?我方集火的情况下，伤害总输出能杀死他几次?

最大吸引值得敌方英雄会成为Gank小组的共同目标

# 总结

在本文中，我们按照从零开始逐步展开，完整描述了MOBA英雄AI的设计流程。限于篇幅，我们仅仅描述了最核心的框架，诸多细节都未展开。在手游 MOBA《全民超神》项目中，按照这个框架，我们在短短一个月时间内就实现了英雄AI。

本方案原创了两个核心设定：Gank和技能伤害量化。

Gank的设定让AI能够有效的躲避危险，也能很精准的捕捉战机，完成很多让人赞叹的绝妙击杀。

伤害量化，让AI理解自己技能的特性。对AI行为收益优化帮助很大。
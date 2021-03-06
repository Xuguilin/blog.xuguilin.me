---
layout: post
title: 如何高效使用亚马逊EC2之服务可用性与数据存储 
---

{{ page.title }}
================

<p class="meta">2014/06/10 - 上海 By 徐桂林</p>

【本文为我在2014年06月发表于《程序员》杂志的一篇文章。[阅读《程序员》原文](http://www.csdn.net/article/2014-06-10/2820154)】

自发布以来，AWS EC2服务（当时只有唯一实例类型：m1.small）已从提供基本运算能力不断扩展成现在非常完善的弹性计算基础设施服务，不仅运算实例的类型极大丰富，而且其相关服务（如EBS、Elastic IP、Auto Scaling、ELB等）也陆续加入。EC2已成为整个AWS服务的基石，很多后来发布的AWS服务，如关系数据库服务（RDS）、弹性MapReduce计算服务（EMR）等都是基于EC2来构建的。现在，EC2服务包括的内容越来越丰富，如何高效、经济地使用好EC2服务成为很多AWS开发人员关注的要点。在过去4年里，我们一直在使用AWS构建整个在线服务。因为项目后台是运算密集型的服务，所以EC2是使用最多的服务（占整个项目AWS成本的90%以上）。本文将从以下几方面分享从实际项目中总结出的EC2使用经验。

- 服务可用性：AWS EC2在自身提供高可用服务 的同时，还提供了多项相关服务帮助我们提高服务可用性。正确使用这些服务可以有助于构建高可用的AWS在线服务。
- 数据存储：AWS提供了各种存储服务，其中和EC2结合最为紧密的是EBS和Instance Storage。如何合理选择和使用这些存储服务是使用EC2中的一个关键考量。
- 服务安全：在公有云服务中，安全永远是一个关键话题。AWS EC2在提供安全保护上下足了功夫，我们需要合理使用好这些服务策略来保护好自己的服务。
- 成本控制：相比于存储、网络等服务，计算服务还是比较昂贵的，EC2也是如此。因此，高效、经济地使用EC2服务就成为整体服务成本控制中最重要的一环。
- 自动部署：在强调敏捷开发的当下，支持快速部署是必不可少的。AWS为快速部署服务提供了多种方案，并且有足够丰富的API来支持自定义部署策略。与此同时，AWS在全球已有多个数据中心，所以在全球快速部署服务也已成为可能。

上面的任何一个方面都可以有非常多的内容。因此，接下来的讨论只会围绕EC2来介绍相应领域的个人经验和建议。

## 服务可用性

通俗来说，服务可用性就是服务对用户可用的概率，一般以百分比表示（如AWS EC2提供99.95%的可用性保障）。保证高服务可用性是一个非常复杂的挑战（尤其是在高并发和复杂技术栈的情况下），而这个指标又是所有在线服务的基础性指标。为此，AWS EC2在保证自身高服务可用性的前提下，还为部署在其上的各种服务提供了构建高可用性服务的各种基础设施。因此，在讨论这个主题时，首先看看AWS推荐的典型Web服务架构（图1），我们便能总结出一些推荐的实践经验。

![Web架构图](/images/2014-06-10/web-arch.jpg)

图1 AWS Web应用推荐架构

1）在多个Availability Zone（AZ）中部署服务。每个AWS Region内的AZ都是物理上单独隔离的，它们有各自独立的供电系统和网络接入，甚至是两个完全独立的机房。当服务部署在多个AZ下，只有部署的所有AZ都出现故障时，服务才会被完全中断。理论上说，如果服务同时部署在N个AZ的EC2上，那么EC2问题影响服务可用性的概率就降到（0.05N) ×100%。

2）用Auto Scaling（AS）来自动调度EC2实例。通常来说，当服务遇到一些突发或者预期的高流量时，或者你的服务出现某些异常时，整个服务的可用性会受到极大挑战，而AS机制就是帮助你应对这些情况的。AS机制可以按照之前设定的Scaling Up和Scaling Down条件自动启动或者关闭EC2实例以应对流量变化和服务异常。一般来说，任何在线服务都应该有这种自动横向伸缩的能力，AS就是在基础设施层提供这种能力，而你只需要关注在应用层支持这种能力。当然，如果AS机制不能满足需求，那么完全可以利用EC2的API实现自己的Scaling算法（其实我们就是这么做的）。

3）用Elastic Load Balancing（ELB）来平衡多个EC2实例的负载。其实，负载均衡是个存在已久的概念，并且在传统的数据中心也已有相应的实现（软件或者直接硬件）。但ELB比起传统数据中心的负载均衡器有以下一些明显的优势。

- ELB自身也是自动弹性伸缩的。当有超大流量时（尤其是受网络攻击或者有突发事件时），Load Balancer自身的负载就会是一个巨大的挑战。而ELB是基于EC2实现的，可以非常方便地自动伸缩容量来应对这些状况（注意，ELB自身弹性伸缩对开发人员完全透明，这应该是称它为弹性负载均衡器的原因）。
- ELB能非常容易地与AWS的其他服务（如Auto Scaling、Cloud Watch、Route53等）结合使用。尽管ELB可以独立工作，直接向注册的EC2实例分发流量，但向ELB注册一个Auto Scaling Group会是一个更常见的选择。同时，ELB还在利用Cloud Watch确定服务实例的健康状况。另外， ELB的Endpoint也常常作为服务在一个AWS Region的入口而配置在Route53的DNS解析服务中。

4）在多个数据中心（Region）部署EC2实例。尽管AWS多数据中心的初衷是帮助客户服务全球各地的用（提供更好的网络环境，符合各地的法律需求等），但多数据中心部署也可以帮助我们提高服务的可用性。你可以在一个数据中心彻底瘫痪的情况下把用户导向其他数据中心。但在使用这个方法时，需要注意网络性能和法律风险等相关问题。

在介绍完保证可用性的各种AWS服务后，下面这些使用建议能帮助我们更好地使用这些服务。

1）尽量在多个AZ中部署大体相当的服务能力（如EC2实例）。尽管EC2可用性是值得信赖的，并且ELB已帮我们很好地平衡负载，但某个AZ整体出现问题的情况还是出现过的。当突发事件发生时，这种多AZ大体平衡的设计可以更好地保证服务可用性。

2）提供简单的多数据中心切换逻辑。尽管需要多数据中心切换的情形并不是很多，但它却是真实存在的。一个月前，AWS美西数据中心EC2服务就出现问题长达4个多小时。在这期间，我们在美西数据中心的所有服务都不可用，而更严重的是我们服务的绝大部分用户都是来自美西的。幸运的是，我们的服务支持动态多数据中心切换（这个设计初衷是帮助用户动态选择最佳的数据中心），简单地修改一个配置就把所有流量导入到美西数据中心（当然用户的使用体验可能会因为网络延时增加而有所下降）。尽管我们自己实现了数据中心动态切换逻辑，但现在Route53会是一个更好的选择。只需要在Route53中简单设置每个Region Endpoint的Failover逻辑就能达到类似效果。我建议部署在AWS上的全球服务最好使用Route53以增加整体架构的灵活性。

3）开启ELB的“Cross Availability Zone”功能。当启动一个ELB时，它会在所有需要的AZ中创建相应的Load Balancer实例，然后把用户流量轮流分配到不同AZ的Load Balancer实例上。默认情况下，每个Load Balancer实例只会把其上面的流量交给处同一AZ的EC2实例处理，如图2所示。

![ELB默认工作图](/images/2014-06-10/elb-default.jpg)

图2 默认的ELB流量分发模式

在绝大部分时候图2的架构可以工作，但有些情况下它会导致各个AZ负载不平衡，甚至服务的不可用。例如，某个AZ上的服务实例整体出现问题，所有分配到该AZ的用户请求都会出错。另外，由于DNS解析返回给用户的是每个AZ上Load Balancer的IP（这个DNS解析的TTL值很短，默认为60秒），如果用户短时间内对服务发起大量请求仍然会导致不同的AZ承担的负载完全不同（我们内部QA测试就遇到过这样的问题）。而且，某些业务需求还需要我们Stick连接，这更加大了负载不平衡的可能性。为此，ELB加入Cross Availability Zone功能。当用户开启它后，每个Load Balancer会在所有实例上平均分配流量，而不仅是当前AZ的实例。整个结构如图3所示。

![ELB跨AZ工作图](/images/2014-06-10/elb-cross-az.jpg)

图3 开启Cross Availability Zone后的ELB流量分发模式

当然，这个功能也会带来一个副作用，那就是Load Balancer和EC2之间通信可能会跨AZ，从而增加网络延时。不过，AWS设计数据中心时对同一个Region下的多个AZ之间的网络延时有严格限制，所以这个副作用一般不会影响到终端用户的体验，除非服务是延时非常敏感的类型。

4）开启ELB的“Connection Draining”功能。在日常服务维护中，运维人员遇到的挑战之一就是处理服务升级和各种异常情况。ELB的“Connection Draining”可以在这方面帮助到运维人员。它的基本思想就是当某些EC2实例需要退出当前服务资源池时，让ELB不要分发新的请求到这些实例上，并且设定一个Timeout时间来尽量保证当前已经在这些实例上的请求仍然能够正常处理完成。类似的思想对于运维人员肯定不新鲜。传统的做法是通过Load Balance或者Proxy来做流量切换（这个ELB自身也支持）。但ELB的“Connection Draining”最大的好处是可以让整个过程自动化完成。实践中可以结合ELB的流量切换和“Connection Draining”，从而达到高效、平滑地升级在线服务或处理异常实例。

## 数据存储

为适应不同需求，AWS提供了多种不同存储服务。在这些存储服务中，EBS和Instance Store与EC2最直接相关（EC2还会和其他如S3、DynamoDB、RDS等进行数据交换，不过它们都超过了这里要讨论的范围）。EBS是AWS提供的持久化块存储服务，可以方便地挂载到EC2实例上作为块设备访问。而Instance Storage是和EC2实例物理连接的块设备，但它上面的数据会随着EC2实例的消亡而丢失，所以也经常被称为非持久存储服务。在使用这些与EC2紧密相关的块存储服务时，下面的一些建议可能会帮助到你。

1）充分使用Instance Storage。在AWS的很多文档中，EBS都是被重点介绍和推荐的。而且基于EBS的AMI也确实比原来基于Instance Storage的AMI有很多明显的优势（如更快的启动速度等），但这并不意味着Instance Storage就一无是处。相反，有时Instance Storage或许是更好的选择原因如下。

- Instance Storage是和EC2实例物理连接的，而EBS盘则是通过网络和EC2实例连接的。因此，Instance Storage会有更快、更稳定的访问速度。尤其是越来越多的新EC2实例（如M3系列）开始配备SSD介质的Instance Storage，这种访问速度的差距更加明显（可以在AWS官方文档中找到各种实例类型配备的Instance Storage介质类型、容量和数目）。
- Instance Storage的费用已包括在EC2实例费用之内，而EBS则要额外付费。
- Instance Storage存取更适合RAID机制以得到更高的访问吞吐量。RAID机制是存储系统用来提高数字存取性能和可靠性的常见途径。在AWS中，你既可以使用EBS盘，也可以使用Instance Storage来做RAID0。但因为EBS盘是通过网络和EC2实例连接的，即使RAID机制提高了整个存储端的吞吐量，但还受限于网络带宽的限制（测试结果甚至证明了这种限制还非常明显）。另外，AWS的大部分实例都提供了超过两块的Instance Storage存储设备，这种设计也方便大家使用RAID。

2）对Instance Storage最大的担心主要来自于它的非持久存储限制。但非持久化并不代表不能使用（其实，计算机中的CPU Cache和Memory都是非持久存储设备，只不过操作系统和硬件在管理它们），我认为Instance Storage至少适合以下两种存储。

- 存储大量的中间数据。例如，科学计算类的服务中常产生巨量的中间数据，这些数据无法完全保存在Memory中，而传回到持久存储服务既慢又没有太大价值。但它们非常适合放到Instance Storage中，因为即使出现部分丢失也没有关系，重新计算一遍就能得到同样数据，并且放在Instance Storage还可以非常快地被接下来的处理逻辑访问。因此，科学运算类服务常用它存储中间计算结果并只把最终的运算结果（通常数据量会比中间结果小很多）存回持久存储服务中（如EBS、S3等）。

- 作为数据文件的Cache服务。例如，我们的服务需要处理很多大的设计文件，并且这些文件都已在Autodesk云存储中。由于访问速度的限制无法做到每次修改都直接写回云存储中，我们就利用Instance Storage来做本地Cache。当然，在这种使用场景中，我们自己需要保证Instance Storage的数据能及时、自动同步回云存储中。整体结构如图4所示。

![Instance Storage作为Cache图](/images/2014-06-10/storage-cache.jpg)

图4 作为数据Cache的Instance Storage同步模式

利用预留IOPS的EBS盘和EBS优化的EC2实例来保证EBS的访问性能。在需要持久性块存取的情况下，EBS仍是最好的选择。但由于EBS和EC2通过网络连接，其访问速度经常受到网络忙碌情况影响。如果服务对于磁盘的访问速度非常敏感（例如，高并发的数据库访问），这个性能波动可能是致命的。为此，AWS推出了预留IOPS的EBS盘，需要额外付出一些费用来购买预留的IOPS，但它能保证毫秒级延时并能保证99.9%时间能提供至少10%以上的预留IOPS。为了保证EC2端的访问带宽，你还需要启动为EBS特别优化的EC2实例（一个AWS物理机器会运行多个EC2实例，而且每个EC2实例的I/O带宽还需要用于EBS、网络访问等多个方面。如果启动EBS特别优化的EC2实例，它会建立和EBS盘的单独访问通道，从而减少与其他实例或网络访问争抢I/O带宽的问题）。当然，EBS优化的EC2实例也需要额外的费用。一般来说，结合使用预留IOPS的EBS盘和EBS优化的EC2实例能对整个EBS访问的性能有一定的保证。
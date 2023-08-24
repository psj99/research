# Wang等人. 为云计算中的存储安全提供公开验证和数据动态

![image-20221101151126923](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221101151126923.png)

## 摘要

- 引入了第三方审计者（TPA）的概念，TPA代表云存储用户来验证存储在云中的动态数据的完整性。消除审计过程中用户的参与，实现云计算的规模经济。

- 支持动态数据操作，如文件块的修改、插入和删除，对数据动态的支持，也是实现实用性的重要一步，因为云计算中的服务并不局限于归档或备份数据。

  之前的工作只能达成两者其中之一，本文同时实现了这两者。

  我们首先确定了从之前的工作中直接扩展完全动态的数据更新的困难下和潜在的安全问题，然后展示了如何在我们的方案设计中构建一个无缝集成这两个显著特征的优雅验证方案。特别地，为了实现高效的数据动态操作，我们通过操作经典的Merkel哈希树构造，改进了可检索证明模型（PoR）。广泛的安全性和性能分析表明，该方案具有高效的和可证明的安全性。





## 1 Introduction

两个需要关注的问题：

- 问题1：TPA应该使用无块验证

- 问题2：安全的数据动态操作。对现有的PDP和PoR方案的直接扩展，以支持数据动态操作，可能会导致安全漏洞。

我们的贡献可以总结如下：

	1. 我们提出了一个具有公共可验证性的POR模型，其中无块和无状态验证同时实现。
	2. 我们为所提出的PoR构造配备了支持全动态数据操作的功能，特别是支持块插入，这是大多现有方案中缺失的。
	3. 我们通过具体的实施和与最先进的比较，证明了我们提出的建设的安全性，并证明了我们的方案的性能。



### 1.1 Related Works

1. Ateniese 等人提出的PDP模型，使用基于RSA的同态标签来审计外包数据，从而可以提供公开验证。但他们没有考虑动态数据存储的情况，而将其方案从静态数据存储直接扩展到动态情况，存在许多设计和安全问题。
2. Ateniese后续提出了一个PDP方案的动态版本，但该系统对查询的数量施加了一个先验约束，且不支持完全动态的数据操作。它只允许功能有限的非常基本的块操作，并且不支持块插入。
3. Wang等人考虑了分布式场景中的动态数据存储，所提出的挑战-响应协议既可以确定数据的正确性，也可以定位可能的错误。只考虑了对动态数据操作的部分支持。
4. Juels描述了一个PoR模型，并对他们的方案给出了一个更严格的证明。在这个模型中，抽查和纠错码用于确保档案服务系统上的数据文件的“占有性”和“可检索性”。具体来说，一些被称为“哨兵”的特殊块被随机嵌入到数据文件F中用于检测，且F会进一步被加密来保护这些特殊块的位置。然而，就像Ateniese后续提出的PDP方案的动态版本一样，客户端可以执行的查询的数量也是一个固定的先验的，并且预先计算的“哨兵”的引入阻止了实现动态数据更新的发展。此外，他们的方案也不支持公共可验证性。
5. Shacham等人设计了一种具有充分安全性证明的改进的PoR方案。他们使用由BLS签名构建的公开可验证的同态身份验证器，并在随机喻言模型下是可证明安全的。基于BLS的构造，实现了公共可检索性，证明可以聚合成一个小的身份验证器值。但作者仍然只考虑静态数据文件。
6.  Erway等人是第一个探索动态可证明数据占有（PDP）的结构。他们扩展了PDP模型，以支持对存储文件的可证明更新，通过使用基于排名的认证跳表。这个方案本质上是PDP解决方案的一个完全动态的版本。特别是，为了支持更新，特别是对于块插入，他们试图消除Ateniese的PDP模型中的“标签”计算中的索引信息。为了实现这一点，在验证过程之前，他们首先使用认证跳表的数据结构对被挑战或更新块的标签信息进行验证。然而，他们的方案的效率仍存在疑问。

可以看出，虽然现有的方案是在不同的数据存储系统下提供完整性验证，但支持公共验证和数据动态的问题尚未得到充分解决。如何实现安全高效的设计，以无缝集成数据存储服务的这两个重要组件，仍然是云计算中一项具有挑战性的开放任务。





## 2 Problem Statement

### 2.1 System Model

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221101195139436.png" alt="image-20221101195139436" style="zoom:67%;" />

云数据存储的代表性网络架构如图1所示。可以识别出三个不同的网络实体：客户端：在云中存储大量数据文件并依赖云进行数据维护和计算的实体，可以是个人消费者或组织；云存储服务器（CSS）：由云服务提供商（CSP）管理的实体，拥有大量的存储空间和计算资源来维护客户端数据；第三方审计员（TPA）：TPA具有客户不具备的专业知识和能力，可以根据要求代表客户评估和揭露云存储服务的风险。

在云范例中，通过将大型数据文件放在远程服务器上，客户端可以减轻存储和计算的负担。由于客户端不再在本地拥有他们的数据，因此确保他们的数据被正确地存储和维护是至关重要的。也就是说，客户端应该配备一定的安全手段，以便即使不存在本地副本，它们也可以定期验证远程数据的正确性。如果客户不一定有时间、可行性或资源来监控他们的数据，他们可以将监视任务委托给受信任的TPA。为了保护客户端数据的隐私，执行审计时不向TPA显示原始数据文件。在本文中，我们只考虑具有公共可验证性的验证方案：任何拥有公钥的TPA都可以作为验证器。我们假设TPA是无偏的，而服务器是不可信的。请注意，我们在本文中没有讨论数据隐私问题，因为云计算中的数据隐私主题与我们在这里研究的问题是正交的。为了实现应用程序的目的，客户端可以通过CSP与云服务器进行交互，以访问或检索其预存储的数据。更重要的是，在实际场景中，客户机可能经常对数据文件执行块级操作。在本文中，我们所考虑的这些操作的最一般的形式是修改、插入和删除。



### 2.2 Security Model



Shacham和Waters在《Compact Proofs of Retrievability》一文中提出了PoR系统。一般来说，检查方案是安全的，如果

​	（1）不存在一种多项式时间算法能够以不可忽略的概率七篇验证者；

​	（2）存在一种多项式时间提取器，它可以通过执行多次挑战-响应协议来恢复原始数据文件。根据该PoR系统的定义，客户端可以定期挑战存储服务器，以确保云数据的正确性，通过与服务器的交互可以恢复原始文件。

他们还定义了PoR方案的正确性和稳健性：

​		如果验证算法在与有效的证明者（例如，服务器返回有效响应）交互时接受了，说明方案是正确的。

​		如果任何一个欺骗客户端它存储了文件的服务器确实存储了拥有该文件，则该方案是健壮的。

请注意，在敌手和客户端之间的“游戏”中，敌手可以完全访问存储在服务器中的信息，即，对手可以扮演证明者（服务器）的角色。在验证过程中，对手的目标是成功地欺骗客户端，即尝试生成有效的响应，并在不被发现的情况下通过数据验证。

在验证过程中，我们的安全模型与原始的PoR有细微但关键的区别。

原始的PoR方案不考虑动态数据操作，并且根本不能支持块插入。这是因为签名的构造与文件索引信息i有关。因此，一旦插入了文件块，计算开销是不可接受的，因为所有相关文件块的签名都要用新的索引重新计算。

为了处理这个限制，我们在生成签名删除索引信息$i$，使用$H(m_i)$作为文件块$m_i$的标签，而不是$H(name||i)$或者$h(v||i)$，因此，在任何文件块上的个别数据操作都不会影响其他文件块。

回想一下，$H(name||i)$或者$h(v||i)$由客户端在验证过程中生成。然而，在我们的新构造中，没有数据信息的客户端没有能力计算$H(m_i)$。为了在实现无块操作的同时成功地执行验证，服务器应该接管计算$H(m_i)$的工作，然后将其返回给证明者。

这种变化的结果将导致一个严重的问题：它将给对手更多的机会，通过操纵$H(m_i)$或$m_i$来欺骗证明者。由于这种构造，我们的安全模型在验证和数据更新过程中都与原始的PoR模型有所不同。具体来说，在我们的方案中，标签应该在每个协议执行中进行身份验证，而不是由验证者计算或预先存储。（详情将见第3节）。



### 2.3 Design Goals

我们的设计目标可以被总结如下：

1. 对存储完整性的公开验证：允许任何人，而不仅仅是最初在云服务器上存储文件的客户机，都有能力按需验证存储数据的正确性；
2. 动态数据操作支持：允许客户机对数据文件执行块级操作，同时保持相同级别的数据完整性保证。设计应尽可能高效，以确保公共可验证性和动态数据操作支持的无缝集成；
3. 无块验证：为了效率和安全性的考虑，在验证过程中，验证者（例如TPA）不应检索任何被挑战的文件块。
4. 无状态验证：在整个数据存储的长期审计之间，消除在验证者侧对状态信息进行维护的需要。





## 3 The Proposed Scheme

### 3.1 Notation and Preliminaries

**Bilinear Map**

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221104200613601.png" alt="image-20221104200613601" style="zoom: 80%;" />



**Merkle Hash Tree**

默克尔哈希树是一种充分研究的身份认证结构，其目的是有效和安全地证明一组元素是没有损坏和改变的。它被构造为一个二叉树，其中MHT中的叶子是真实数据值的哈希值。虽然MHT通常用于验证数据块的值。

·下图描述了一个认证的例子。具有真实的$h_r$的验证者请求验证$\{x_2,x_7\}$的完整性。



![image-20221104201600030](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221104201600030.png)



**但是**，在本文中，我们进一步使用MHT来同时验证数据块的值和**位置**。我们把叶节点当作从左到右的序列，从而任何叶节点都可以通过遵循这个序列和在MHT中计算根节点的方法来唯一确定。

```
使用MHT确保数据节点在位置上的完整性，利用基于BLS签名的PDP机制来确保数据块内容的完整性。
动态更新操作时，在更新节点的同时，需返回辅助认证信息给用户，用户重新生成根节点的哈希值．之后，更新存储在TPA中的根哈希值。
```







### 3.2 Definition

![image-20221102151811139](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102151811139.png)

---

![image-20221102151827809](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102151827809.png)

---

![image-20221102151842967](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102151842967.png)

---

![image-20221102151853112](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102151853112.png)

---

![image-20221102151910568](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102151910568.png)

---

![image-20221102160746664](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221102160746664.png)

---

### 3.3 Our Construction

以BLS短签名为基础，构造了一个支持数据动态操作的系统。该方案在RSA构造中也可以实施。在第3.4节的讨论中，我们将展示对以前工作[2,1]的直接扩展存在安全问题，我们相信支持动态数据操作的协议设计是云存储系统的一项具有重大挑战性的主要任务。



**现在我们开始展示方案的主要思想**。

<img src="\Enabling Public Verififiability and Data Dynamics.assets\image-20221102163018454.png" alt="image-20221102163018454" style="zoom:67%;" />

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221102163059625.png" alt="image-20221102163059625" style="zoom:67%;" />





#### Setup



<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221102201849289.png" alt="image-20221102201849289" style="zoom: 67%;" />



---

#### Default Integrity Verifification

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221102202349095.png" alt="image-20221102202349095" style="zoom: 67%;" />



<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221102203924390.png" alt="image-20221102203924390" style="zoom: 67%;" />

<img src="\Enabling Public Verififiability and Data Dynamics.assets\image-20221102201818153.png" alt="image-20221102201818153" style="zoom: 67%;" />



```
敌手拥有计算H（mi）的优势，可以通过操作H（mi）和mi来欺骗验证者。例如，假设验证程序想要一次检查m1和m2的完整性。在收到挑战后，证明者可以只使用文件中两个块的任意组合来计算这对块（σ，µ）。现在，以这种方式制定的响应可以成功地通过完整性检查。因此，为了防止这种攻击，我们应该在验证之前首先对标签信息进行身份验证，即确保这些标签与要检查的块相对应。
```



---

#### Dynamic Data Operation with Integrity Assurance

现在，我们将展示了我们的方案如何显式和有效地处理完全动态的数据操作，包括用于云数据存储的数据修改(M)、数据插入(I)和数据删除(D)。

请注意，在以下关于动态操作的协议设计的描述中，我们假设文件F和签名Φ已经生成并正确地存储在服务器上。根的元数据R已由客户端签名并存储在云服务器上，因此任何拥有客户机公钥的人都可以质疑数据存储的正确性。

- **Data Modification**

我们从数据修改开始，这是云数据存储中最常用的操作之一。一个基本的数据修改操作是指用新的块替换指定的块。

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184126446.png" alt="image-20221103184126446" style="zoom: 67%;" />

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184159156.png" alt="image-20221103184159156" style="zoom:67%;" />

**Data Insertion**

数据插入，是指在数据文件F中的某些指定位置之后插入新的块。

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184426572.png" alt="image-20221103184426572" style="zoom:67%;" />



<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184510272.png" alt="image-20221103184510272" style="zoom:67%;" />

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184524733.png" alt="image-20221103184524733" style="zoom:67%;" />

**Data Deletion**

数据删除只是数据插入的相反操作。对于单块删除，是指删除指定的块并将所有块向前移动一个块。

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184728539.png" alt="image-20221103184728539" style="zoom:67%;" />

<img src=".\Enabling Public Verififiability and Data Dynamics.assets\image-20221103184743019.png" alt="image-20221103184743019" style="zoom:67%;" />

---





## 4 Security Analysis

![image-20221104211231159](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221104211231159.png)

---



![image-20221104211241979](\Enabling Public Verififiability and Data Dynamics.assets\image-20221104211241979.png)

---

![image-20221104211259060](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221104211259060.png)




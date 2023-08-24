# Shacham, Waters： Public Verifiability PoR



## 基于BLS签名的PDP机制



相比于基于BLS签名的PDP机制而言，在进行初始化阶段之前，需要增加冗余编码数据预处理过程，是数据文件具有容错能力，即将F分为n个块，然后对n个块进行分组，每个组为k个块；之后对每组数据利用Reed-Solomn纠错码进行容错编码，形成新的数据文件$\widetilde F$。

![image-20221104181333637](.\Enabling Public Verififiability and Data Dynamics.assets\image-20221104181333637.png)


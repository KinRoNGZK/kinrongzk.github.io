<center><font face="黑体" size=32>工作报告</font></center>

[TOC]

##### 主要工作

先介绍一下去年11月入职以来，这近半年的时间自己所作的主要工作。

1. 非强制引导，
2. 集结，
3. 试炼

##### 遇到问题&解决思路

overlay下z值造成cull问题。ienumator异常捕获问题，嵌套迭代器停不下来问题，寻路缓存还是有点卡问题。气泡要有先后次序问题。

##### 目前困惑

ecs是否有更好的实现方式，一种entity一个context。数据是否能细分一下不要都放在map、和player中。

```c#
EntityQueryDesc ChunksWithChunkComponentADesc = new EntityQueryDesc()
{
    All = new ComponentType[]{ComponentType.ChunkComponent<ChunkComponentA>()}
};
```

**除了上述的有点，ecs。**

##### 职业规划

渲染方向的架构师，R&D。

##### 离开莉莉丝希望不同

技术上，能做一点业界前沿的探索。项目上，参与一款成功上线的爆款。

##### 希望改变什么、如何改变

Review返工问题，UI布局是否合理、效率问题，骨骼动画烘焙结合ecs。希望某些功能能专门的人负责。

##### 过去工作更好的地方

配置类生成问题，二进制读写，单例声明周期，mono限制使用问题，update统一管理问题，寻路问题，领土划线问题，timeline，裁剪问题，去场景化，场景中的东西尽量简单，ui的次序是否可以更精确控制。UI代码生成问题。

##### 财务自由想做什么

做电影，动画电影，做中国的维塔、工业光魔。

##### 附件

[1]ECS

[2]DOTween

[3]UGUI

[4]VT、CDLOD
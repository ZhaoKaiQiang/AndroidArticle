# 如何理解Zygote和ServerManager这两个进程？

在开始介绍这两个人物之前，我先给你讲一个故事。

在地球的平行宇宙——Android宇宙里，存在着一个进程，他叫Init，在内存和硬盘开化之前，Init进程就是这个世界的上帝，除此之外，别无一物。

突然有一天，Init进程觉得偌大个内存只有自己好无聊呀，于是，他就用内存空间里的0和1作为材料，创造出一个全新的进程，为了纪念这第一个孩子，Init给他取名为zygote进程。

随着zygote慢慢的长大，他开始迎来了青春期。

“好无聊呀！”zygote这样想。“我既然是上帝之子，为什么不能像亚当那样，取出一根肋骨产生一个新的进程呢？”

于是，zygote照着自己的样子fork出一个新的进程，他给她取名为SystemService进程。

再后来，两个青梅竹马的进程慢慢的长大，zygote和SystemService撑起了Android世界的两片天，就和海马一样，生孩子这种累活由zygote承担，而SystemService则负责处理孩子们大大小小的请求。

“妈，我想打开一个Activity！”于是SystemService就安排AMS去做了。


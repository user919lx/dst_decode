# 这个系列讲什么

解读源码，用全新的视角重新认识我们熟悉的饥荒世界。我会讲解饥荒里涉及的基本编程概念，游戏运行的机制等等。

我在几年前曾经兴致勃勃地想要写一个完整的饥荒Mod教程，但后来弃坑了，主要就是写起来太过费劲，每一篇都要写上万字，查阅许多资料。当时还是个刚入行的编程菜鸟，啥都不懂，读源码也很吃力。这几年我已经成长成了一个资深程序员，对编程有了深入的理解，读源码已经是如同喝水一般轻松了。不过工作比较忙，时间紧张，也没精力再像以前一样写上万字的详细教程了。所以我就打算先写这个科普向的系列，在有限的时间精力行下，给大家带来一些对Mod制作的兴趣。这个系列虽然是以饥荒为切入点，但实际讲解的是游戏编程的知识，其中很多概念比如Prefab、Component等等，都是经典的游戏编程概念，在许多游戏中都是相通的，如果你能理解这些概念，不仅能用在其它游戏Mod的编程，甚至可以由此入门尝试编写自己的游戏。


# 谁适合阅读？

想学Mod制作的，想创建自己的地图的，想自己修改Mod的，想更好地使用控制台的，想要深入了解游戏机制的，总之，对饥荒的代码和运转机制感兴趣的一切玩家，都可以读一读。如果你对游戏编程感兴趣，饥荒的代码也是很好的学习材料。

对于想学Mod、制作自己的地图的同学，可以通过阅读这个系列，深入理解游戏的编程框架，降低阅读游戏源码的门槛。Mod制作有三个阶段：刚入门的，只能跟着教程一步步走，做出教程示例的东西，或者在这个基础上稍作调整。更进一步，在大量学习了别人写出来的教程之后，把知识串联起来，做出自己想要的东西。而最高级的Mod制作者，则是可以自己阅读游戏和其他Mod的源码，从中获取知识，进而可以在游戏框架内实现任何需求。我这个教程的目的之一，就是教大家如何成为一个最高级的Mod制作者。

即使是不打算做Mod的同学，也可以通过阅读这个教程，对游戏的机制有更本质的理解，从而更好地玩游戏。

# 会怎么讲？

首先第一期，我会介绍这个世界的构成元素，这一期不会涉及具体的代码，只是介绍各种概念，建立对世界的整体认知，为后续学习打下基础。接着我会在每一期里仔细解读一个元素，并且写一个小Mod来展示如何应用。Mod会发布在steam创意工坊上，也会发布在github上。

我之前建了一个饥荒Mod答疑QQ群 559477977，欢迎加入。

# 源码在哪里？

官方的源码被打包成了一个zip文件，位于`游戏根目录/data/databundles/scripts.zip`，解压之后即可阅读lua文件
创意工坊的Mod源码，位于`游戏根目录/mods/workshop-xxxx`，可以直接阅读。
推荐大家使用VS Code + lua拓展插件进行阅读，有代码高亮，还能以项目的形式进行浏览多个文件，比原生的文本编辑器要方便得多。

# 附加的代码
部分章节中，为了便于学习理解，我制作了小mod，可以参考我的项目
[dst_mod_tutorial](https://github.com/user919lx/dst_mod_tutorial)
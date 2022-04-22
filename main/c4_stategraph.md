## 引言

玩家在进行各种活动的时候，会播放相应的动画。有些动画可以被打断，比如玩家攻击时有个明显的抬手动作，如果在砍下去之前执行了其它操作，攻击的动画就会被中断，也不会对目标造成伤害。而另一些动画，比如进食，是不可打断的，强制处于硬直状态。另外，在动画播放到某个时间点后，还会开始播放音效。这些变化、硬直，其底层的逻辑结构就是一个StateGraph。我们可以把攻击、移动、进食等等动作各自看成是一个独立的状态(state)，在这个状态下，会播放相应的动画和音效，然后通过某些触发条件将这些state连接起来。这就构成了一个有联系的状态图(StateGraph)。一个拥有状态图的角色，在任何一刻，必定处于其中一个状态中。这样一来，角色就能够动起来了。不仅玩家如此，其它能够行动的猪人、蜘蛛也都是如此。

## StateGraph概览

StateGraph，直接翻译过来叫「状态图」，在游戏编程中有类似的概念，叫「状态机」，但这两者还是有一些区别的，状态图允许通过多种方式从任意状态跳转到指定状态，而状态机则是有固定的跳转路线。

图能带来更直观的感受，让我们来看一下状态图是怎样的：下面是我们的好伙伴猪人的状态图。

方形的结点都是一个state，每个state会自动执行一段动画，部分还会添加音效，并且会设定在特定的时间点让部分功能生效。比如attack这个state，会在进入state后的第13帧（1帧=1/30秒）触发攻击效果，这意味着猪人的攻击前摇为13帧。
两个特殊的椭圆形结点，分别是动作和事件处理器。当以某种形式触发了角色的动作或者事件，就会进入相应的处理器流程，如果当前角色的状态满足处理流程的条件，就会将角色转为对应的state。比如触发了猪人砍树的动作，那么就会检测猪人当前是否处于busy的state，如果没有，就会让猪人进入chop的state，接着就会执行砍树的动作。
在结点之间的线，就是相应的触发事件或动作。在state之间转换的线，上面的字代表进入下一个state需要触发的事件（多数都是animover，也就是动画播放结束）。如果是从动作处理器转换的，则代表着相应的触发动作。如果是从事件处理器转换的，线代表着对应事件。部分情况下，同一个事件或动作可能有多个转换的分支，具体会流转到哪，取决于分支条件。



![sg.png](/images/dst_sg.png)


然后我们再来看看代码层面，来自文件`stategraphs/SGpig.lua`

一份典型的SG代码类似如下结构。
```lua
-- 添加commonstates模块，当需要用CommonStates添加一些通用的state时
require("stategraphs/commonstates")
local actionhandlers = 
{...
}
local events = 
{...
}

local states = 
{...
}

-- 这里的CommonStates是通用类，方便快速添加一些通用的state，比如idle, frozen等
CommonStates.AddXxxStates(states,
{..
})
...

return StateGraph("pig", states, events, "idle", actionhandlers)
```

从`StateGraph`的构造参数就可以看出来，它的组成部分如下

* name:SG的名称，这里是pig，并不要求与prefab一致，因为可能有多个prefab共用一个SG
* states:所有state结点定义信息，每个结点都是一个table，记录一个状态的全过程中每个环节的操作，主要处理的内容是动画音效的播放、系统交互逻辑的瞬时生效（比如攻击命中扣血）、流程结束后的状态转换等。
* events:事件处理器集合。每一个事件处理器在触发了相应事件并满足条件的情况下，会直接将角色转换为特定状态。
* defaultstate:每次进入游戏时，默认初始化的状态
* actionhandlers:动作处理器。和事件处理器类似，但触发的是动作。

以上的states,events,actionhanlders中的元素分别为对应的类State, EventHandler, ActionHandler对象，包括StateGraph类在内，这几个类的定义代码都可以在`stategraph.lua`中查看。接下来我们逐个根据代码来做进一步的讲解。

## State的构造

State类明面上只有一个传入参数，就是args。实质上这个args默认为一个table，承载了许多参数。

* args.name:字符串，state的名称
* args.onenter:函数，在进入state时执行的代码
* args.onexit:函数，在转换state前执行的代码
* args.onupdate:函数，在系统推进更新状态时执行的代码（实质上也就是系统时间发生跳动，每1/30秒跳动一次，这样的跳动称之为1帧）
* args.ontimeout:函数，在timeout时执行的代码
* args.tags:table，其中的元素是字符串。sg标签集，仅限于sg使用，与inst的tag有区别，部分tag可通用。
* args.events:table，其中的元素是EventHandler对象。事件处理器列表，在当前状态下如果触发了相应事件，会执行事件对应的回调函数。需要注意的是只是执行函数，并不一定会跳转状态。比如走路时处罚移动的事件，可能只是播放一下走路的音效而已。
* args.timeline:table，其中的元素是TimeEvent对象。这个类很简单，第一个参数是time，表示定位在哪一帧。官方提供了常量FRAMES表示1帧的时间，通常的传入的参数是x*FRAMES，表示从进入状态开始过了x帧。第二个参数是fn，也就是在这一帧内要执行的代码，比如播放音效、触发特定效果等等。这个table不需要手动排序，系统会自动根据time的大小重新排序，保证每一帧都能按顺序执行相应代码。

代码结构大致如下
```lua
State{
    name = "xxxx",
    tags = { "a", "b", "c" },
    onenter = function(inst, item)...
    end,

    onexit = function(inst)...
    end,
    
    ontimeout = function(inst)...
    end,
    
    onupdate = function(inst)...
    end,

    timeline =
    {...
    },

    events =
    {...
    },
}
```
除了name是必须的外，其它的都可以不填，通常大多数state会有tag和onenter，其它参数根据需要使用。

仍然以猪人为例，我们来看看猪人的进食state

```lua
State{
    -- name是必填项
    name = "eat",
    -- tags中含有busy，意味着这个state在流程走完之前，通常不会被打断，除非是遇到被攻击之类的强制打断事件
    tags = { "busy" },

    -- onenter决定了进入状态时要执行什么，
    onenter = function(inst)
        -- 让角色停止移动
        inst.Physics:Stop()
        -- 角色播放进食的动画
        inst.AnimState:PlayAnimation("eat")
    end,

    -- timeline可以精准控制时间轴上的代码执行
    timeline =
    {
        -- 在第10帧，执行代码
        TimeEvent(10 * FRAMES, function(inst)
            -- 执行缓存中的动作的fn。要转换到进食这个状态，需要有进食的动作，因此这里的动作就是进食
            inst:PerformBufferedAction()
        end),
    },

    events =
    {
        -- 当动画播放完后会触发animover事件，并执行相应的回调函数
        EventHandler("animover", function(inst)
            -- 角色的状态跳转到idle
            inst.sg:GoToState("idle")
        end),
    },
},
```

## Handler的构造
Handler的中文学名叫句柄，这个翻译太难理解，我更愿意称之为处理器。Handler的作用就是，监听特定的内容，当出现该内容时，执行预先设计好的代码。在SG中有两种Hanlder，分别是EventHandler和ActionHandler。

EventHandler有两个参数，分别是要监听的事件event和相应的回调函数fn。它的作用就是指定一个要监听的事件，当监听到目标身上有触发该事件时，执行相应的函数。如果EventHandler是写在某个state的events中，则只有当角色处于在该state下才会进行监听。如果是写在StateGraph的envents中，则在任意情况下都会进行监听。触发监听的事件本身并不会直接导致状态转换，要实现状态转换，需要在回调函数中执行相应的代码`inst.sg:GoToState("xxx")`

ActionHandler则有三个参数，分别是action 触发动作, state 转换状态, condition 条件。其中第三个参数condition在实际使用中几乎没有出现，可以忽略。
根据第二个参数state的数据类型不同，ActionHandler有两种处理流程。当state是字符串时，这个参数必须是角色定义好的state之一。当角色触发了指定的action，他就会自动转换为对应的state。当state是一个函数时，函数的返回值必须是一个字符串，也得是state。在函数的形式下，就可以灵活设置不同的返回state，创造state转换的分支。



## 总结

本期主要说明在游戏中，角色的动画、音效变化是如何实现的。StateGraph不仅仅控制着动画、音效的转变，还决定着具体功能逻辑执行的确切时间点。最容易理解的概念就是人物的攻击前摇，是由相应的attack state的timeline来决定攻击生效的具体时间点的。在简单的Mod中，通常都不会涉及SG相关的内容。但如果你想要更进一步，创造新的动作表现形式，或者修改人物的攻击频率等等，就必须深入了解SG，学习如何构建state和handler。

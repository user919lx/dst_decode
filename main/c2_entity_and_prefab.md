## 引言

entity，中文一般称为实体，指的是在游戏中出现的一切看得见和看不见的物体。比如控制世界变化的TheWorld，就是一个看不见的实体，随处可见的花草，也是实体。每个实体经由系统生成，会分配一个游戏全局唯一的ID，称为GUID(GLOBAL Unique ID)。
创建实体没有任何门槛，在创建之后你也可以随意改造。同样的，也可以改造一个已有的实体，你甚至可以把一个猪人变成蜘蛛。如果我希望生成一只蜘蛛，首先是创建一个实体，然后给它添加蜘蛛的动画，再加上蜘蛛具备的一些能力，添加SG和brain。但是，如果每次在需要生成某一类entity（比如蜘蛛）的时候，都要写上大段的代码（下文简称为初始化代码），就很不方便。而且也不利于后续维护修改，比如要调整蜘蛛的属性，就需要在每一个生成代码里都进行修改。
为了解决这个问题，最直接的想法是，把初始化代码写成一个初始化函数，每次要生成蜘蛛的时候，就调用这个函数。这样一来，无论是使用还是维护，都方便了许多。但仍然存在问题：如果要生成的entity类别很多，有几百个，那就得写几百个函数，这些函数名要维护起来也会有点麻烦。另外，entity还需要关联动画、音效等资源，如果这部分代码写在创建函数里，也不太合适。
更进一步的方案是，制作一个模板，存放所有能体现这个entity独特性的东西，包括各种资源和初始化代码，并且为这个模板取一个唯一的名字，然后把这个名字注册到系统中。然后构建一个生成函数，可以根据模板信息自动生成相应的实体。当系统需要生成这一类实体的时候，直接根据名字找到模板来生成就可以了，这就是prefab的由来。这是个经典的游戏编程概念，中文名一般称为预制物或预制体。

prefab与entity的区别在于，entity是实实在在会占用大量系统资源，每一个entity都是独立存储计算，有自己的数据，数量越多，消耗的资源越多。而prefab只是个模板，仅仅占用很小的固定资源，在游戏过程中不会发生变化。
对编程有基本了解的同学，都知道class这个经典的概念。class与prefab之间也有区别。prefab一经定义，就无法在游戏运行中再做修改。而在许多解释型语言中，都提供了在运行中动态修改class的能力。

简而言之，各种各样不同的Prefab作为模板，根据需要生成了各种entity，从而构成了丰富多彩的游戏世界。



本篇会从源码的角度，剖析prefab是如何生成的。


**注：下文涉及代码的地方都会给出文件的相对路径，默认根目录是游戏的代码文件夹，也就是scripts**

## prefab的定义

所有的prefab都通过Prefab类来构建，实质上是Prefab类的一个实例。
Prefab类的定义代码写在`prefabs.lua`文件里，完整的构建函数形如`Prefab(name, fn, assets, deps, force_path_search)`，这几个参数的含义如下
* name: prefab的名称，必须是唯一的，不能与其它prefab的名称重复。这个名称会注册到系统里，可以在游戏控制台中通过c_spawn("{name}")来生成对应的prefab
* fn: prefab的初始化函数，当系统生成相应prefab的实体时，会执行这个初始化函数，生成对应的prefab。不同prefab的区别就在于这个初始化函数的不同
* assets: 相关的动画、音效等静态资源文件的路径
* deps: 相关的Prefab依赖
* force_path_search: 强制搜索路径

大多数情况下，都只会用到name, fn, assets这三个参数，deps和force_path_search一般是可以忽略的。

而在这其中，最重要的就是初始化函数fn，它描述了一个Prefab在生成时应该如何设置，比如使用何种动画骨架，最开始的外形如何，是否能发起攻击等等，如果有HP的话，初始化的血量是多少等等。如果你想了解一个物体的详细参数，最直接有效的方法就是找到这个物体的Prefab的初始化函数，仔细阅读。


## 初始化函数fn

初始化函数的代码，在复杂的Prefab中通常比较庞杂，像生成人物的代码就有足足1000多行，如果只是单纯地逐行阅读，是比较吃力的。事实上，代码的编写是有一定的思路的，总的来说，可以把其中的内容分类为以下几块。

* 创建实体：提供一个实体，后续的一切操作都围绕着这个实体来进行
* 系统底层组件：添加基本功能组件，这些组件的源码无法看到，仅暴露可调用的函数
* tag：打上标签，主要用于对Prefab进行分类，方便系统对特定类的Prefab进行操作
* StateGraph和Brain：在Prefab的初始化中一般分别只有一句，用于设置对应的StateGraph和Brain
* Component：添加组件，与系统底层组件的区别在于可以看到源码。
* 监听：设置对事件或者世界状态（比如是否月圆之夜，白天还是晚上）的监控，当触发特定事件或者进入某个状态后，执行相应的操作
* 其它设置：其它个性化的配置

除了必须以「创建实体」作为第一句代码外，其它的部分一般来说并没有严格的顺序限制。要加什么，完全取决于你希望这个Prefab长什么样，能做什么，在这一点上是非常自由的。在联机版中，客机的数据需要从主机同步，许多场景的数据，比如人物的血量、饱食度等，都是来自Component的。因此，在客机中，Component是不需要添加的，即使强行添加，也不会被使用。类似的还有StateGraph和Brain。因此，在联机版的Prefab定义中，通常会有以下这段代码

```lua
if not TheWorld.ismastersim then
    return inst
end
```

这段代码的意思是，检查当前系统环境是否是主机。如果不是主机，就到此为止，不再执行后面的代码。在这段代码之前的部分，是主客机通用的，通常只包含创建实体，添加系统底层组件，添加tag。

虽然代码的编写顺序没有严格的限制，但官方的代码多数是按照上面的顺序来编写的，其中添加StateGraph和Brain的代码通常只有一句，通常写在component的部分之前。有一个例外是locomotor，根据官方代码给出的注释，这个component需要在添加StateGraph之前就添加。

联机版的典型代码的示例如下

```lua
local function fn()
    -- 创建实体
    local inst = CreateEntity()
    
    -- 添加系统底层组件
    inst.entity:AddTransform()
    ...
    
    -- 添加tag
    inst:AddTag("xxx")
    ...
    
    -- 主客机分割代码
    if not TheWorld.ismastersim then
        return inst
    end
    
    -- 添加locomotor组件，并设置速度
    inst:AddComponent("locomotor")
    inst.components.locomotor.runspeed = xxx
    
    -- 添加StateGraph
    inst:SetStateGraph("SGxxx")
    
    -- 添加Brain
    inst:SetBrain(brain)
    
    -- 添加其它Component并设置初始化参数
    inst:AddComponent("xxxx")
    inst.components.xxx.xattr = "xxx"
    ...
    
    
    -- 注册监听和回调
    inst:ListenForEvent("event_name", callback_fn)
    ...
    
    -- 其它设置，通常是封装好的一系列预处理内容
    MakeHauntablePanic(inst)
    ...
    
    return inst
end

```


下面就分块来讲解这些代码，这里选用的示例Prefab是**rabbit**，代码在`prefabs/rabbit.lua`。它不像猪人、蜘蛛那样有多种形态写在同一个Prefab文件里，整体代码结构比较清晰，以上讲到的部分都有涉及，代码量适中（整个脚本共300～400行），是很好的学习参考对象。


### 创建实体

实体创建只有一行代码：`local inst = CreateEntity()`，由此得到一个inst变量，它代表着Prefab初始化的实体，后续的所有操作，无论是添加组件还是tag，都需要围绕这个实体来进行。

在这个部分，顺带讲讲`CreateEntity`的具体执行内容和`inst`对应的类

`CreateEntity`是定义在lua脚本代码里的，可以在`mainfunctions.lua`下面找到，定义如下

```lua
function CreateEntity()
    local ent = TheSim:CreateEntity()
    local guid = ent:GetGUID()
    local scr = EntityScript(ent)
    Ents[guid] = scr
    NumEnts = NumEnts + 1
    return scr
end
```

`EntityScript`这个类的定义，则可以在`entityscript.lua`里找到，上面初始化构建`EntityScript`的实例时传入的ent,可以通过`inst.entity调用`。

根据上面代码可以看到，我们得到的inst实际上是用lua脚本定义出来的`EntityScript`类的一个实例，对真正实体的一个封装。真正的实体是通过系统底层的函数`TheSim:CreateEntity`来创建的。当我们要使用系统底层的组件时，必须通过这个真正的实体去调用。因此，在添加底层组件的时候，使用的是这样的的格式:`inst.entity:AddXXX()`，这里的`inst.entity`就是真正的实体。

为什么会做这样的设计呢？这是因为系统底层返回的entity所拥有的函数是比较有限的，要添加或者修改都比较麻烦。在进行entity层面上的修改时，会很不方便。比如添加Component这个工作：Component是直接用lua定义的，自然地，要添加Component也需要通过lua来完成。这中间会涉及到一些重复的，相对有点复杂度的处理，会希望抽象提取成一个函数，也就是`AddComponent`。希望能通过inst直接调用这个函数，而系统底层返回的entity又不方便直接修改。因此，使用`EntityScript`来做一层封装，在`EntityScript`这个类中定义`AddComponent`,从而实现`inst:AddComponent('xxx')`这样简洁的调用。


### 系统底层组件

系统底层组件，指的是那些和Component有类似用法，但定义代码被封装编译在游戏引擎内部，我们无法看到具体源码的部分。相比于数量接近400个的Component，在官方代码中出现的系统底层组件较少，只有近30个，其中被大量使用的也不到10个。使用底层组件而不是Component的原因可能有很多，每一个决策都有其依据。比较常见的理由是，组件需要非常频繁调用底层系统的其它资源，比如控制动画、声音的播放，或者控制entity的位置变化等，这样的情况下，封装在底层系统，可以有效地提升游戏性能。

在这个部分，代码可以再细分成两部分：添加组件和设置初始化参数。

具体代码示例如下

```lua
inst.entity:AddTransform()
inst.entity:AddAnimState()
inst.entity:AddSoundEmitter()
inst.entity:AddDynamicShadow()
inst.entity:AddNetwork()
inst.entity:AddLightWatcher()

MakeCharacterPhysics(inst, 1, 0.5)

inst.DynamicShadow:SetSize(1, .75)
inst.Transform:SetFourFaced()

inst.AnimState:SetBank("rabbit")
inst.AnimState:SetBuild("rabbit_build")
inst.AnimState:PlayAnimation("idle")
```

添加组件的代码比较简单，每个组件单独添加，格式统一为`inst.entity:Add{组件名}()`

然后是设置初始化参数，格式统一为`inst.{组件名}:调用函数(参数列表...)`。比如`inst.DynamicShadow:SetSize(1, .75)`，就是调用了在设置「动态影子」组件`DynamicShadow`，使用`SetSize`函数来设置影子的大小。

并不是所有的组件都需要做初始化。比如Network组件，是负责主客机通信的，只有在主客机之间发生数据交流时才被使用，不需要设置初始化参数。

`MakeCharacterPhysics(inst, 1, 0.5)`这一句代码，实际上是一个封装好的Physics组件初始化函数，代码可以在`standardcomponents.lua`下找到。这个函数的含义是为entity添加一个Physics组件，并设置一系列的初始化参数，构成一个生物的物理效果。

问题在于，看不到源码的情况下，怎么确定都有哪些系统组件可用，每个组件都有哪些函数呢？如果完全没有其它内部资料，唯一的手段就是从官方源码来学习。通过对官方源码的全方位分析，可以发现共使用了29个不同的底层组件， 其中调用添加的次数在20以上的常用组件如下，了解这些组件如何使用就可以了。

* Transform: 控制空间位置变换
* AnimState: 控制动画播放
* Physics: 物理引擎，控制碰撞体形状和大小，以及运动状态等
* Network: 控制网络数据传输
* SoundEmitter: 控制声音播放
* MiniMapEntity: 小地图图标
* DynamicShadow: 控制影子
* Light: 控制光照
* LightWatcher: 监控光照强度

其中物理引擎的部分与其它组建不同，具有标准化的特点：同一类物体都用同一套初始化参数，比如`MakeCharacterPhysics`不仅仅用在rabbit上，也用在玩家角色、猪人和鱼人上。因此，一般都会使用官方自己写好的标准组件函数来完成初始化，而不是一条条地调用Physics组件函数。

系统底层组件，以及下面会谈到的Component，是Prefab里最主要的组成部分，它们决定了一个Prefab有什么功能，每个功能具体如何与系统进行交互。要详细列出来每个组件的用法，篇幅会比较巨大，因此这部分内容会单独写一篇文章来进行介绍。

### Tag

tag，其实就是一个标记，用于区分一类Prefab，让系统能够针对性地对这一类Prefab作出反应。比如在人物的初始化函数里，都会添加「player」tag，这样在进行一些判定的时候，就可以通过检测这个标记，来有效地筛选出属于玩家的实体。举例来说，玩家死后变成幽灵，丢失「player」，换成了「playerghost」，与世界的互动方式也会变成「作祟」。这就是通过检测tag来区分给出不同的动作。类似的还有蜘蛛不会主动攻击韦伯，是因为蜘蛛不会对有「spider」tag的生物发起攻击。

在做Prefab初始化的时候，主要是根据需要添加相应的tag，可以通过`inst:AddTag('xxx')`来添加。

rabbit的tag相关代码如下

```lua
inst:AddTag("animal")
inst:AddTag("prey")
inst:AddTag("rabbit")
inst:AddTag("smallcreature")
inst:AddTag("canbetrapped")
inst:AddTag("cattoy")
inst:AddTag("catfood")

inst:AddTag("cookable")
```

一般来说，如果自己做Mod，有定位类似官方生物的，比如我的《Samansha》人物Mod里的鹿，定位和羚羊相似，则应该添加相同的tag。官方给出的许多tag，都能从名字上直观地看出来是有何含义，会用在什么场景，尽量保持Tag一致，有利于在游戏中取得一致的系统反馈。

到tag为止，往下的代码都只在主机中执行。


### StateGraph和Brain

关于SG和Brain，在第一篇中已有简略介绍，本篇重点也不在于此，因此不做过多的重复。
在Prefab的初始化中，与SG和Brain相关的部分就是为Prefab设置SG和Brain，代码非常简单，如下几句

```lua
local brain = require("brains/rabbitbrain")

inst:SetStateGraph("SGrabbit")
inst:SetBrain(brain)
```

一般SG和Brain视情况添加。

* SG：如果所构建的Prefab有较复杂的动画转换，或者就是和某个官方生物使用同一套动画骨架，则需要设置SG。
* Brain：如果希望这个Prefab有一定复杂度的自主智能，能自动判断什么情况下做什么，则应该添加Brain。

### Component

和系统底层组件类似，区别在于定义的代码都是可以直接在`components`文件夹下的同名lua文件看到。Component相比于底层组件，数量多得多，有将近400个，为Prefab提供了丰富多样的功能。

添加Component的代码形如`inst:AddComponent("{component_name}")`，调用相应的组件函数则是形如`inst.components.{component_name}:{fn_name}(args)`


比如让rabbit可以吃东西，就写了以下代码
```lua
inst:AddComponent("eater")
inst.components.eater:SetDiet({ FOODTYPE.VEGGIE }, { FOODTYPE.VEGGIE })
```
这两句代码的意思是，添加`eater`这个Component，让rabbit具备吃食物的能力。然后通过`SetDiet`函数来设置rabbit能吃哪些种类的食物。

Component这部分的代码，参考官方的写法，每添加一个Component就接着写相应的初始化，这一点与系统组件先添加全部再初始化不同。这是因为Component的组件数量很多，而且更改也相对频繁，将添加的代码与初始化代码放在一起，更方便管理。

在rabbit中，总共添加了如下几个Component
* locomotor：提供移动能力
* eater: 提供吃东西的能力
* inventoryitem: 可以装进玩家的物品栏
* sanityaura: 可以影响周围生物的sanity
* cookable: 可以用于烹饪
* knownlocations: 能够记住位置，这里主要是用于能够找到自己家的洞穴
* health: 拥有血量
* lootdropper: 会掉落物品
* combat: 可以战斗
* inspectable: 可以被检查
* sleeper: 可以睡觉
* tradable: 可以用于交易
* burnable: 会着火
* freezable: 会结冰
* hauntable: 可以被作祟
* perishable: 会腐坏

因为内容过多，不会讲解每个component的细节，主要谈谈以下几个函数，他们都是来自`standardcomponents.lua`，可以通过这一个函数来完成相应目的的配置。推荐在遇到类似的需求时，使用官方提供的这些函数，可以避免因版本变动引起的bug。

* `MakeSmallBurnableCharacter`: 能着火，适用于小型生物，类似的还有`MakeMediumBurnableCharacter`,`MakeLargeBurnableCharacter`，分别是和中型和大型生物，下面的Freezable也类似
* `MakeTinyFreezableCharacter`: 能结冰
* `MakeHauntablePanic`: 能被作祟，被作祟后会逃开
* `MakeFeedableSmallLivestock`: 能被喂食，并且长期不喂食会死掉


### 监听

所谓监听，就是当满足特定条件的时候，执行某些动作。比如rabbit到了冬天会换成白色的冬兔外貌，被攻击了之后会迅速逃跑到最近的洞穴里，这些变化都是通过监听做到的。

监听的实现方式多种多样，通常来说，只需要一个合适的触发器(Trigger)，以及设定一个在满足条件触发时要执行的回调函数(Callback Function)即可。

在Prefab的初始化定有中，常用的触发器有两个：

* 对世界状态的监听: WatchWorldState，当世界进入某一个特定的状态时，执行回调函数
* 对自身事件的监听: ListenForEvent，当自己触发了某个事件时，执行回调函数


在rabbit中的实际应用如下

#### 对世界状态的监听

`WatchWorldState`的代码链路比较深，可以这样找到：初始化函数->OnInit->OnWake

这段代码的含义是，监听世界的状态是否是冬天。如果是冬天，会切换成冬兔形态，如果是其它季节，会切换成普通兔子的形态。

```lua
local function OnIsWinter(inst, iswinter)
    if inst.task ~= nil then
        inst.task:Cancel()
        inst.task = nil
    end
    if iswinter then
        if not IsWinterRabbit(inst) then
            inst.task = inst:DoTaskInTime(math.random() * .5, BecomeWinterRabbit)
        end
    elseif IsWinterRabbit(inst) then
        inst.task = inst:DoTaskInTime(math.random() * .5, BecomeRabbit)
    end
end

inst:WatchWorldState("iswinter", OnIsWinter)
```



#### 对自身事件的监听

通过`inst:ListenForEvent`来设定事件触发器，第一个参数是触发的事件，第二个参数是回调函数。
这串代码的含义是当rabbit自身被攻击的时候，会触发一个attacked事件，然后执行这个回调函数，rabbit就会往自己家的洞穴跑。如果距离自己家太远，找不到回家的路的话，就不会再跑了。

```lua
local function OnAttacked(inst, data)
    local x, y, z = inst.Transform:GetWorldPosition()
    local ents = TheSim:FindEntities(x, y, z, 30, { "rabbit" }, { "INLIMBO" })
    local maxnum = 5
    for i, v in ipairs(ents) do
        v:PushEvent("gohome")
        if i >= maxnum then
            break
        end
    end
end

inst:ListenForEvent("attacked", OnAttacked)
```


### 其它设置

其它所有不归属于上面几个模块的都归属于此，这个部分就是各种个性化的处理了，每个Prefab都大不相同。此处介绍一点技巧：`inst:DoTaskInTime`在初始化中的应用。

在很多Prefab的初始化函数里，都可以找到类似的语句: `inst:DoTaskInTime(0, OnInit)`，这是为实体做特定的初始化，OnInit就是初始化函数。之所以会这样写，而不是直接执行`OnInit`，是因为经过`DoTaskInTime`的封装，`OnInit`可以在生成Entity，完成初始化后再执行。有人可能会说，把`OnInit`放在最后执行就可以了。实际上并非如此，在执行完初始化函数之后，还有一些后续处理，才算真正完成了初始化。这样的操作可以保证某些参数设置绝对完整，比如说不会出现要调用的Component没有添加的情况。如果有某些特别的初始化需求，可以仿照这样的操作来进行。比如rabbit里的这个初始化函数，在生成后会判断是冬天还是夏天，如果是冬天，会等5秒后变为冬兔，这个过程需要执行一个等待5秒的task，如果在初始化函数中设置，可能会因为各种原因导致无法执行。




## 结语

prefab的主要目的就是提供生成entity的模板，其中最重要的部分是初始化函数。初始化函数中的代码可以分为这几个部分：创建实体、系统底层组件、tag、SG和Brain、Component、监听和其它设置。希望本篇文章能够为各位同学在阅读Prefab代码时提供一些思路。下期会讲解饥荒游戏中最核心的部分——组件，包括系统组件和Component。Prefab解释了各种物体是如何构建的，组件则为这些物体提供了丰富多样的功能。

























## 引言

在开始本篇内容之前，先来思考一个小问题：抛开代码，仅仅从游戏本身的视角来看，猪人与玩家有什么相同点和不同点？相同点：都有血量、都能移动和战斗、都能戴帽子、都能吃东西等等；不同点：猪人没有精神值，可以徒手砍树，无法装备武器等等。这些相同点和不同点，都是一些属性或者功能，这就引出了今天要介绍的内容：如何为entity添加各种功能，又如何让这些功能与游戏世界里的其它entity进行交互，产生影响。要解答这些问题，就需要弄清楚两个概念：组件和动作。


## 定义
「吃东西」是一个很基础的能力，玩家都能吃东西，猪人、蜘蛛等生物也可以。我们就以「吃东西」为例子来进行讲解。一个完整的进食流程如下

![吃东西.png](https://s2.loli.net/2022/03/12/CyYxGXHIPQO2zjN.png)


首先，通过某种方式，触发动作。这些方式可以抽象出来，称为「动作触发器」。触发器有很多，只需要其中一个满足条件触发就可以进入下一环节。

触发之后，会进行动作判定和执行环节。在action里。每一个动作都有一个自己的执行函数，记作fn，用于调用相应的组件来执行功能，比如在「吃东西」这个流程中，会调用吃东西的生物的eater组件，执行Eat函数，进食带来的效果，比如提升饱食度或者血量，也会在这个函数中执行。最后会返回进食执行的结果——成功还是失败。根据结果不同，可能会执行不同的后续操作。比如如果执行成功，生物会进入进食状态，播放相应的进食动画等等。

相关代码片段如下，位于`actions.lua`

这段代码的意思是，会检测要吃的东西和执行吃的动作的生物是否分别具备相应的组件`edible`和`eater`，如果符合要求则执行`eater`的`Eat`函数。`soul`和`souleater`的判定属于wortox特有，是另一种形式的进食，处理逻辑是相似的，这里略过。

```lua
ACTIONS.EAT.fn = function(act)
    local obj = act.target or act.invobject
    if obj ~= nil then
        -- 要吃的东西得有edible组件，要进食的生物得有eater组件
        if obj.components.edible ~= nil and act.doer.components.eater ~= nil then 
            return act.doer.components.eater:Eat(obj, act.doer)
        -- wortox 特有的进食方式
        elseif obj.components.soul ~= nil and act.doer.components.souleater ~= nil then
            return act.doer.components.souleater:EatSoul(obj)
        end
    end
end
```

`eater`的`Eat`函数代码如下，它详细定义了要吃东西的各种细节。因为代码较长，以注释的形式来逐行解读。

```lua
function Eater:Eat(food, feeder)
    -- food指食物，feeder为喂食者，如果喂食者不存在，则设置为进食者
    feeder = feeder or self.inst
    -- 判断是否可以进食，比如女武神不能吃蔬菜就在这一步判断
    if self:PrefersToEat(food) then
        -- 记录堆叠数量，eater的eatwholestack属性设置为True的话，会一次吃掉所有的食物，这会出现在诸如春、秋季BOSS身上。
        -- 常规情况下，eatwholestack=False，一次只会吃一个食物。如果你希望打造一个「贪吃」特色的人物，可以把eatwholestack设置为True，每次进食都会吃完整个堆叠的食物。
        local stack_mult = self.eatwholestack and food.components.stackable ~= nil and food.components.stackable:StackSize() or 1
        
        -- 下面这一段和woody吃木头有关系，吃木头只会加木头值，不影响其它属性
        local iswoodiness = false
        if self.inst.components.beaverness ~= nil then
            -- 获取木头制增量
            local delta = food.components.edible:GetWoodiness(self.inst)
            if delta ~= 0 then
                -- 调用beaverness组件来增加相应的木头值
                self.inst.components.beaverness:DoDelta(delta * stack_mult)
                iswoodiness = true
            end
        end

        -- 此段是真正的进食影响三项基本属性的代码，如果是吃木头则略过判定
        if not iswoodiness then
            -- 计算进食效果倍率，默认为1
            local base_mult = self.inst.components.foodmemory ~= nil and self.inst.components.foodmemory:GetFoodMultiplier(food.prefab) or 1
            -- 执行health变化代码。如果食物会扣血的，还需要检测进食者+食物是否符合扣血要求
            -- health,hunger和sanity的计算方法都类似，下面的代码不再额外说明
            if self.inst.components.health ~= nil and
                (food.components.edible.healthvalue >= 0 or self:DoFoodEffects(food)) then
                -- 实际变化量 = 食物基础变化量 * 进食效果倍率 * 进食者的吸收率
                local delta = food.components.edible:GetHealth(self.inst) * base_mult * self.healthabsorption
                if delta ~= 0 then
                    -- 执行实际的血量变化
                    self.inst.components.health:DoDelta(delta * stack_mult, nil, food.prefab)
                end
            end
            -- 执行hunger变化代码，类似health的过程
            if self.inst.components.hunger ~= nil then
                local delta = food.components.edible:GetHunger(self.inst) * base_mult * self.hungerabsorption
                if delta ~= 0 then
                    self.inst.components.hunger:DoDelta(delta * stack_mult)
                end
            end
            -- 执行sanity变化代码，类似health的过程
            if self.inst.components.sanity ~= nil and
                (food.components.edible.sanityvalue >= 0 or self:DoFoodEffects(food)) then
                local delta = food.components.edible:GetSanity(self.inst) * base_mult * self.sanityabsorption
                if delta ~= 0 then
                    self.inst.components.sanity:DoDelta(delta * stack_mult)
                end
            end
        end
        
        -- 如果有喂食者，则触发喂食者相关的细节，这里是触发了一个feedincontainer事件，后续处理由事件的监听回调完成。
        if feeder ~= self.inst and self.inst.components.inventoryitem ~= nil then
            local owner = self.inst.components.inventoryitem:GetGrandOwner()
            if owner ~= nil and
                (   owner == feeder
                    or (owner.components.container ~= nil and
                        owner.components.container.opener == feeder)    ) then
                feeder:PushEvent("feedincontainer")
            end
        end

        -- 进食者触发oneat事件
        self.inst:PushEvent("oneat", { food = food, feeder = feeder })
        
        -- 如果有设置进食者的进食回调函数，则会执行。
        if self.oneatfn ~= nil then
            self.oneatfn(self.inst, food)
        end
        
         -- 如果有设置食物的进食回调函数，则会执行。
        if food.components.edible ~= nil then
            food.components.edible:OnEaten(self.inst)
        end
        
        -- 吃掉的食物在此处会被移除
        if food:IsValid() then --might get removed in OnEaten...
            if not self.eatwholestack and food.components.stackable ~= nil then
                food.components.stackable:Get():Remove()
            else
                food:Remove()
            end
        end
        
        -- 记录进食时间
        self.lasteattime = GetTime()
        if self.inst.components.foodmemory ~= nil then
            self.inst.components.foodmemory:RememberFood(food.prefab)
        end
        
        -- 返回True，说明进食动作顺利完成了，会影响到后续的处理
        return true
    end
end
```

执行完进食的过程后，会在最后返回进食是否成功，继而执行不同的后续操作。比如进食成功会播放相应的进食动画，失败了的话，人物就会说话提示。

整个动作的过程，可以抽象出三个模块，分别是
* component-组件：记录属性，提供方法。它定义了一个方法/功能与系统的逻辑交互细节，角色的数值变化和系统交互主要由component完成。
* action-动作：判定动作是否能执行，并调用相应的component来完成。它决定了一个动作该如何调用component与环境发生交互。
* 动作触发器：通过任意动作触发器，让实体执行动作过程。

接下来分模块进行解读

### component-组件

先来给出组件的定义：**与特定的功能主题具有高度关联性的一套行为和属性的集合**。

在代码上，component被定义成一个class，不同的component属于不同的class。每个component都可以围绕着这个component的功能主题来定义的一系列属性和方法。

一个典型的component定义代码如下

```lua
-- 这个是下文中会用到的处理属性变化的函数
local function oncaneat(xxx)...
end

-- 定义组件名称，Class的三个参数分别为构建函数、父类（一般留空，component不继承）、属性变化处理的table
local Eater = Class(function(self, inst)
    self.inst = inst
    -- 各项属性初始化设置
    self.eater = false
    ... 
    self.sanityabsorption = 1
end,
nil,
-- component属性变化时，client端的处理。这涉及到主客机网络编程的问题，暂时不谈
{
    caneat = oncaneat,
})
-- 定义组件的方法
function Eater:Eat(food, feeder) ...
end


function Eater:OnRemoveFromEntity() ...
end


function Eater:SetDiet(caneat, preferseating) ...
end

return Eater
```

以`eater`这个component来做说明。除了「Eat」以外，还提供了其它的操作。比如设定能吃的食物类型、设定进食时产生的额外效果等等，我们把这些统称为「行为」，在代码中则是设置相应的函数。还可以设置各项属性，如食物的吸收效率，进食时的特殊效果等。通过设置各项属性，可以让同一个功能在不同的prefab上产生不同的效果。比如每个人物的可食用物品不同，产生的食物效果不同等等。

`eater`是一个典型的侧重「行为」的component，也就是说它的函数操作主要用于描述一个「行为」产生的效果，比如进食会调用饱食度的组件hunger，让hunger来执行饱食度的提升。而我们很熟悉的饱食度、精神值等，也有相应的组件`hunger`,`sanity`，是侧重「属性」的组件，它们的函数操作主要用于描述组件内属性的变化，比如通过函数直接更改角色的饱食度。一个组件可以全是行为，也可以全是属性，甚至仅仅是一个空壳都可以，只要是合理抽象而且能与世界交互就行。比如定义一个「无情铁手」的组件，希望拥有这个组件的人物可以徒手伐木和碎石。那么，不需要为组件设置任何属性和方法。只要定义好对应的动作和触发器，就可以实现这一过程。待讲完动作和触发器后，再来说明一下这个小功能该如何实现。

在上一篇我们也提到了系统底层组件，它的使用模式和功能定位和component很像，所以也可以归为组件，作为基础功能，被广泛嵌套应用在各种代码中。与component的区别在于，系统底层组件封装在游戏底层引擎中，无法看到定义的源码，要使用和理解这些组件，只能学习参考官方源码中使用了相关组件的代码。另外，它们本身无法关联到相应的动作，也不能直接获取属性，只能调用函数。

### action-动作

action定义了如何调用组件进行互动，以及决定了如何算是动作执行成功或失败。在实际的代码中，一般会先通过`Action`类构建一个具体的动作，传入一些动作的相关参数，比如优先级、执行距离、骑乘/幽灵状态是否可用等等。然后再设置它的执行函数fn，这个函数中的代码会描述如何调用component来完成动作流程。

代码参考如下，位于`actions.lua` 所有的action都会写进一个全局变量的table:`ACTIONS`

```lua
ACTIONS =
{
    ...
    EAT = Action({ mount_valid=true })
    ...
}

ACTIONS.EAT.fn = function(act) ...
end
```

### 动作触发器

前面提到，动作触发器有很多。粗略分类可以分为两种，一是由玩家通过游戏界面触发，比如右键点击采花，二是由生物AI自行触发。本篇主要讲解一，至于二的部分，留待以后讲解生物AI时再一并说明。

动作可以通过玩家的游戏界面触发，比如装备上了斧头后能够右键砍树，或者右键点击物品栏中的食物就能自动进食。这一点依赖于动作触发器-ComponentAction。它把component和action联系了起来：将一个component与特定场景关联，在执行函数中设置满足某些条件后允许角色执行某个动作，那么角色就可以执行这个动作。注意有时候可能会出现同时满足多个动作的执行条件，这时候会按照优先级取最高的来执行。

系统定义的场景有5个，分别使用不同的参数

* SCENE：参数inst, doer, actions, right。这一场景指的是在游戏主界面上对着实体的操作。比如右键点击收获浆果。
* USEITEM：参数inst, doer, target, actions, right。这一场景是选取一件物品，再点击地图上的东西或装备栏的物品，比如给篝火添加燃料
* POINT：参数inst, doer, pos, actions, right。这一场景指的是对地图上任意一点执行的操作，比如装备传送法杖后，你可以右键点击地板，传送过去。
* EQUIPPED：参数inst, doer, target, actions, right。这一场景指的是装备了一件物品后，可以实施的操作，比如装备斧头后可以砍树。
* INVENTORY：参数inst, doer, actions, right。这一场景是点击物品栏执行的操作。比如右键点击物品栏里的木甲，就会自动装备到身上。
* ISVALID：参数inst, action, right。这个不是定义的场景，是用于检测动作是否合法的，我们可以忽略它。

以上场景各自的操作是不同的，一个组件可以在每个场景中分别定义其执行函数，并连接不同的动作。比如，edible这个组件是用于表明一个物品是可食用的。这个可食用的物品，可以选取后点击地图上的鸟来喂鸟，也可以在物品栏里直接点击右键食用，同一个组件分别绑定了USEITEM和INVENTORY两个不同的场景。

上面几个场景都有参数，同名的参数含义是一样的，这些参数会传入执行函数中。

* inst: 指拥有对应组件的物品。当场景参数没有target时，一般指的就是鼠标点击的目标。当场景参数有target时，在USEITEM中指的是你选取的物品，在EQUIPPED中指的是你装备的物品。
* doer: 指执行动作的角色，通常是玩家
* actions：指动作序列，在执行函数中，要添加一个动作XXX，就是执行代码`table.insert(actions, ACTIONS.XXX)`
* target：指执行动作的目标，比如给篝火添加燃料时，篝火就是target。这个target既可以是地图上的某个东西，也可以是物品栏里的某个物品
* pos：指地图上某一点，比如传送法杖指定的传送位置。
* right：是否使用右键来执行动作，如果要使用右键，这个值为True，只有当玩家点击右键时才会执行。如果使用左键，则可以在执行函数的定义中省略这个传入参数。


让我们来看看代码是怎么写的。ComponentAction的定义代码位于`componentactions.lua`中的`COMPONENT_ACTIONS`变量。它定义了官方的所有场景、组件与动作的连接。这里主要是展示不同的场景下，组件如何连接动作的，关键在于了解原理，所以会选择几个执行函数比较简单的组件。

```lua
...
local COMPONENT_ACTIONS =
{
    -- 注意每个场景传入参数的数量和顺序都是固定的，只有最后一个right参数可以看情况省略。请参考每个场景里，官方添加的args注释，
    SCENE = --args: inst, doer, actions, right
    {
        ...
        -- 蜂箱、蘑菇农场都有这个组件
        harvestable = function(inst, doer, actions)
            -- 检测目标是否拥有harvestable标签
            if inst:HasTag("harvestable") then
                -- 满足条件后，将HARVEST动作插入actions序列中即可
                table.insert(actions, ACTIONS.HARVEST)
            end
        end,
        ...
    }
    USEITEM = --args: inst, doer, target, actions, right
    {
        ...
        -- 睡袋的组件，可以让人物进入睡眠状态
        sleepingbag = function(inst, doer, target, actions)
            -- 睡袋要对着玩家自己使用，并且不能处于失眠和睡着的状态。
           if doer == target and doer:HasTag("player") and not doer:HasTag("insomniac") and not inst:HasTag("hassleeper") then
                table.insert(actions, ACTIONS.SLEEPIN)
            end
        end,
        ...
    }
    POINT = --args: inst, doer, pos, actions, right
    {...
    }
    EQUIPPED = --args: inst, doer, target, actions, right
    {...
    }
    INVENTORY = --args: inst, doer, actions, right
    {...
    }
    ISVALID = --args: inst, action, right
    {...
    }
}
...
```


## replica
component中储存了大量的玩家属性数据，比如饱食度，精神度等等。为了保证数据一致性，这些数据都是储存在主机(server)中的，从触发动作到最后产生属性变化，整个计算过程也是在主机中完成。但是，其他玩家(client)也有获取调用这些数据的需求，比如玩家的UI界面需要显示饱食度的具体数值，就需要读取数据。但是客机上又不储存这个数据，怎么办呢？原始的做法是，每当要调用数据的时候，就发送一个请求给主机，然后主机会返回一个网络变量，读取这个变量就可以了。但这样的处理方式在代码上不够优雅，会让整体布局变得凌乱。因此，专门针对component的属性，就派生出了replica。replica可以存在于client中，是component的同态附加。每当数据发生变化，就可以通过replica进行同步，这就大大简化了component主体部分的代码，可以让component的主体部分只关心游戏逻辑本身，而把网络数据传输的部分放在replica中。
大部分component都不需要在client中存在，因为不需要读取相应的数值，这里也只是略作说明，让大家有个基本了解。更详细的介绍，我会放在后面的文章里，和网络编程的部分一起讲解。

## 结语
entity是灵魂，prefab是骨架，component是血肉。正是不同的component提供了大量的功能，才使得整个游戏世界的交互变得丰富多彩。如果你希望深入了解游戏机制，那就需要好好地阅读不同的component。
本周末还会发布一个相关Mod【无情铁手】，通过实际的代码实践来帮助大家进一步理解本期的内容，请关注本专栏的消息。



## 附录1:系统组件介绍

系统底层组件，是被封装好的，无法看到底层的定义代码，只能根据官方代码来猜测用法。之所以做这样的封装，往往是因为相关的功能需要调用非常底层的API或者需要高速运算节约性能。它们被广泛地应用，在任何代码里都有可能见到。本文会介绍常用的系统组件以及其中常用的函数。


### AnimState

这个组件主要负责对角色的外观和动画进行控制。人物执行不同的动作会播放不同的动画，佩戴某些装备会改变外观，以及改变视觉上的大小，甚至计量条的变化等。凡是和物体形态变化有关的，都由AnimState来操作实现。


| fn                        | 用途说明                                                                         | 传入参数说明（按参数顺序）                                                                                                                | 返回值说明                       |
|---------------------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|-----------------------------|
| PlayAnimation             | 播放指定名称的动画，会立刻中断当前动画的播放                                                       | anim 动画名; loop 是否重复，可省略，默认值是fasle                                                                                            | 无                           |
| PushAnimation             | 将指定动画推送到播放序列中，当前动画播放完后会接着播放这个动画，常见于要通过一组动画来表现人物的场景                           | anim 动画名; loop 是否重复，可省略，默认值是fasle                                                                                            | 无                           |
| SetBuild                  | 设置指定的外观。比如兔子有夏、冬两种形态，就是通过设置不同的Build来实现的                                      | build 外观名                                                                                                                    | 无                           |
| SetBank                   | 设置指定的动画组。玩家站在地上和骑在牛上，使用的是两套不同的动画，就是通过设置不同的Bank来实现的                           | bank 动画组名                                                                                                                    | 无                           |
| Show                      | 展示某个部分，通常和Hide搭配使用。比如，装备武器时，会隐藏ARM_normal，显示ARM_carry，人物的手就发生了变化             | part 部分                                                                                                                      | 无                           |
| Hide                      | 隐藏某个部分，和Show搭配使用                                                             | part 部分                                                                                                                      | 无                           |
| OverrideSymbol            | 覆盖某个标记点，常见于装备武器后，手上就出现了一把武器。                                                 | inst_symbol 要覆盖的标记点 ;swap_build 用于替换的build ;swap_symbol 用于替换的build上的标记点                                                      | 无                           |
| ClearOverrideSymbol       | 清除指定标记点的覆盖                                                                   | inst_symbol 要清除覆盖的标记点                                                                                                        | 无                           |
| IsCurrentAnimation        | 检测当前播放的动画是否为指定的动画                                                            | anim 动画名                                                                                                                     | 无                           |
| SetTime                   | 设置动画停留在第几秒                                                                   | time 停留位置                                                                                                                    | 无                           |
| GetCurrentAnimationTime   | 获取当前动画停留在第几秒                                                                 | 无                                                                                                                            | time 停留位置                   |
| GetCurrentAnimationLength | 获取当前动画的长度                                                                    | 无                                                                                                                            | length 动画长度                 |
| SetPercent                | 设置动画百分比，对于一些通过动画帧来表现数值的物品很有用。比如雨量计，实际上是设置了一个动画，从0到100%，然后根据实际数值设置相应的百分比      | anim 动画名; percent 百分比                                                                                                        | 无                           |
| SetFinalOffset            | 不确定，根据函数名猜测，是设置动画的帧偏移量。                                                      | offset 动画偏移量，可以设置为负数。                                                                                                        | 无                           |
| SetScale                  | 设置缩放比例                                                                       | length_scale 长度缩放;width_scale 宽度缩放 。取值填小数，如果是负数，则是相应的方向颠倒。                                                                   | 无                           |
| SetMultColour             | 设置角色的r,g,b,a，也就是三个颜色+透明度。可以通过这个函数来让角色变得透明                                    | r,g,b,a四个参数，分别对应红，绿，蓝的颜色值以及透明度。取值均在[0,1]之间。对于透明度，取0时就是完全透明。                                                                  | 无                           |
| GetMultColour             | 获取角色的r,g,b,a                                                                 | 无                                                                                                                            | r,g,b,a 同SetMultColour的输入参数 |
| SetAddColour              | 设置附加颜色                                                                       | r,g,b,a四个参数，分别对应红，绿，蓝的颜色值以及透明度。取值均在[0,1]之间。对于透明度，取0时就是完全透明。                                                                  | 无                           |
| SetBloomEffectHandle      | 设置Bloom效果的处理器                                                                | path 处理器路径                                                                                                                   | 无                           |
| SetOrientation            | 设置刚体轴方向。不同的轴方向会影响看到的视觉效果，比如池塘看起来是贴着地面的，就是因为设置了这一参数为ANIM_ORIENTATION.OnGround | direction 方向，这里使用定义于constants.lua中的全局变量，ANIM_ORIENTATION下的各个值                                                                | 无                           |
| SetLayer                  | 设置图层，图层是有固定的摆放顺序的，比如土地是最下一层，然后农场是中间层，农场里的作物是最上层。在构建一些多层结构的东西时，都需要设置图层。       | layer 图层变量，这里使用定义于constants.lua中的全局变量，LAYER_BACKGROUND/LAYER_WORLD/LAYER_WORLD_BACKGROUND/LAYER_WORLD_CEILING/LAYER_FRONTEND | 无                           |
| SetSortOrder              | 设置排序优先级，常与SetLayer配套使用，当有多个物品重叠时，优先级高的排在前面。                                  | priority 优先级，整数                                                                                                              | 无                           |

### SoundEmitter

声音控制组件，用于播放物体本身拥有的音效资源。
| fn                  | 用途说明                  | 传入参数说明（按参数顺序）                                         | 返回值说明                               |
|---------------------|-----------------------|-------------------------------------------------------|-------------------------------------|
| PlaySound           | 播放指定音效                | path 音效文件路径                                           | 无                                   |
| KillSound           | 停止播放指定音效              | sound 音效名                                             | 无                                   |
| PlayingSound        | 判断是否在播放指定音效           | sound 音效名                                             | is_playing 是否在播放该音效，取值为true 或 false |
| SetParameter        | 设置音效参数，可能会影响某些音效的播放效果 | sound 音效名;param 参数名;value 参数值                         | 无                                   |
| SetVolume           | 设置音量                  | sound 音效名;volume 音量                                   | 无                                   |
| PlaySoundWithParams | 带着参数播放音效              | path 音效文件路径,param_tbl 参数表，形如{param_a=xxx,param_b=xxx} | 无                                   |


### Transform

这一组件负责管理物体的位置、大小、方向等。其中「大小」与AnimState有区别的地方在于，AnimState设置的大小是视觉上的大小，而Transform则是实际上的大小，会影响到物体的碰撞等相关物理效果。

| fn               | 用途说明                                                                                                           | 传入参数说明（按参数顺序）                            | 返回值说明                   |
|------------------|----------------------------------------------------------------------------------------------------------------|------------------------------------------|-------------------------|
| SetPosition      | 设置物体的世界坐标，可以让物体瞬移到指定位置                                                                                         | x,y,z 物体的三维坐标                            | 无                       |
| GetWorldPosition | 获取物体的当前世界坐标                                                                                                    | 无                                        | x,y,z 物体的三维坐标           |
| SetScale         | 设置物体的缩放比例                                                                                                      | x,y,z 物体在三个方向上的缩放比例                      | 无                       |
| GetScale         | 获取物体的缩放比例                                                                                                      | 无                                        | x,y,z 物体在三个方向上的缩放比例     |
| SetFromProxy     | 不确定，猜测是设置所有参数同步于proxy，一般常见于各种特效，同步于对应的附加物上                                                                     | proxy_guid 常用的写法是传入proxy变量，然后取proxy.GUID | 无                       |
| SetRotation      | 设置物体的朝向                                                                                                        | degree 朝向角度，取值范围[0,360]                  | 无                       |
| GetRotation      | 获取物体的朝向                                                                                                        | 无                                        | degree 朝向角度，取值范围[0,360] |
| SetFourFaced     | 设置物体有四个面，会影响物体在相应朝向时展示的形态。四个面就是每个面占90度。其它类似的还有SetNoFaced(1面),SetTwoFaced(2面),SetSixFaced(6面),SetEightFaced(8面) | 无                                        | 无                       |

### Physics

这一组件负责管理物体的物理运动相关的内容，包括物理参数，移动，碰撞等。
通常而言，不建议自己配置物理运动相关的参数，错误参数可能造成一些诡异的物理效果。最好使用官方给出的预设函数来一键配置，位于`standardcomponents`

RemovePhysicsColliders
**相关函数待定，需要测试，结合设置各种物理配置的脚本来查看**
MakeCharacterPhysics 等



### MiniMapEntity
这一组件负责管理小地图的图标


| fn                  | 用途说明               | 传入参数说明（按参数顺序）     | 返回值说明 |
|---------------------|--------------------|-------------------|-------|
| SetIcon             | 设置小地图图标            | image 图标文件名       | 无     |
| SetPriority         | 设置优先级，高优先级可以显示在更上面 | priority 优先级，整数   | 无     |
| SetEnabled          | 设置小地图图标是否可用        | 是否可用,取值ture/false | 无     |
| SetCanUseCache      | 待定，设置可使用缓存         | 是否可用,取值ture/false | 无     |
| SetDrawOverFogOfWar | 待定，设置可无视迷雾显示图标     | 是否可用,取值ture/false | 无     |
| SetIsFogRevealer    | 待定，设置可以显示迷雾        | 是否可用,取值ture/false | 无     |

### DynamicShadow

这一组件负责管理物体的影子。每个物体的影子都各不相同，甚至同一个角色，在不同的装备下也有不同的效果。大家可以试试装备不同的雨伞观察一下影子的变化。
| fn      | 用途说明    | 传入参数说明（按参数顺序）                              | 返回值说明 |
|---------|---------|--------------------------------------------|-------|
| SetSize | 设置影子大小  | length_scale 长度缩放;width_scale 宽度缩放 。取值填小数。 | 无     |
| Enable  | 设置是否有影子 | 取值 ture/false                              | 无     |

### Light

这一组件负责管理光源，可以为一个物体添加光源并调整设置相关参数如发光半径、强度、衰减度等。

| fn                     | 用途说明                                                     | 传入参数说明（按参数顺序）              | 返回值说明                      |
|------------------------|----------------------------------------------------------|----------------------------|----------------------------|
| Enable                 | 设置光源是否可用                                                 | 取值 ture/false              | 无                          |
| IsEnabled              | 判断光源是否可用                                                 | 无                          | 取值 ture/false              |
| SetRadius              | 设置光源半径。在很多涉及光的计算中都会用到光源半径，比如作物生长需要光源，这个光源是有距离要求的，太远的就不算。 | radius 半径距离                | 无                          |
| GetCalculatedRadius    | 获取光源半径                                                   | 无                          | radius 半径距离                |
| SetIntensity           | 设置光源亮度                                                   | intensity 亮度,取值[0,1]       | 无                          |
| SetFalloff             | 设置衰减强度                                                   | falloff 衰减强度,取值[0,1]       | 无                          |
| SetColour              | 设置光源颜色                                                   | r,g,b 分别对应红，绿，蓝的颜色，取值[0,1] | 无                          |
| GetColour              | 获取光源颜色                                                   | 无                          | r,g,b 分别对应红，绿，蓝的颜色，取值[0,1] |
| EnableClientModulation | 待定，未确认用途                                                 |                            |                            |

### LightWatcher

光照监视器，这一组件可以监控光照的情况。在游戏中，很多动植物的活动都和光照有关。例如蜘蛛一般情况下只在晚上和黑暗环境下活动，猪人则在白天或者明亮的环境下活动。如何判断黑暗和明亮，就是光照监视器来做的。

| fn             | 用途说明      | 传入参数说明（按参数顺序）     | 返回值说明         |
|----------------|-----------|-------------------|---------------|
| IsInLight      | 判断是否在明亮环境 | 无                 | 取值 ture/false |
| SetLightThresh | 设置明亮阈值    | thresh 阈值，取值[0,1] | 无             |
| SetDarkThresh  | 设置黑暗阈值    | thresh 阈值，取值[0,1] | 无             |
| GetTimeInLight | 获取光照时长    | 无                 | time 光照时长     |
| GetLightAngle  | 获取光照角度    | 无                 | angle 光照角度    |
| GetLightValue  | 获取光照值     | 无                 | light_val 光照值 |


### Follower

在component中也有一个同名的follower，但这个系统组件的follower是更底层的，常用于一些特效跟随的处理。而component的follower更侧重于一个生物跟随另一个生物。

| fn           | 用途说明               | 传入参数说明（按参数顺序）                                         | 返回值说明 |
|--------------|--------------------|-------------------------------------------------------|-------|
| FollowSymbol | 跟随标记，常用于特效跟随于某个物体。 | target_guid 跟随目标的guid;symbol 跟随的标记点; x,y,z 在三个方向上的偏移量 | 无     |

### Network
网络相关，未能根据代码直接看出用途，需要在游戏中进行较多的实验才能确定，暂时略过。



## 附录2:常见组件介绍：以功能为依托

数量太多，以后慢慢补充，如果有兴趣的同学，也可以私信我帮助完善。
以下是我觉得使用比较广泛的component，其实有点基础的同学可以自己看代码来理解。

* [ ] 移动：locomotor
* [ ] 劳作：workable，pickable，finiteuses
* [ ] 种植：growable
* [ ] 食用：edible，eater，perishable，hunger，sanity
* [ ] 战斗：combat，weapon,health ，aoetargeting（是否可以做出闪避的效果？）
* [ ] 口袋：inventory, inventoryitem，stackable，container，equippable，inspectable
* [ ] 掉落：lootdropper
* [ ] 篝火：fueled，fuel
* [ ] 交易：trader，
* [ ] 状态：burnable，propagator，freezable
* [ ] 检查：inspectable
* [ ] 作祟：hauntable
* [ ] 动物行为
    * [ ] 跟随：follower，leader
    * [ ] 睡觉：sleeper
    * [ ] 产出,生成：spawner，periodicspawner（拉屎），childspawner
    * [ ] 骑：rider
    * [ ] 回家：knownlocations，homeseeker
    * [ ] 轨迹：entitytracker（大象脚印、
* [ ] 其它
    * [ ] playercontroller
    * [ ] timer

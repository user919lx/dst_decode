## 引言

在开始本篇内容之前，先来思考一个小问题：抛开代码，仅仅从游戏本身的视角来看，猪人与玩家有什么相同点和不同点？相同点：都有血量、都能移动和战斗、都能戴帽子、都能吃东西等等；不同点：猪人没有精神值，可以徒手砍树，无法装备武器等等。这些相同点和不同点，都是一些属性或者功能，这就引出了今天要介绍的内容：如何为entity添加各种功能，又如何让这些功能与游戏世界里的其它entity进行交互，产生影响。要解答这些问题，就需要弄清楚两个概念：组件和动作。


## 定义
「吃东西」是一个很基础的能力，玩家都能吃东西，猪人、蜘蛛等生物也可以。我们就以「吃东西」为例子来进行讲解。一个完整的进食流程如下

![吃东西](/images/eat.png)


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

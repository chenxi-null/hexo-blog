title: 记一次有趣的代码重构
author: chenxi
tags:
  - clean-code
categories: []
date: 2020-02-14 23:40:00
---

> 我等采石之人，当心怀建造大教堂之愿景。
>——《程序员修炼之道》

发现代码重构和灭霸的响指有一个共同点，两者的出发点都是为了消除系统之中的一部分，让剩下的另一部分存活得更好，从而使得整个系统运更为有序。  

不同之处就是灭霸是无差别清除，而重构对于代码的清除却是经过深思熟虑精心设计的。  

闲话少说，下面开始正题。

## 重构前

背景是足球比赛的项目，需要处理各种类型的比赛数据。

比赛数据的展示维度如图：
![](https://user-gold-cdn.xitu.io/2020/2/13/1703ef2678ba061f?w=1640&h=332&f=png&s=61336)

用代码表示是这样的：
```java
// 表示一场比赛里两支球队各个阶段的数据
@Data
class MatchStat {
    private TeamStat homeTeamStat;
    private TeamStat guestTeamStat;
}

// 一支球队各个阶段的数据
@Data
class TeamStat {
    /** 上半场的数据 */
    private Stat firstStageStat;   
    /** 下半场的数据 */
    private Stat secondStageStat;   
    /** 全场的数据 */
    private Stat fullStageStat;   
}

// 这个是表示比赛数据的 Model
@Data
class Stat {
    /** 得分 */
    private int score;
    /** 传球数 */
    private int pass;
    /** 抢断数 */
    private int steal;
    // ...
}
```
另外，项目中存在很多种比赛数据类型，如：  
>HeartIntensity 心率强度;   
ExerciseLoad 负荷强度;   
DistanceSpeed 跑动距离-速度分布;   
DistanceTime 跑动距离-速度分布;  
......

对于每种数据类型 Model，如 `ExerciseLoad`，都要再定义一个 TeamModel，如 `TeamExerciseLoad`，表示一只球队比赛各阶段的数据，
然后再定义一个 MatchModel, 如 `MatchExerciseLoad`，表示一场比赛中两队各阶段的数据。  

那么，N 种数据类型的话，一共就要定义 3 * N 个 Model 类。

**问题一：是不是有办法减少 Model 类的数量呢？**

我们暂时先不管这个问题，继续往下看，如果有这样一个需求，我们拿到一个 `MatchStat` 对象，要把两队所有阶段的传球数都设为 0：
```java
public void processMatchStat(MatchStat matchStat) {
    // 如何实现这个需求呢？
}
```

之前在项目里一般是这样处理的：
```java
// 1. 定义一个方法处理单队的数据
void processTeamStat(TeamStat teamStat) {
    teamStat.getFirstStageStat().setPass(0);
    teamStat.getSecondStageStat().setPass(0);
    teamStat.getFullStageStat().setPass(0);
}

public void processMatchStat(MatchStat matchStat) {
    // 2. 分别传入主客队的 TeamModel 作为入参, 调用上面那个方法
    processTeamStat(matchStat.getHomeStat());
    processTeamStat(matchStat.getGuestStat());
}
```

上面的代码看似没有问题，也没有一行重复代码，这种类似的需求（处理两队各个阶段的比赛数据）在项目里还是不少的，可是每次都这样写一遍不免有些枯燥。  
显示的重复代码确实找不到，但是“隐式”的重复代码呢？

**问题二：有没有办法简化项目里的这种模版代码?**

## 重构后
下面揭示答案，直接贴出重构后的处理方式：
```java
public void processMatchStat(TwoTeamNestedAllStageModel<Stat> matchStat) {
    // 一行代码解决问题!
    ModelUtils.handleTwoTeamNestedAllStageModel(matchStat, (stat) -> stat.setPass(0));
}
```
是不是简洁了很多，枯燥指数大大降低。

**实现细节**： 

### 减少 model 类的数量
定义两个通用的 Model：
```java
public class TwoTeamNestedAllStageModel<T> {
    private AllStageModel<T> home;
    private AllStageModel<T> guest;

    public TwoTeamNestedAllStageModel(AllStageModel<T> home, AllSatgeModel<T> guest) {
        this.home = home;
        this.guest = guest;
    }

    public AllStageModel<T> getHome() {
        return home;
    }

    public AllStageModel<T> getGuest() {
        return guest;
    }
}

@Value // lombok will generate consructor and getters
public class AllStagetModel<T> {
    private T firstStage;
    private T secondStage;
    private T fullStage;
}
```

`TwoTeamNestedAllStageModel` 和 `AllStageModel` 可以看成是一种容器类，有点类似于 JDK 里的 `List` 和 `Map`。  

以 `ExerciseLoad` 这个 Model 为例，我们原先需要定义 `TeamExerciseLoad` 和 `MatchExerciseLoad` 两个类。   

现在有了这两个“容器 Model”，我们就只需要声明一个 `TwoTeamNestedAllStageModel<ExerciseLoad>` 就行了，
这样项目里一下子就减少了 2 * N 个 Model 类，省去了重复定义这些 Model 的枯燥工作。

这就回答了刚才提出的问题一。

下面我们来接着回答问题二：

### 消除“隐式”的重复代码
借助“容器 Model”我们把这些模版代码都提取到一个工具类里了，通过这种方式消除了"隐式“的重复代码
```java
public class ModelUtils {
    public static<T> void processTwoTeamNestedAllStageModel(
        TwoTeamNestedAllStageModel<T> m, java.util.function.Consumer<T> action) {
        processAllStageModel(m.getHome(), action)
        processAllStageModel(m.getGuest(), action)
    }

    private static <T> processAllStageModel(
        AllStageModel<T> m, java.util.function.Consumer<T> action) {
        action.accept(m.getFirstStage());
        action.accept(m.getSecondStage());
        action.accept(m.getFirstStage());
    }
}
```

## 总结

业务开发不仅仅是简单的 CURD，我们在开发的过程中，针对不同的业务场景其实有很多地方是可以归纳提炼的，让自己的代码更加简洁优雅，
切记不要惰于思考，只是凭直觉简单地堆砌代码、面向任务编程。   好的代码都是设计出来的。  

我们要以工程师和设计师的身份自居，培养自己对于坏代码、重复代码的敏锐嗅觉，逐步提升自己的代码品味。

好处:
- Model 类的数量减少了 2 / 3，处理这些 model 类的方式也更简单了，提升了代码的简洁度，提高了编码效率

坏处：
- 比起之前的代码，重构后的 Model 定义和使用方式具有一定的学习成本
- 如果 Model 类不是很多的话，重构的收益其实有限，有过度设计的嫌疑
- 代码量变少了，“每千行 Bug 数”提升了，老板可能觉得你的工作效率和质量下降了（手动狗头）

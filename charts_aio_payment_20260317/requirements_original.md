## 总体背景

背景：希望研究AIO平台的用户付费情况，主要是付费前行为/是否付费的研究
数据来源：BigQuery
核心议题：用户是因为什么付费的？
时间范围：2026.2.16到2026.3.16

## 用户分层

用户分层方面，总体是生成过至少1个set或EP用户，两个分层维度（但我还没看出来这个分层和关心的指标有啥关系）
总体：生成过至少 1 个 set 或 ep 的用户

- **Set (卡片集)**：对应业务数据库中的 `deck` 表，按 `uid` 统计创建数量 $\ge 1$。
    
- **EP (预测卷)**：对应业务数据库中的 `ep_detail` 表，按 `uid` 统计生成数量 $\ge 1$。
    

两个分层维度如下：

1. 付费维度拆分（已付费、触发且未付费、未触发）
    

这个维度需要结合“订阅状态”与“前端弹窗埋点”来交叉对比。

- **已付费用户**： 主要通过公共服务接口获取的 `SubscriptionInfo` 订阅信息来判断。
    
- **相关字段**：`expireTimestamp`（过期时间需大于当前时间）、`refund`（退款状态需为 0 未退款）、`regular`（付款状态需为 1 已付款）或 `isTrialPeriod`（是否试用期为 1）。只有符合这些条件，才是处于有效订阅内的“已付费”用户。
    
- **触发了 Paywall（付费墙）但是还未付费**： 这部分状态业务表里不会直接存“是否看过弹窗”，强依赖于**前端****埋点****事件**。
    
- **相关字段/事件**：你需要去数仓捞取各类 Paywall 曝光事件（例如拦截到达上限抛出 `EXCEED_LIMIT` 时前端弹出的 `paywall` 相关展示事件）。
    
- **逻辑**：圈出**“有 Paywall 展示埋点事件”** 并且 **“当前订阅状态（SubscriptionInfo）为未生效/无记录”** 的用户。
    
- **未触发 Paywall**：
    
- **逻辑**：即不在上述“已付费”名单内，且在埋点数据中**从未上报过** Paywall 展示相关事件的用户。
    

2. 订阅周期拆分（周包、月包、年包）
    

对应用户订阅信息（`SubscriptionInfo`）里的具体套餐数据。

- **相关字段**：
    
- **productId（核心）**：商品 ID。不同平台（iOS、Android、Web 的 Stripe）的周包、月包、年包都有对应唯一的商品编号（如 `prod_xxxx`）。通过匹配这个字段，能最精确地归类套餐类型。
    
- **period（订阅时长）**：订阅周期字段（代表该套餐对应的时间跨度数值，如 1、12 等）。
    
- **subscribePlan**：套餐大类（通常这里标的是 `unlimited` 大类，具体细分周期仍需要依赖 `productId` 和 `period`）。
    

  

  

  

## 分析指标（这些东西是干嘛的？和目标的相关性怎样？）

1. ### **给paywall分类**
    

按照source分类：触发paywall的用户中，按照source进行分类。

按照paywall触发位置分类：不同端的触发是否不同？

  

（这里不知道对不对）在拉取未付费但“触发了 Paywall”的用户时，从埋点数据 event_properties 提取 source 或 trigger_name 字段，将其划分为：

- 解题额度不足拦截：比如主页问答、追问场景 (Web_Solve_Success 为空并直接导向 Paywall)。
    
- 文档/内容限制拦截：例如创建了太多 Deck/EP 被拦截（API 返回 EXCEED_LIMIT），或者点击“生成 EP”按钮直接弹框（Web_EP_Cover_Popup）。
    
- Onboarding拦截：比如新用户注册完成后必弹的订阅引导（iOS_Onboarding_Store_Click）。
    

  

  

2. ### **哪些使用功能和付费相关性比较大？**（怎么呈现比较合适）
    

付费前使用功能（study guide scroll，quiz,flashcard, exam predictor create, podcst create (Web_FC_Podcast_Generate) ,mastery topic click, solver solve success)

- **study guide scroll** (`Flashcard_Web_StudyGuide_Scroll` / `Flashcard_Web_Summary_Scroll` / `AS_iOS_Summary_scroll`): 对应 Flashcard / AI Note 页面中，用户在右侧或独立面板**滑动查看 Study Guide (Summary 总结/Full notes 全文)** 时的动作。
    
- **quiz** / `quiz_enter` (`Flashcard_Web_Quiz_Enter` / `Flashcard_Web_Quizzes` 等): 对应用户在 Flashcard 页面**进入或点击 Quiz（测试练习）模块**，开始使用基于文档生成的练习题功能。
    
- **flashcard** (`Flashcard_Web_Flashcard_Tab` 等相关事件): 对应 AIO 业务中 **Flashcard 核心功能的各类操作**（包括切换到卡片标签、查看图片、学习卡片集等）。
    
- **exam predictor create** (`Web_EP_Create_Exam` / `EP_iOS_Create_Click` 等): 对应用户在 Exam Predictor（预测卷）模块中，点击**创建/生成新的预测卷（Create my prediction）**的环节。
    
- **podcst create / Web_FC_Podcast_Generate** (`FC_iOS_Podcast_Generate` / `Web_FC_Podcast_Generate`): 对应用户在 Flashcard 的 Podcast（播客）模块，点击**生成音频播客**的转化环节。
    
- **mastery topic click** (`FC_Web_set_mastery_view` / `topicClick` 方法): 对应在 Exam 或 Flashcard 中，用户在 **"Topic Mastery" (知识点掌握度) 面板点击具体某个知识点 (Topic)** 以展开或重试该知识点练习的环节。
    
- **solver solve success** (`Web_Solve_Success` / `Solve_Success` 等): 对应 AIO 及搜题业务中，用户上传图片或输入文本后，**AI 成功解析并生成解答结果**的成功回调环节。
    

  

一些疑问：

1. 用什么来衡量最相关？
    
2. 应该如何呈现相关性
    

  

3. ### **不付费的原因**
    

触发paywall但7天内未付费用户，第一次触发paywall后，之后还会回来使用吗？(周留存(按dau），触发paywall后再次create 多少set)

**圈定目标人群（研究谁？）**

- **条件：** 碰到了付费弹窗，但在接下来的 7 天内**没有**付钱的用户。
    
- **重点：** 我们只看他们**第一次**碰到弹窗后的表现。
    

1. **观察指标一：之后还会回来使用吗？（周****留存****）**
    

- **含义：** 在第一次碰到弹窗后的第 1 天、第 2 天……直到第 7 天，这些人里有多少人还会每天打开 App（也就是日活 DAU，采用的是和DAU相同的口径）。
    
- **目的：** 看看收费弹窗会不会导致用户大规模流失（比如碰壁后直接卸载了）。
    
- 疑问：如何呈现这一形式比较合适
    

2. **观察指标二：回来后还在继续用核心功能吗？（再次 Create Set 的数量）**
    

- **含义：** 那些没被劝退、还愿意回来用的用户，在碰到弹窗之后，又创建了多少个新的卡片集（Set）。
    
- **目的：** 看看用户是只回来随便看看，还是依然在活跃地使用系统允许的免费额度进行学习和创作。
    
- 疑问：为什么要用再次 Create Set 的数量来衡量，会不会出现大部分人都是只创建了一个
    

  

  

4. ### 触发了多少次paywall后付费
    

思路是画一个分布图，比如说对触发了paywall的这些用户，触发次数和其中付费人数及其比例的关系

这个思路是否正确，有无更好的想法？

  

  

## 假设验证

- 触发paywall但未付费用户，之后还会来使用。可以营销手段提升（discount，宣传solver等）
    
- 使用ai study和EP功能用户，更可能付费。cross promote，突出会员价值
    
- 使用功能越多，觉得价值越高，越可能付费。cross promote，突出会员价值
    
- 使用mastery topic click，用户觉得我们是系统学习平台，不是工具，更可能付费。
    

  

针对以上四个假设进行理解，并尝试提出方案并实施验证

是否需要做AB测试？
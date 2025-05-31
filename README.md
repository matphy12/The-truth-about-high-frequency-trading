# The truth about high-frequency trading
Personal quant trading open-source strategy and notes
我决定开源我迄今为止构建的数字货币高频量化交易策略。虽花费了我大量时间构建，但随着我对市场机制与金融伦理的深入思考，我选择不再继续这条道路。接下来，我将陆续公开策略代码、建模思想与验证过程，供有志者学习、检视与质疑。原因有三：

透明认知：经过实践与回测，我发现这些策略虽建立在复杂的分析与高频数据处理之上，但在真实市场中的表现不如预期。我意识到这些方法在信息结构的顶层早已趋于公开，或成为大型机构内的“共识性秘密”。对个人而言，它们并非通向财富的捷径。
分享探索与展示能力：我在构建这套策略过程中，投入了一些时间精力，也体现了我的建模能力、代码实现力和对市场结构的深入理解。所有核心因子均为独立思考所得，核心思想在三年前就有雏形，但当时缺乏辅助工具与验证手段，且因我当时不知道还有人能监控微信聊天而泄露，如今借助 ChatGPT 和代码能力，我才真正将其系统化实现。同时，早期我曾通过高风险实践验证部分市场猜想，虽有亏损，却加深了理解。公开这段历程，一方面是对自己探索的记录，另一方面也希望借此寻找合作者，或吸引潜在雇主关注。
价值观驱动：高频量化交易本质上是一种“微观套利”行为，通常建立在散户信息劣势之上。我逐渐意识到，这与我追求的长期价值投资理念存在根本冲突。我希望用这次公开行为，引发对金融效率与公平的更多讨论，哪怕微不足道，也愿作为一点改变的火种。我计划将全部策略代码和思考记录开源，并同步发布至多个平台，供有兴趣者学习、借鉴或完善。
我的放弃并非源于策略失败，而是源于它的成功基于他人的失败。这是我难以接受的根本逻辑。市场的效率若建立在信息不对称和系统性博弈之上，那就值得被重新审视。高频交易违背了我对公平交易的理解。如果一个市场只鼓励算法碾压，而不是基本面研究，那这不是“有效”，而是“失衡”。

事实上，我们都知道——在赌场里是几乎不可能赚钱的。唯一可能赚钱的方式，就是存在一个比你更大的“韭菜”，与其反向操作，但是赌场也完全可以先割“高手玩家”，然后其他韭菜“先养肥”。市场机制中如果激励错误，那聪明人也终将沦为被猎杀的目标。

事实上，做市商拥有绝大多数散户不知道的信息：如某个订单流的订单，下单的交易者是否是API下单的字段。甚至可能存在做市商能够直接获取交易者的杠杆、止损位置等信息，因此高频交易虽然依靠技术，但其实盈利的本质还是来自于不公平竞争。

总的而言，市场上曾存在一个强大的因子，少数拥有深度思考能力并往此方向思考过的人能够想到，但是可能这个时代该因子已经几乎失效。可能存在其他我不知道的因子，但是无论如何市场的游戏规则需要改变。我希望能以此为起点，引发更多对市场公平、交易结构、金融伦理的探讨。

该策略的启发来自于我个人在加密市场上的赌博，首先我们确认一个问题：赌徒在加密货币市场使用高倍杠杆后，上涨和下跌的最终概率会因为赌徒的开仓而变化吗？有人认为不会变化，大部分的赌徒只是因为手续费的磨损，以及大数定律的最终归零而亏损的。实则不然，即使赌场不来狙击你的仓位（比如赌徒杠杆做多，狙击你的仓位的意思就是反向做空，打压你的仓位，等你被迫爆仓卖出以后再以低价买回来把赌徒的钱赚走），通过市场结构的微观信息，高频量化交易者依旧能探测到赌徒的仓位。

在A股大概也存在类似的策略，也就是探测经常止损的“意志不坚定”反复买入卖出尤其反复止损的玩家的订单，然后反向进行做空操作。虽然是聪明人赚了韭菜的钱，但是这样的操作逻辑完全与价值投资的理念相悖。

为了说明这个策略的思路究竟有多清晰，我现在直接公开它的实现方式：

1、 实时获取交易所某标的物（如BTC、ETH、DOGE）的订单流数据（即实时的所有的成交订单金额与价格），并存储。

2、 根据获取的订单流数据，找到相同数额且开仓方向相反的平仓单（比如检测到一个单子做多0.345个比特币，在不长的时间后检测到另一个做空0.345个比特币的单子，这大概率就是一个平仓单）。

3、 根据开仓与平仓价格，计算用户的手续费与亏损，预估下一次用户的开仓数量与价格。

4、 在不长的时间内检测到符合的开仓数量与价格，反向割韭菜。

但是这里还是存在诸多问题的：

1、如何具体计算开仓数额？

2、我不知道交易者的原始杠杆，如何预估他的本金？

3、韭菜不一定全仓交易

4、做市商完全可以自己和自己交易，使用“虚假订单”。

我把这个重要的因子公布的原因是因为我认为该策略应该是赚不了钱了，即使能赚，我也不想赚这个钱。我希望这个世界更加美好，而不是大量聪明人把心思浪费在如何“割韭菜”上。

对于第一个问题，我一开始的思路是拟合一个开仓公式，也就是如何找到一个公式，精确根据余额计算出玩家的开仓结果，或者根据开仓结果估计玩家的余额。但是后来一拍脑袋发现，我不需要完全拟合全部的可能的开仓数额，我只需要抓住我能够发现的韭菜就行了，也就是错过机会没关系，把确定性高的机会抓住就行。

对于第二个问题，我需要遍历所有可能的杠杆，因为只要最终结果精确相符，就一定可以检测到韭菜。

对于第三个问题，我们假设韭菜全仓交易，虽然可能过拟合，只要存在概率上的优势，就是好因子。

对于第四个问题，我假设“做市商“是随机投放”部分虚假自成交订单“，那么我需要用其他的因子”筛选“出更多韭菜出现的可能的情况，如连续趋势、大波动、支撑位与压力位等，来提高我的综合胜率，这样我筛选出来的订单大概率就是”韭菜订单“而不是做市商自己成交的”虚假订单“。

尽管策略可能仍有未尽之处，但我倾向于认为它已被主流做市商所针对，而非仍藏有尚未开发的巨大价值。可能做市商已经进行了大量干扰性自成交行为，他们已经针对已知策略进行了针对性对策的制定，因此我放弃优化我的策略转向开源：既坚持了公平正义的初衷，又能展现自己的能力。


在我的量化策略中，核心是对“开仓公式”的识别与建模。下面简要介绍：

所谓开仓公式，简而言之就是：
给定账户余额 Balance，标记价格 Price_mark，最新成交价 Price_latest，杠杆倍数 Leverage，假设全仓买入——我能具体开多少数量的合约？
这是每个交易者在下单前都被动回答的问题，但很少有人去主动研究它的机制。

这个公式的反问题同样重要：
如果我观察到某人在某一时刻以某个数额建仓，能否反推其账户的初始本金范围？遍历所有可能杠杆，是否能锁定他的资金规模？

这两个问题，99.999% 的交易者从未思考过。为了解答它们，我进行了大量截图与录屏，对比交易所“开仓前预估数量”与“实际成交数量”，并记录下相关参数。在不断实验中，我发现，在多数情况下，标记价格、最新价格、杠杆、余额这四个变量基本决定了开仓后的实际成交数量。也就是说：交易页面显示的“预估开仓量”往往就是实际成交结果。但也并非完全一致。在一些场景下，公式预测与真实结果存在偏差。

起初，我陷入了“死脑筋”——我执念于必须构建一个 100% 准确的开仓公式，哪怕要借助神经网络来拟合。我为此停滞了数月。直到某一天，我灵光一闪：我并不需要识别所有人的仓位，只需要识别出我能确认的韭菜就足够了。错过大部分机会没关系，只要能抓住那些确定性高的交易，策略依旧成立。

因此我可以直接使用加密货币交易所公开的公式：这在大型数字货币交易所官方是公开披露的。

具体而言，对于做空，目前(20250531)的公式为：Quality=int(balance*leverage/(1.001*latest_price))


由于开仓量是随着余额单调递增的，我们可以作图：Q Q* Q+1分别对应Balance1 Balance2 Balance3

我们假设括号内部 
 的计算结果是Quality:Q*，那么取整后的结果就是Q,也就是说余额为Balance2的时候开仓数量是int(Q*)=Q（为方便起见，这里展示开仓数额是整数的情况）.显然地，可以看出余额为[Balance3,Balance1)的时候，实际开仓数额也是Q，据此，我们可以根据实际开仓金额Q与Q+1对应的Balance区间来估算交易者的余额。


做多的公式较为复杂，需要用到标记价格。

具体建模、数学推导细节与代码实现，我将在后续文章中逐步更新，欢迎关注。

金融市场并非全然理性，它既是数字与算法的竞技场，也承载着无数普通人的梦想与命运。我的策略虽然曾一度接近“成功”，但我选择放弃，是因为它违背了我对公平、公正市场的理解。愿这次开源不仅是一段技术旅程的终点，也是一场金融伦理反思的起点。未来，我将继续探索那些能在尊重他人、构建更透明市场的前提下，依然有效的策略。也欢迎志同道合的朋友，与我一起探讨更有意义的交易之道。


I’ve decided to open source the high-frequency cryptocurrency trading strategy I’ve developed so far. Although it took me a great deal of time and effort to build, deeper reflection on market mechanics and financial ethics led me to stop pursuing this path.
    Going forward, I will gradually release the strategy’s source code, modeling rationale, and validation process—for others to study, examine, and critique. I do this for three main reasons:

1、Transparency and Realization:
Through hands-on experimentation and backtesting, I found that while these strategies are built on sophisticated analysis and high-frequency data processing, their real-world performance fell short of expectations. I came to realize that such methods have largely been exposed at the top layers of the information hierarchy—or have become "consensus secrets" among large institutions. For individuals, they are not shortcuts to wealth.
2、Sharing Exploration and Demonstrating Capability:
In building this strategy, I invested significant time and energy. It showcases my skills in modeling, code implementation, and deep understanding of market microstructure. All core factors were derived through independent reasoning, with the foundational ideas dating back three years. Back then, I lacked tools and means to validate them. Now, with the help of ChatGPT and improved coding capabilities, I’ve been able to systematize them. I also tested some hypotheses through high-risk trading in the early stages—while it led to losses, it deepened my understanding. By making this journey public, I aim not only to document my own process, but also to attract collaborators or potential employers.
3、Ethical Motivation:
High-frequency trading, at its core, is a form of "micro-arbitrage" that often relies on the informational disadvantage of retail traders. I gradually realized this stands in direct conflict with my belief in long-term value investing. Through this act of transparency, I hope to spark broader discussion around financial efficiency and fairness—even if my influence is limited, I want to be a small spark for change. I intend to open source all strategy code and related insights, and publish them across multiple platforms for those interested in studying, building upon, or challenging the work.
  My decision to walk away does not stem from the strategy’s failure, but from the realization that its success depends on the failure of others.
This is a fundamental logic I find difficult to accept. If market efficiency is built upon information asymmetry and systematic exploitation, then it deserves to be critically reexamined. High-frequency trading contradicts my understanding of fair and ethical markets. When a system rewards algorithmic domination over fundamental research, it is not "efficient"—it is imbalanced.
    In truth, we all know: it’s nearly impossible to win in a casino.The only way to profit is if someone more naive than you walks in—and you bet against them. But even then, the casino can choose to target the "skilled player" first, while letting the rest fatten up. If market incentives are misaligned, then even the smartest participants will eventually become prey.
    In reality, market makers possess a wealth of information that retail traders don’t—
such as whether an order was placed via API, or even potentially a trader’s leverage and stop-loss levels.
While high-frequency trading appears to be a battle of algorithms, its profitability often stems from unfair informational advantages, not just technical prowess.
    In general, there was once a powerful factor in the market that was thought of by a few people who had the ability to think deeply and in this direction, but perhaps this factor has almost become ineffective in this era.
    That edge has likely been eroded, absorbed, or countered by today’s dominant players.
Other edges may still exist, but regardless, the rules of the game need to change.
    This open-source initiative is my way of starting a conversation—
about fairness, about structural incentives, and about the ethics of modern markets.
    This strategy was inspired by my own gambling experiences in the crypto market.
It began with a core question:
   Does the probability of price movement change after a gambler opens a high-leverage position?
Some argue it doesn’t—that gamblers lose primarily due to fees and the law of large numbers slowly driving them to ruin.
But that’s only part of the story.
    Even if the “casino” doesn’t actively hunt their positions—say, a gambler goes long with 50x leverage, and no entity tries to short and liquidate him directly—the structure of the market leaks information.
Through micro-level order flow analysis, high-frequency traders can still detect where gamblers are positioned, and trade accordingly.
    A similar pattern likely exists in traditional equity markets (such as China’s A-shares), where algorithms may identify traders who frequently stop out—those lacking conviction and consistency—and then take the opposite side of their trades.
    Yes, clever people are profiting off of the uninformed.
But let’s be clear: this behavior is in direct conflict with the ideals of value investing and fair participation.
    To demonstrate just how clear the logic of this strategy is, I’m now going to openly share its core implementation:

1. Real-time Order Flow Capture
Continuously collect real-time trade data from an exchange for a given asset (e.g., BTC, ETH, DOGE), recording each trade’s amount and price.
2. Detect Matched Closing Trades
Identify closing positions by finding trades with the same amount but in the opposite direction within a short time window.
For example, if someone buys 0.345 BTC and shortly after someone sells 0.345 BTC, there’s a high probability this is a round-trip (open + close) by the same retail trader.
3. Estimate Fees and Losses
Calculate the difference between the open and close prices, adjust for estimated fees, and infer how much the trader likely lost. Use this information to predict their next likely entry size and price.
4.Exploit Recurring Patterns

If a similar trade appears soon after—same size, similar direction—you reverse trade against it, essentially betting that this “gambler” is likely to lose again.
    But this approach still faces several significant challenges:
1. How to accurately calculate the size of the opening position?
2. Without knowing the trader’s original leverage, how can I estimate their capital?
3. Retail traders don’t always go all-in.
4. Market makers can trade with themselves using “fake” or wash orders.
   
    I’m choosing to disclose this key factor because I believe the strategy is no longer viable—or even if it could still make money, I no longer want to profit this way. I want to help build a better world, not one where brilliant minds are wasted on figuring out how to "harvest retail traders."
    As for the first issue, my initial approach was to create a fitting formula—one that could either predict the player’s opening size based on their account balance, or reverse-engineer their balance based on observed positions.
    But eventually, I realized: I don’t need to fit everypossible scenario.
  I just need to catch the trades I can detect with high certainty. It’s okay to miss some opportunities, as long as the ones I act on are solid.
    As for the second issue, I need to iterate through all possible leverage levels. As long as the result matches precisely, I can identify the retail trader with confidence.
    For the third issue, I assume the retail trader is going all-in. This may result in some overfitting, but as long as there's a statistical edge, it qualifies as a good factor.
    For the fourth issue, I assume that market makers are randomly injecting some “fake self-traded orders.” Therefore, I need to apply otherfilters to isolate likely retail trades—for example, trends, large volatility, support/resistance proximity, etc.—to improve my overall accuracy. The goal is to ensure that what I’m detecting are most likely retail orders, not spoofed ones by market makers.
    Despite the remaining limitations, I now believe this strategy has already been targeted and neutralized by dominant market makers. It’s likely they’ve flooded the market with deceptive self-trades and engineered specific countermeasures against known strategies.
    That’s why I’m shifting away from optimizing this strategy toward open-sourcing it: to honor my commitment to fairness and transparency—and to demonstrate the depth of thought and technical ability behind it.
   At the core of my quantitative strategy lies the identification and modeling of what I call the "position sizing formula." Here's a brief introduction:

 Simply put, the position sizing formula answers the following question:
 Given an account balance (Balance), a mark price (Price_mark), a latest traded price (Price_latest), and a leverage multiplier (Leverage), how many contracts can I open if I go all-in?
  Every trader answers this question implicitly before placing an order, yet very few ever examine the mechanism behind it.
The inverse question is just as crucial:
  If I observe a trader opening a position of a specific size at a certain time, can I infer the approximate range of their initial capital? By testing across all possible leverage values, can I estimate their total account size?
These are questions that 99.999% of traders have never consciously explored. To answer them, I captured countless screenshots and recordings—comparing the “estimated position size” displayed before order execution to the actual size after execution—and logged all relevant variables. Through repeated experimentation, I found that in most cases, these four factors—mark price, latest price, leverage, and balance—largely determine the final position size. In other words, the estimated position shown on the exchange is usually accurate. But not always. In certain scenarios, the predicted result diverges from the actual one.

  Initially, I fell into a mental trap—I was obsessed with creating a formula that could predict the position size with 100% accuracy, even considering using neural networks to fit it. This obsession stalled my progress for several months. Then one day, I had a flash of insight: I don’t need to identify every trader's position—just the ones I'm sure are retail victims. Missing most signals is acceptable, as long as the ones I do catch have high certainty. That alone makes the strategy viable.
Fortunately, many exchanges publicly disclose their formulas for calculating position size. I can directly reference the official formulas published by major crypto exchanges.

  Specifically, for short positions, the formula is as follows:Quality=int(balance*leverage/(1.001*latest_price)) (20250531)


  Since the position size increases monotonically with account balance, we can visualize this relationship through a graph:Q Q* Q+1 correspond to Balance1 Balance2 Balance3 respectively


  Let’s assume the inner expression of the formula evaluates to a quantity Q∗. The final position size after rounding (typically by floor or truncation) is Q=int(Q∗). This means that when the balance is Balance2​, the opened position is Q contracts (for simplicity, we consider the case where the position size is an integer).
  It is evident that within the balance interval [Balance3,Balance1), the actual opened position remains Q. Based on this, we can estimate a trader’s account balance by identifying the balance range that corresponds to the observed position size Q or Q+1.

我知道我可以继续挖掘，以而进一步提高胜率——但我决定不再继续下去了。
策略的第一个关键问题，是对（现在）订单流数据结构的理解。在实际构建高频模型前，我们必须明确：
当用户以市价扫单，吃掉多个挂单时，交易所返回的订单流数据，是将其合并成一笔均价订单，还是逐笔拆分记录每一档成交？我打印出示例订单流（为简洁仅展示部分）：
{"p":"0.22970000","q":"227.00000000","T":1748246611586}
{"p":"0.22969000","q":"120.00000000","T":1748246611975}
{"p":"0.22968000","q":"25.00000000","T":1748246611975}
{"p":"0.22967000","q":"9713.00000000","T":1748246613038}
{"p":"0.22966000","q":"10398.00000000","T":1748246613038}
{"p":"0.22965000","q":"16900.00000000","T":1748246613038}
分析订单流结果可以看到，大部分订单流的时间间隔在数十毫秒以上，而少部分时候会出现同毫秒的订单流，极少数情况会有相邻毫秒的订单，因此：

在同一毫秒（例如 1748246613038）内，存在多个成交价格与数量，表明这是一次扫单行为被拆分成多笔成交。

因为如果数据流是发布均价的话，那么数据分布不会不均匀——应当是相邻毫秒订单流数量与同毫秒的订单流出现的频率差不多。同时查阅文档可以看到交易所内部的数据精度达到微秒甚至更精细的级别，进一步验证了该判断。

由此构建出我的获取订单流部分代码，我采用了合并同毫秒订单计算并存储高精度均价的方式，不过我现在想来可能可以直接存储扫单的全部成交信息，不过我也不想继续做下去了：
json json_msg = json::parse(msg->get_payload());
        if ((json_msg.contains("ping")) || (json_msg.contains("PING"))) {
            std::cout << "收到PING消息，发送PONG响应..." << std::endl;
            ws_client_trade.pong(hdl, msg->get_payload());  // 使用相同的payload回复Pong
        }
        // 提取字段
        long long timestamp = json_msg["T"];
        double price = std::stod(json_msg["p"].get<std::string>());
        double quantity = std::stod(json_msg["q"].get<std::string>());
        bool m = json_msg["m"];
        // 创建 Order 对象并设置方向
        Order new_orderyubei(timestamp, price, quantity);//把数据给neworderyubei
        new_orderyubei.direction = m ? "SELL" : "BUY";//根据交易所规则 如果订单流方向是True 意味着是主动卖出的订单
        new_orderyubei.mark_price = current_mark_price.load();//有另一个函数同时接收标记价格
        safe_push_back(existing_ordersyubei, new_orderyubei, max_existing_ordersyubei, mtx_existing_ordersyubei);
        bool yubeiwanbi = false;
        {
            std::lock_guard<std::mutex> lock(mtx_existing_ordersyubei);
        if (existing_ordersyubei.size() > 2) 
        {  // 确保队列长度大于 2
        Order& last_order = existing_ordersyubei[existing_ordersyubei.size() - 1];
        Order& second_last_order = existing_ordersyubei[existing_ordersyubei.size() - 2];
        long long time_diff = last_order.timestamp - second_last_order.timestamp;
            if (time_diff == 0 &&
                last_order.direction == second_last_order.direction) {

                    double total_quantity = second_last_order.quantity + last_order.quantity;
                    double weighted_price = (second_last_order.price * second_last_order.quantity +
                                    last_order.price * last_order.quantity) / total_quantity;
                    weighted_price = std::round(weighted_price * 1e11) / 1e11;  // 高精度存储均价
                      // 更新第二个订单
                    second_last_order.price = weighted_price;
                    second_last_order.quantity = total_quantity;
        // 删除最后一个订单
            existing_ordersyubei.pop_back();
        }
        else {
            yubeiwanbi = true;
        }
        }
        }
        if (yubeiwanbi) {  // 将新订单加入现有订单队列
        Order new_order = existing_ordersyubei[existing_ordersyubei.size() - 2];
        // 线程安全地将新订单添加到 existing_orders 下面是我定义的安全添加函数
        safe_push_back(existing_orders, new_order, max_existing_orders, mtx_existing_orders);
        构建高频量化策略的另一个关键，是准确识别交易者的实际开仓数量与背后的资金规模。虽然部分交易所公开了开仓计算公式，但由于其涉及较多隐藏参数与边界条件，理论公式未必与实战一致。因此，我尝试通过实证方法加以验证，并收集了大量真实下单数据，构造如下实验。为方便，这里展示我收集的10几条关于狗狗币的数据——简单写了个Python代码加以验证
        输入包含标记价格、最新价格、余额、杠杆、和最大可做多做空量的实验结果。
        # 用于估算最大可做空数量的估算函数
def estimate_max_short(mark_price, latest_price, balance, leverage, max_long, max_short):
    return int(balance * leverage / (1.001 * latest_price))
# 示例输入
data = [
(0.31624, 0.31615, 30.94635585, 75, 6928 ,7297),
(0.31645, 0.31635, 30.94635585, 62, 5792 ,6032),
(0.31655, 0.31645, 30.94635585, 8, 777 ,781),
(0.31659, 0.31650, 30.94635585, 51, 4777 ,4958),
(0.31656, 0.31653, 86.94635585, 23, 6179 ,6308),
(0.31660, 0.31656, 86.94635585, 23, 6183 ,6307),
(0.31662, 0.31662, 86.94635585, 57, 14749 ,15587),
(0.31655, 0.31653, 86.94635585, 40, 10580 ,10976),
(0.31658, 0.31655, 86.94635585, 40, 10592 ,10975),
(0.31659, 0.31658, 86.94635585, 40, 10618 ,10976),
(0.31657, 0.31659, 86.94635585, 10, 2714 ,2743),
(0.31654, 0.31657, 86.94635585, 10, 2713 ,2743),
(0.24079, 0.24078, 1.09, 75, 315 ,338),
(0.31651, 0.31647, 1086.94635585, 4, 13678 ,13725),
(0.31649, 0.31651, 1086.94635585, 30, 99535 ,102716),
(0.31652, 0.31658, 1086.94635585, 45, 146061 ,153810),
(0.31658, 0.31652, 1086.94635585, 45, 148449 ,153839),
(0.31665, 0.31665, 1086.94635585, 32, 106219, 109559)
]
correct_count = 0
for i, row in enumerate(data):
    pred = estimate_max_short(*row)
    is_correct = pred == row[-1]  # max_short
    print(f"[{i}] 预测: {pred}, 实际: {row[-1]} -> {'✅' if is_correct else '❌'}, 差距: {pred - row[-1]},杠杆与本金{row[-3]}、{row[-4]}")
    if is_correct:
        correct_count += 1

print(f"\n✅ 共预测正确: {correct_count} / {len(data)} 条")

示例输出：
[0] 预测: 7334, 实际: 7297 -> ❌, 差距: 37,杠杆与本金75、30.94635585
[1] 预测: 6058, 实际: 6032 -> ❌, 差距: 26,杠杆与本金62、30.94635585
[2] 预测: 781, 实际: 781 -> ✅, 差距: 0,杠杆与本金8、30.94635585
[3] 预测: 4981, 实际: 4958 -> ❌, 差距: 23,杠杆与本金51、30.94635585
[4] 预测: 6311, 实际: 6308 -> ❌, 差距: 3,杠杆与本金23、86.94635585
[5] 预测: 6310, 实际: 6307 -> ❌, 差距: 3,杠杆与本金23、86.94635585
[6] 预测: 15637, 实际: 15587 -> ❌, 差距: 50,杠杆与本金57、86.94635585
[7] 预测: 10976, 实际: 10976 -> ✅, 差距: 0,杠杆与本金40、86.94635585
[8] 预测: 10975, 实际: 10975 -> ✅, 差距: 0,杠杆与本金40、86.94635585
[9] 预测: 10974, 实际: 10976 -> ❌, 差距: -2,杠杆与本金40、86.94635585
[10] 预测: 2743, 实际: 2743 -> ✅, 差距: 0,杠杆与本金10、86.94635585
[11] 预测: 2743, 实际: 2743 -> ✅, 差距: 0,杠杆与本金10、86.94635585
[12] 预测: 339, 实际: 338 -> ❌, 差距: 1,杠杆与本金75、1.09
[13] 预测: 13724, 实际: 13725 -> ❌, 差距: -1,杠杆与本金4、1086.94635585
[14] 预测: 102921, 实际: 102716 -> ❌, 差距: 205,杠杆与本金30、1086.94635585
[15] 预测: 154348, 实际: 153810 -> ❌, 差距: 538,杠杆与本金45、1086.94635585
[16] 预测: 154377, 实际: 153839 -> ❌, 差距: 538,杠杆与本金45、1086.94635585
[17] 预测: 109734, 实际: 109559 -> ❌, 差距: 175,杠杆与本金32、1086.94635585

✅ 共预测正确: 5 / 18 条

从实验结果看，该估算公式在部分场景下预测准确，尤其在小额账户或中低杠杆时表现较佳。然而当杠杆较高或余额增大后，预测误差迅速放大。在18组样本中，仅有5组完全命中（误差为0），部分样本误差较大，说明实际成交数量存在非线性边界或隐藏规则，可能与交易所的整数截断、滑点、风险缓释机制等有关。一开始我是想用拟合手段拟合的，使用了基于遗传算法的符号回归等方法，依旧拟合不出来，我想这是交易所的黑箱。

长时间的建模停滞，使我逐渐意识到：我无需准确还原每一个交易者的仓位结构。策略的成功不依赖于全面识别，而是抓住那些可以确定的机会即可。因此我在实用中放弃了对100%拟合的执念，转而允许 ±1 单位的误差窗口，只要最终的反推落入这个容差区间，即视为有效识别。

同理还有关于做多的公式，较为复杂，在文章发布日期是：
int(balance / ((latest_price * 1.001) / leverage + abs(mark_price - (latest_price * 1.001))))
此处不再赘述。总的来说，策略的关键在于：不是拟合一切，而是精准捕捉可以识别的那部分机会。

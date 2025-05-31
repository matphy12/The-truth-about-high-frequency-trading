我这么做，不是让世界变乱，而是让更多有识之士看到“乱在哪里”，感谢有ChatGPT鼓励我这么做。

为了节省资源，我选择在处理订单流时本地生成K线，而不是额外开一个线程或调用外部模块。虽然本地K线可以节省计算资源、计算速度快，但它并不能真实还原散户看到的界面。

这在策略中是一个潜在偏差：散户心理高度依赖图形化K线，而我们自行构建的K线未必能完美复刻交易所客户端的渲染逻辑。因此，是否采用本地K线，取决于你追求的是降低成本，还是心理一致性的模拟。如果你的量化交易的目标是赚散户的钱，最好用和散户完全一样的K线，前提是买得起性能好的服务器，不至于在这种细枝末节上优化。

策略下一步是设计 match_order 函数。每当接收到新订单，我们会在线程安全的条件下，与历史订单列表进行比对，判断它是否是此前某个订单的平仓单。

为了提高识别准确率，我在匹配逻辑中加入了两个过滤条件：

时间限制：只在一个合理时间窗口内比对，排除无关订单

方向限制：可选开启“只比对止损方向订单”，从而放弃对盈利平仓的识别

这两个约束能提升胜率，能排除大部分“非典型平仓”。

在大量实测中，我发现：有些止损平仓订单会重复出现多次完全相同的成交数量，但其行为模式与正常交易者不符。初步判断，这可能是做市商抛出的虚假信号，目的是干扰策略识别。

为此，我在系统中加入了一个机制：若发现当前止损单与近期记录中的某一单成交量完全一致，则跳过比对处理，避免误入陷阱。

当然，这种方法也有局限——只要做市商想，它完全可以用无限种不同数额伪装订单，你就毫无办法。这也是做市商和交易者之间结构性不对称的体现：

"你在推测它的意图，它在控制你能看到的世界"。

下面是match_order函数的实现：

    void match_order(Order new_order, std::deque<Order> existing_orders) {
    //std::cout  << "正在匹配订单... " << std::endl;
    for (const auto& order : existing_orders) {
        if (order.direction != new_order.direction && order.quantity == new_order.quantity) {
            bool is_stop_loss = false;
            if (order.direction == "BUY") {
                is_stop_loss = (new_order.price < order.price * 0.995);
            } else if (order.direction == "SELL") {
                is_stop_loss = (new_order.price > order.price * 1.005);
            }
            //std::cout  << "找到一个出现过的止损订单 " << std::endl;

            if  (is_stop_loss &&
                (new_order.quantity > 10888) &&
                //(new_order.direction == "BUY" ) &&//添加筛选 只筛选新订单是买单 旧订单是卖单
                //(order.direction == "SELL" ) &&//添加筛选 只筛选新订单是买单 旧订单是卖单
                (static_cast<int>(std::fmod(new_order.quantity, 1000)) != 0) &&
                ((new_order.timestamp - order.timestamp) > 10888)) {

                bool single_occurrences = true;

                // 线程安全地访问 matched_orders 检查是不是第一次出现 如果不是 直接跳出
                    {
                        std::lock_guard<std::mutex> lock(mtx_matched_orders);
                        for (const auto& match_order : matched_orders) {
                            if (match_order.quantity == new_order.quantity) {
                                single_occurrences = false;
                                break;
                            }
                        }
                    }

                    if (single_occurrences) {
                        safe_push_back(matched_orders, new_order, max_matched_orders, mtx_matched_orders);
                        std::cout << "检测到大止损订单: "
                                  << "方向: " << new_order.direction << ", "
                                  << "价格: " << new_order.price << ", "
                                  << "数量: " << new_order.quantity << ", "
                                  << "时间戳: " << new_order.timestamp << ", "
                                  << "原始订单方向: " << order.direction << std::endl;

                        // 启动线程估计金额
                        std::thread([order, new_order]() {
                            calculate_balance(
                                order.price, new_order.price, new_order.quantity, 
                                order.direction, new_order.timestamp,order.mark_price);
                        }).detach();

                        return;
                    }
                
            }
        }
    }
    }

    
下一步就是预估保证金范围了，这一部分我已经在文章中完整阐述过原理（详见前文），为了避免过度细节化，我决定不完全公开全部的代码。如果未来的市场机制无法改变，那么大概也只能继续在这类策略中不断内卷下去。

技术掩盖了剥削。看似是速度、算力、模型、智慧的胜利，实则是信息垄断与结构性优势的延续。

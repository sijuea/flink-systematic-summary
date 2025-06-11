---
### 1.概念
主要包括：连接器，格式类，表/SQL扩展，流计算高级功能，运维/监控

**当前：流计算高级功能库（CEP、ML、Graph）** 包括 CEP（复杂事件处理）、Flink ML（机器学习）、Gelly（图计算）。

---

### 2.原理
**CEP 使用 NFA 模型进行事件模式识别。** 
**流数据—>CEP规则—>输出结构【类似Pattern用法】**
  
  * 每个事件模式定义一组状态及状态间的转换条件（如事件类型、时间窗口等）。
  * 输入流中的事件被逐步匹配，NFA 维护多个可能的状态路径（因非确定性）。
  * 当满足终态条件时，输出匹配结果（符合预定义事件序列的复杂事件）。
** NFA模型(非确定性有限自动机) **        
  * NFA 是一种自动机模型，用于处理字符串或事件序列的匹配问题。它由一组状态（states）、输入符号（events）、状态转移函数和一个或多个终止状态组成。
        
  * **非确定性** 在同一输入下，NFA 允许从一个状态转移到多个状态（即存在多条可能的路径），它并不要求每个输入只能对应一个唯一的后继状态。
        
**匹配过程**
  * 当输入事件流到来时，NFA 会同时维护多个可能的状态路径（状态集合），任何一条路径到达终止状态时即匹配成功。

---

### 3. 实现
**CEP案例**
```
  //检测用户连续三次失败登录事件
// 定义事件类
public class LoginEvent {
    public String userId;
    public String eventType; // "fail" or "success"
    public long eventTime;
}

// 构建 CEP Pattern：连续三次失败
Pattern<LoginEvent, ?> pattern = Pattern.<LoginEvent>begin("firstFail")
    .where(evt -> evt.eventType.equals("fail"))
    .next("secondFail")
    .where(evt -> evt.eventType.equals("fail"))
    .next("thirdFail")
    .where(evt -> evt.eventType.equals("fail"))
    .within(Time.minutes(5));

// 应用 CEP
PatternStream<LoginEvent> patternStream = CEP.pattern(loginEventStream, pattern);

// 事件匹配处理
patternStream.select(patternMatches -> {
    List<LoginEvent> fails = patternMatches.get("firstFail");
    fails.addAll(patternMatches.get("secondFail"));
    fails.addAll(patternMatches.get("thirdFail"));
    return "User " + fails.get(0).userId + " failed login 3 times!";
});

```
**常用方法**

| 方法 | 说明 | 示例或备注 |
| --- | --- | --- |
| `begin()` | 定义第一个事件模式 | `Pattern.begin("start")` |
| `next()` | 定义紧接着的事件（严格顺序） | `pattern.next("middle")` |
| `followedBy()` | 定义后续事件（非严格顺序） | `pattern.followedBy("middle")` |
| `within(Time)` | 限定事件序列的最大时间范围 | `pattern.within(Time.minutes(5))` |
| `where()` | 添加过滤条件（布尔表达式） | `.where(e -> e.getType().equals("click"))` |
| `or()` | 为同一个模式添加额外条件 | `.where(...).or(...)` |
| `times(int n)` | 指定事件必须出现固定次数 | `.times(3)`：恰好 3 次 |
| `optional()` | 指定某事件为可选事件 | `.optional()` |
| `oneOrMore()` | 匹配一个或多个重复事件 | `.oneOrMore()` |

**PatternStream 处理方法**

| 方法 | 说明 |
| --- | --- |
| `select(PatternSelectFunction)` | 对完整模式进行处理，输出结果 |
| `flatSelect(PatternFlatSelectFunction)` | 输出 0~多条结果（灵活） |
| `process(PatternProcessFunction)` | 更强大的低级接口，支持定时器、侧输出等 |
| `injected()` | 结合广播状态动态注入模式（高级用法） |
---

### 4.场景
**复杂事件规则识别**
  * 使用 CEP 扩展库实现风控、入侵检测等规则链场景。
  * 可构造模式，如 A → B → not(C) within 10min。
---

### 5.常见问题
**Q：CEP 的模式匹配为什么用 NFA 而不是 DFA？**
> A：
> - NFA（非确定性有限自动机）支持同时跟踪多条匹配路径，适合复杂事件模式（如重复、循环、可选事件）
> - DFA（确定性有限自动机）状态唯一，不支持多路径并行匹配，表达能力受限
>  CEP 需要识别多样复杂事件序列，故采用 NFA 实现模式匹配。

**Q: CEP 中事件时间和处理时间的区别对匹配有何影响？**
> A：
> - **事件时间**：事件实际发生的时间，保证正确的时间顺序，适合乱序事件处理。
> - **处理时间**：事件被处理的系统时间，实时性强但可能乱序。

**Q: CEP 匹配模式中，next() 和 followedBy() 有何区别？**
>A:
>- `next()` 表示严格紧邻的事件序列，匹配时不允许其他事件插入。
>- `followedBy()` 允许中间插入无关事件，匹配更宽松。

**Q：CEP 模式如何支持重复匹配和跳过部分事件？**
> A:
> - 使用量词如 `oneOrMore()`, `times()`, `optional()` 控制重复匹配。
> - 通过 `skipToNext()`, `skipPastLastEvent()` 控制跳过匹配事件。

**Q: CEP 匹配结果如何输出？如何保证结果顺序？**
>A:
>- 匹配结果通过 `select()` 或 `flatSelect()` 输出到下游。
>- 结果顺序默认基于事件时间排序，可结合 Watermark 保证事件顺序一致性。

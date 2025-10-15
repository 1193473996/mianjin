# 十万级别用户登录数据，包括uid、loginTime、logoutTime，怎么统计登录峰值及对应的时间段
好的，我们抛开具体编程语言，专注于描述解决这个问题的算法思想和核心步骤。

### 核心算法思想

这个问题的本质是 **“最大重叠区间”** 问题。目标是找到所有用户登录时间段（即区间 `[loginTime, logoutTime]`）在时间轴上最大程度重叠的那个点（或时段）以及重叠的数量。

最适合此问题的经典算法是 **“事件扫描法”** 或 **“时间点计数法”**。

---

### 算法步骤描述

#### 第一步：数据转换 - 生成事件流

将每个用户的**时间段**信息，分解为在时间轴上离散发生的**事件点**。

1.  **定义两类事件**：
    *   **`登录事件 (IN)`**：发生在 `loginTime`。代表一个用户上线，**并发用户数 +1**。
    *   **`登出事件 (OUT)`**：发生在 `logoutTime`。代表一个用户下线，**并发用户数 -1`。

2.  **构建事件列表**：
    *   为数据中的每一条记录生成两个事件：一个 `IN` 事件和一个 `OUT` 事件。
    *   将所有事件（来自所有用户）放入同一个列表中。

#### 第二步：时间轴扫描 - 计算并发数

按时间顺序处理这些事件，动态计算并发用户数。

1.  **排序**：
    *   将第一步生成的事件列表，按照**事件发生的时间戳**进行**升序**排序。
    *   **重要规则**：如果两个事件时间戳完全相同，必须定义处理优先级。通常约定 **`登出 (OUT)` 事件优先于 `登录 (IN)` 事件** 处理。这样可以避免在精确的同一秒内，先算登录后算登出而虚高瞬时峰值。

2.  **扫描与计数**：
    *   初始化一个计数器 `current_count = 0`，代表当前的并发用户数。
    *   初始化一个变量 `max_count = 0`，用于记录遍历过程中遇到的最大并发数。
    *   从头到尾遍历排序后的事件列表：
        *   如果遇到一个 `IN` 事件，执行 `current_count = current_count + 1`。
        *   如果遇到一个 `OUT` 事件，执行 `current_count = current_count - 1`。
    *   在每次更新 `current_count` 之后，将其与 `max_count` 比较：
        *   如果 `current_count > max_count`，则更新 `max_count = current_count`。这个 `max_count` 就是我们要找的**登录峰值**。

#### 第三步：峰值时段识别 - 记录时间区间

第二步只能找到峰值的**大小**，要找到峰值发生的**时间段**，需要在扫描过程中增加记录机制。

1.  **追踪峰值平台的起止**：
    *   除了 `current_count` 和 `max_count`，我们还需要一个变量 `peak_start_time`，用于临时记录当前这个“峰值平台”可能开始的时间。
    *   在扫描过程中：
        *   当 `current_count` **上升至** `max_count` 时（即刚刚刷新了历史最高纪录），记录下当前这个事件的时间戳作为 `peak_start_time`。这标志着一段新的峰值时段的开始。
        *   当 `current_count` **从** `max_count` **下降**时（即 `current_count = max_count - 1`），记录下当前这个事件的时间戳作为 `peak_end_time`。此时，`[peak_start_time, peak_end_time]` 就是一个完整的峰值时间段。将其保存到结果列表中。
        *   **特殊情况处理**：如果数据扫描结束时，`current_count` 仍然等于 `max_count`，那么最后一个峰值时段尚未闭合，应将最后一个事件时间作为 `peak_end_time`，或者根据业务逻辑将其结束时间设置为“至今”。

2.  **处理多个峰值平台**：
    *   上述逻辑可以自然地处理存在多个不连续的峰值时段的情况。每次 `current_count` 重新达到 `max_count`，都会开启一个新的 `peak_start_time`。

### 算法总结

*   **输入**：包含 `(uid, loginTime, logoutTime)` 的列表。
*   **输出**：峰值并发数 `max_count`，以及一个或多个峰值时间段 `[(start1, end1), (start2, end2), ...]`。
*   **关键**：将区间问题转化为基于时间排序的事件序列问题，通过一次线性扫描（排序后）即可高效解决。
*   **优势**：算法时间复杂度主要由排序步骤决定，为 **O(N log N)**，其中 N 是用户记录数的两倍（因为每条记录产生两个事件）。这对于十万级别数据量是非常高效的。


基于前面的算法思想，以下是完整的Java代码实现：

```java
import java.util.*;

public class LoginPeakAnalyzer {

    public static class LoginInfo {
        public int usrId;
        public int loginTimestamp;
        public int logoutTimestamp;

        public LoginInfo(int usrId, int loginTimestamp, int logoutTimestamp) {
            this.usrId = usrId;
            this.loginTimestamp = loginTimestamp;
            this.logoutTimestamp = logoutTimestamp;
        }
    }

    public static class PeakResult {
        public int peakCount;
        public List<int[]> peakPeriods;

        public PeakResult(int peakCount, List<int[]> peakPeriods) {
            this.peakCount = peakCount;
            this.peakPeriods = peakPeriods;
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("登录峰值: ").append(peakCount).append(" 人\n");
            sb.append("峰值时间段:\n");
            for (int i = 0; i < peakPeriods.size(); i++) {
                int[] period = peakPeriods.get(i);
                sb.append(String.format("  时间段 %d: %d ~ %d (持续: %d秒)\n", 
                    i + 1, period[0], period[1], period[1] - period[0]));
            }
            return sb.toString();
        }
    }

    // 事件类型枚举
    private enum EventType {
        LOGIN(1),  // 登录事件，计数+1
        LOGOUT(-1); // 登出事件，计数-1

        final int value;

        EventType(int value) {
            this.value = value;
        }
    }

    // 事件类
    private static class Event {
        final int timestamp;
        final EventType type;

        Event(int timestamp, EventType type) {
            this.timestamp = timestamp;
            this.type = type;
        }
    }

    /**
     * 分析登录数据，统计峰值及时间段
     * @param loginData 登录数据列表
     * @return 峰值结果
     */
    public static PeakResult analyzeLoginPeak(List<LoginInfo> loginData) {
        if (loginData == null || loginData.isEmpty()) {
            return new PeakResult(0, new ArrayList<>());
        }

        // 第一步：生成事件流
        List<Event> events = generateEvents(loginData);

        // 第二步：按时间戳排序事件，时间相同则登出优先
        events.sort((e1, e2) -> {
            if (e1.timestamp != e2.timestamp) {
                return Integer.compare(e1.timestamp, e2.timestamp);
            }
            // 时间戳相同时，登出事件优先于登录事件
            return Integer.compare(e1.type.value, e2.type.value);
        });

        // 第三步：扫描事件流，计算并发数和峰值时段
        return scanEvents(events);
    }

    /**
     * 生成事件流
     */
    private static List<Event> generateEvents(List<LoginInfo> loginData) {
        List<Event> events = new ArrayList<>(loginData.size() * 2);
        
        for (LoginInfo info : loginData) {
            // 添加登录事件
            events.add(new Event(info.loginTimestamp, EventType.LOGIN));
            // 添加登出事件
            events.add(new Event(info.logoutTimestamp, EventType.LOGOUT));
        }
        
        return events;
    }

    /**
     * 扫描事件流，计算峰值
     */
    private static PeakResult scanEvents(List<Event> events) {
        int currentCount = 0;
        int maxCount = 0;
        Integer currentPeakStart = null;
        List<int[]> peakPeriods = new ArrayList<>();

        for (Event event : events) {
            // 更新当前并发用户数
            currentCount += event.type.value;

            // 检查是否达到新的峰值
            if (currentCount > maxCount) {
                // 发现新的更高峰值，重置记录
                maxCount = currentCount;
                peakPeriods.clear();
                currentPeakStart = event.timestamp;
            } 
            // 检查是否持平峰值
            else if (currentCount == maxCount && currentPeakStart == null) {
                // 重新回到峰值水平，开始新的峰值时段
                currentPeakStart = event.timestamp;
            }
            // 检查是否从峰值下降
            else if (currentCount == maxCount - 1 && currentPeakStart != null) {
                // 从峰值下降，记录完整的峰值时段
                peakPeriods.add(new int[]{currentPeakStart, event.timestamp});
                currentPeakStart = null;
            }
        }

        // 处理扫描结束时仍处于峰值的情况
        if (currentPeakStart != null && maxCount > 0) {
            int lastTimestamp = events.get(events.size() - 1).timestamp;
            peakPeriods.add(new int[]{currentPeakStart, lastTimestamp});
        }

        return new PeakResult(maxCount, peakPeriods);
    }

    // 测试用例
    public static void main(String[] args) {
        // 构造测试数据
        List<LoginInfo> testData = Arrays.asList(
            new LoginInfo(1, 1000, 1200),  // 用户1：1000秒登录，1200秒登出
            new LoginInfo(2, 1010, 1150),  // 用户2：1010秒登录，1150秒登出  
            new LoginInfo(3, 1050, 1300),  // 用户3：1050秒登录，1300秒登出
            new LoginInfo(4, 1100, 1250),  // 用户4：1100秒登录，1250秒登出
            new LoginInfo(5, 1120, 1180)   // 用户5：1120秒登录，1180秒登出
        );

        PeakResult result = analyzeLoginPeak(testData);
        System.out.println(result);

        // 测试边缘情况：空数据
        System.out.println("空数据测试:");
        PeakResult emptyResult = analyzeLoginPeak(new ArrayList<>());
        System.out.println(emptyResult);

        // 测试边缘情况：同时登录登出
        List<LoginInfo> edgeCaseData = Arrays.asList(
            new LoginInfo(1, 1000, 1100),
            new LoginInfo(2, 1100, 1200)  // 用户1登出和用户2登录同时发生
        );
        System.out.println("边缘情况测试:");
        PeakResult edgeResult = analyzeLoginPeak(edgeCaseData);
        System.out.println(edgeResult);
    }
}
```

### 代码核心要点说明：

1. **数据结构设计**：
   - `LoginInfo`: 输入数据结构
   - `Event`: 内部事件表示，包含时间戳和类型
   - `PeakResult`: 返回结果，包含峰值人数和峰值时段列表

2. **关键算法步骤**：
   - **生成事件流**：将每个登录记录转换为登录和登出两个事件
   - **事件排序**：按时间戳排序，时间相同时登出优先（避免虚高峰值）
   - **扫描计算**：维护当前并发数，动态追踪峰值时段的开始和结束

3. **边界情况处理**：
   - 空输入数据
   - 事件时间戳相同的情况
   - 扫描结束时仍处于峰值的情况
   - 多个不连续峰值时段的处理

4. **时间复杂度**：O(N log N)，主要来自排序操作
5. **空间复杂度**：O(N)，用于存储事件流

这个实现能够准确找出登录峰值及其对应的所有时间段，适用于十万级别数据量的场景。


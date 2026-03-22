这道题是考察**海量数据处理能力**的经典场景题。9亿条数据，即使每条只用1个字节，也要 ~9GB，根本不可能直接加载进内存。

面试官想看的不是你会背哪种算法，而是你**如何把一个大问题拆解成若干可解决的小问题**，以及你的工程化思维。

---

### 1\. 核心结论 (Core Conclusion)

针对9亿IP地址去重，**没有银弹**。最佳策略是**分层过滤、分而治之**：

1.  **第一层（内存级）：** 用 **BitMap（位图）** 将IPv4地址映射为比特位，极限情况下只需 **512MB 内存**，单线程即可高效完成去重。
    
2.  **第二层（磁盘级）：** 如果数据量大到连512MB也放不下，就用 **分治 + 外部排序**（Hash分流到多个小文件，分别排序，合并去重）。
    
3.  **第三层（集群级）：** 如果单机跑不动，就用 **MapReduce / Spark** 分布式并行处理。
    

---

### 2\. 深度解析：四种方案层层递进 (Detailed Breakdown)

#### 方案一：位图法 BitMap（推荐，单机 512MB 搞定）

**为什么能用位图？**

因为 IPv4 地址的本质是一个 **32位的无符号整数**（范围 `0 ~ 4,294,967,295`），共约 **43亿** 个可能的值。

**原理：** 用一个比特（Bit）来标记某个数字是否存在。43亿个比特 = 43亿 / 8 = **512MB**。

```java
public class IpDeduplicatorByBitMap {
    // 使用字节数组模拟 BitMap
    // 2^32 需要 2^29 个字节 = 512MB
    private byte[] bitmap = new byte[1 << 29]; 

    // 将IP地址转换为 32 位整数
    private long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        long result = 0;
        for (String part : parts) {
            result = (result << 8) | Integer.parseInt(part);
        }
        return result;
    }

    // 添加IP到 BitMap（去重逻辑核心）
    public boolean addAndCheckDup(String ip) {
        long offset = ipToLong(ip);
        int byteIndex = (int) (offset >> 3); // offset / 8
        int bitIndex = (int) (offset & 7);  // offset % 8
        
        // 查看当前比特位是否为1（已经存在）
        boolean existed = (bitmap[byteIndex] & (1 << bitIndex)) != 0;
        
        // 设置为1（标记为已存在）
        bitmap[byteIndex] |= (1 << bitIndex);
        
        return existed; // true = 重复, false = 新IP
    }

    public static void main(String[] args) {
        IpDeduplicatorByBitMap dedup = new IpDeduplicatorByBitMap();
        // 模拟读取9亿条IP并去重
        // 实际读取文件，每行一个IP，调用 addAndCheckDup
    }
}
```

**优点：** 内存占用固定（512MB），时间复杂度 O(1)，代码简洁。 **缺点：** 只能处理IPv4，IPv6需要更大的位图（2^128位，约 4 亿 TB，根本不可行）。

---

#### 方案二：布隆过滤器 BloomFilter（内存更省，但有代价）

**适用场景：** 如果你**可以容忍极低的误判率**（把不重复的判断为重复，但不会漏掉真正的重复），布隆过滤器是更优雅的方案。

**原理：** 使用多个哈希函数，将IP映射到位图的多个位置。只有当所有位置都为1时，才认为IP存在。空间占用可以低到**几十MB**。

```java
// 使用 Guava 的 BloomFilter（实际工程强烈推荐）
public class IpDeduplicatorByBloomFilter {
    
    // 预期插入9亿条，去重场景期望误判率越低越好
    // 以下参数是估算值，实际需精确计算
    private BloomFilter<String> bloomFilter = BloomFilter.create(
        Funnels.stringFunnel(StandardCharsets.UTF_8),
        900_000_000L,   // 预期插入数量
        0.0001          // 期望误判率 0.01%（实际可能略高）
    );

    public boolean addAndCheckDup(String ip) {
        return bloomFilter.put(ip); // put返回false表示不重复，true表示可能重复
    }
}
```

**关键陷阱：** 布隆过滤器存在**假阳性（False Positive）**，即不重复的IP会被误判为重复。但这在去重场景通常可接受（宁可少算一条，不可重复处理）。 **解决假阳性：** 如果业务要求100%准确，可以在布隆过滤器判定为重复后，再去磁盘或数据库查一下原始记录做二次确认。

---

#### 方案三：分治 + 外部排序（磁盘友好，适合更大数据）

**场景：** 如果数据大到连512MB位图都无法接受（比如数据里有大量无效IP，或者需要兼容IPv6），就要把战场拉到磁盘。

**核心思想：** 把9亿条数据**分到多个小文件**里，每个小文件单独加载进内存去重，最后再合并。

```java
public class IpDeduplicatorByPartition {
    
    private static final int BATCH_SIZE = 1_000_000;  // 每批100万条
    private static final int FILE_COUNT = 900;        // 分成900个文件

    public void deduplicate(String inputFile) throws IOException {
        // ============ 第一步：Hash分流 ============
        // 用 HashMap<文件编号, List<IP>> 把数据打散到多个小文件
        Map<Integer, BufferedWriter> writers = new HashMap<>();
        
        try (BufferedReader reader = new BufferedReader(
                new FileReader(inputFile), 1024 * 1024)) { // 1MB大文件缓冲
            
            String ip;
            while ((ip = reader.readLine()) != null) {
                int bucket = Math.abs(ip.hashCode() % FILE_COUNT);
                
                // 获取该 bucket 对应的写入流（懒加载）
                BufferedWriter writer = writers.computeIfAbsent(bucket, 
                    k -> new BufferedWriter(new FileWriter("temp_" + k + ".txt")));
                writer.write(ip);
                writer.newLine();
            }
        }
        
        // ============ 第二步：分别排序去重 ============
        // 对每个小文件在内存内排序 + 去重（可以用 Set 或 排序后单次遍历）
        for (int i = 0; i < FILE_COUNT; i++) {
            sortAndDedupInFile("temp_" + i + ".txt", "dedup_" + i + ".txt");
        }
        
        // ============ 第三步：多路归并合并 ============
        // 将900个已去重的小文件，用最小堆进行多路归并，最终输出一个有序去重文件
        mergeFiles(FILE_COUNT, "final_result.txt");
        
        // 清理临时文件...
    }

    private void sortAndDedupInFile(String input, String output) throws IOException {
        List<String> lines = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(input))) {
            String ip;
            while ((ip = reader.readLine()) != null) lines.add(ip);
        }
        // Java 的 TimSort 时间复杂度 O(n log n)
        lines.sort(String::compareTo);
        
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(output))) {
            String prev = null;
            for (String ip : lines) {
                if (!ip.equals(prev)) { // 去重核心
                    writer.write(ip);
                    writer.newLine();
                    prev = ip;
                }
            }
        }
    }
}
```

**缺点：** 需要 2-3 倍磁盘空间（原始文件 + 临时文件 + 最终结果）。但优点是**天然支持分布式**，每个小文件可以分配到不同机器处理。

---

#### 方案四：MapReduce 分布式处理（海量数据终极方案）

**架构图（文字版）：**

```text
Input (9亿IP)
    │
    ▼
【Map Phase】
    │ 每个 Mapper 读取一批 IP
    │ 对 IP 做 Hash，输出 (Hash(IP) % N, IP)
    ▼
【Shuffle Phase】 自动按 Key 分区
    │
    ▼
【Reduce Phase】
    │ 每个 Reducer 拿到的是同一个桶内的所有 IP
    │ 在 Reducer 内部用 HashSet 去重
    ▼
Output (N 个已去重的小文件)
    │
    ▼
【最终合并】 多路归并输出全局去重结果
```

**伪代码（Spark 版本，代码更简洁）：**

```java
public class SparkIpDedup {
    public static void main(String[] args) {
        SparkConf conf = new SparkConf().setAppName("IP去重");
        JavaSparkContext sc = new JavaSparkContext(conf);
        
        // 读取HDFS上的大文件
        JavaRDD<String> ips = sc.textFile("hdfs:///data/ips.txt");
        
        // 直接用 distinct() 去重（底层也是 Hash 分区 +  Reduce 端去重）
        JavaRDD<String> uniqueIps = ips.distinct();
        
        // 保存结果
        uniqueIps.saveAsTextFile("hdfs:///output/unique_ips");
    }
}
```

---

### 3\. 面试官视角 (The Interviewer's Lens)

**高频追问与致命陷阱：**

1.  **陷阱：BitMap 的 512MB 怎么算出来的？**
    

-   *满分回答：* IPv4 共 2^32 = 43亿 个可能的地址。每个地址用 1 个比特标记，43亿 / 8 = 5.37亿字节 ≈ **512MB**。这就是 BitMap 的极限内存消耗。
    

2.  **追问：BitMap 方案里，如果 IP 文件里有大量无效/格式错误的 IP，怎么处理？**
    

-   *满分回答：* 在 `addAndCheckDup` 前加一层正则校验，或者自己解析（判断是否符合4段0-255的格式）。无效的 IP 直接跳过，不消耗位图空间。
    

3.  **追问：去重后的结果怎么存储？还能怎么用？**
    

-   *工程延伸：* 可以将去重结果转成**布隆过滤器**持久化到磁盘，下次启动直接加载，快速判断新IP是否重复。或者结合地理位置库（GeoIP）统计各地区的UV。
    

---

### 4\. 总结 (Summary)

| 方案  | 内存占用 | 准确性 | 适用场景 |
| --- | --- | --- | --- |
| **BitMap** | 512MB（固定） | 100% 精确 | IPv4去重，内存足够 |
| **布隆过滤器** | 几十MB | 有极低误判率 | 海量数据，允许容忍误差 |
| **分治+外部排序** | 取决于每批大小 | 100% 精确 | 超大文件，必须落盘 |
| **MapReduce/Spark** | 可扩展 | 100% 精确 | PB级数据，集群环境 |

---

**面试官点评：** “这道题你能从 BitMap 讲到分治再讲到分布式，并且在每种方案后都能说出其局限性（比如布隆过滤器的假阳性、分治的磁盘开销），说明你具备完整的大数据工程视野，而不是只会背一个标准答案。”

**海量数据处理聊完了，如果你想继续挑战，可以聊聊：Redis 的 HyperLogLog（也是做去重的，你知道它和 BitMap 的区别吗？），或者 MySQL 的分库分表如何设计全局唯一ID？**
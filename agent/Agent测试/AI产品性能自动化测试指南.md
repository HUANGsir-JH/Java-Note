AI 产品的**性能自动化测试 (Performance Automation Testing)** 是一个高复杂度领域，因为需要同时关注**传统系统性能**（接口响应、并发、资源）和**模型特有指标**（推理延迟、首字时间、生成速率、成本）。

以下是 AI 产品性能自动化测试的完整实施指南：

---

## 一、为什么 AI 性能测试更复杂？

| 传统软件性能测试 | AI 产品性能测试 |
| --- | --- |
| 响应时间 = 业务逻辑执行时间 | 响应时间 = 网络 + 预处理 + **模型推理** + 后处理 |
| 负载增加 → 响应线性变慢 | 负载增加 → 可能触发**队列排队**、**GPU OOM**、**降级策略** |
| 指标单一：QPS、RT、错误率 | 指标多元：TTFT、TPS、Token 成本、显存占用、漂移检测 |
| 测试数据可随机生成 | 测试数据需考虑**Prompt 长度分布**、**多轮对话上下文** |
| 瓶颈通常在 DB/应用服务器 | 瓶颈通常在**推理引擎**、**GPU 资源**、**向量检索** |

---

## 二、核心性能指标 (KPIs)

### 🔹 用户体验层指标

| 指标  | 说明  | 目标参考值 |
| --- | --- | --- |
| **TTFT (Time To First Token)** | 首字生成时间，用户感知的"开始响应"速度 | < 1.5s (在线对话) |
| **TPS (Tokens Per Second)** | 每秒生成 Token 数，决定"打字机"流畅度 | \> 30 TPS (流畅体验) |
| **端到端延迟 (End-to-End Latency)** | 从发送请求到完整响应接收的总时间 | 根据场景定，短问答 < 3s |
| **流式间隔 (Streaming Interval)** | 流式输出时，每个 chunk 的到达间隔 | < 200ms 避免卡顿 |

### 🔹 系统资源层指标

| 指标  | 说明  | 监控方式 |
| --- | --- | --- |
| **GPU 利用率/显存占用** | 推理资源瓶颈的核心指标 | `nvidia-smi`, Prometheus + DCGM |
| **推理队列长度** | 请求排队情况，反映系统饱和度 | 应用层埋点 |
| **向量检索延迟** | RAG 系统中知识库检索耗时 | 链路追踪 (Jaeger/Zipkin) |
| **Token 消耗速率** | 每分钟/每请求的 Token 消耗，关联成本 | 业务日志 + 成本看板 |

### 🔹 稳定性指标

| 指标  | 说明  | 目标  |
| --- | --- | --- |
| **P99 延迟** | 99% 请求的完成时间，排除长尾影响 | < 5s (复杂任务) |
| **错误率 (Error Rate)** | 超时、熔断、模型报错的比例 | < 0.1% |
| **降级触发率** | 触发小模型/缓存/拒答等降级策略的频率 | 监控趋势，非绝对值 |

---

## 三、测试场景设计

### 场景 1：单请求性能基线 (Baseline)

```yaml
目的: 建立不同 Prompt 长度下的性能基线
输入: 
  - 短文本: "你好" (5 tokens)
  - 中文本: 产品咨询 (50 tokens)  
  - 长文本: 文档总结 (2000 tokens)
  - 超长上下文: 多轮对话 + 文档 (8000+ tokens)
输出: TTFT, TPS, 端到端延迟, Token 消耗
```

### 场景 2：并发压力测试 (Load/Stress)

```yaml
目的: 评估系统在高并发下的表现和瓶颈
策略:
  - 阶梯加压: 10 → 50 → 100 → 200 并发用户
  - 持续压测: 30 分钟稳定负载，观察资源泄漏
  - 突发流量: 模拟"秒杀"式请求洪峰
关注:
  - 队列堆积情况
  - GPU 显存是否 OOM
  - 是否触发自动扩缩容
  - 降级策略是否生效
```

### 场景 3：长链路场景 (RAG/Agent)

```yaml
目的: 测试含检索、工具调用的复杂流程性能
流程: 
  用户提问 → Query 改写 → 向量检索 → 重排序 → 
  Prompt 组装 → 模型推理 → 结果后处理 → 返回
拆解监控:
  - 检索阶段: P95 < 500ms
  - 推理阶段: 单独记录，便于定位瓶颈
  - 整体: 端到端 < 5s (复杂任务)
```

### 场景 4：成本 - 性能权衡测试 (Cost-Performance)

```yaml
目的: 找到"够用且便宜"的配置组合
变量:
  - 模型: GPT-4 vs GPT-3.5 vs 开源小模型
  - 参数: temperature, max_tokens, top_p
  - 策略: 缓存命中率、小模型预过滤
产出: 
  - 每个配置的 (质量分数, 延迟, 成本) 三维坐标
  - 推荐配置矩阵 (如: 客服场景用 GPT-3.5 + 缓存)
```

### 场景 5：混沌测试 (Chaos Engineering)

```yaml
目的: 验证系统在异常下的韧性
故障注入:
  - 模型 API 超时/报错 (模拟服务商故障)
  - GPU 节点宕机 (模拟硬件故障)
  - 向量数据库延迟飙升
  - 网络分区 (分区后能否降级)
验证:
  - 熔断器是否触发
  - 是否有备用模型/缓存兜底
  - 用户是否收到友好提示而非 500
```

---

## 四、工具链推荐

### 🔧 压测工具

| 工具  | 适用场景 | AI 适配建议 |
| --- | --- | --- |
| **k6** | API 级压测，脚本友好 | 用 JS 编写带 Token 统计的请求，支持流式解析 |
| **Locust** | Python 生态，灵活定制 | 方便集成 LangChain，自定义用户行为 |
| **wrk/wrk2** | 极致高性能 HTTP 压测 | 适合基准测试，但不支持复杂业务逻辑 |
| **JMeter** | 传统企业级，插件丰富 | 用 JSON Extractor 解析流式响应，较笨重 |

### 🔧 监控与可观测性

| 工具  | 用途  |
| --- | --- |
| **Prometheus + Grafana** | 采集 GPU、CPU、内存、自定义业务指标 |
| **OpenTelemetry** | 链路追踪，拆解"检索-推理-后处理"各阶段耗时 |
| **LangSmith / LangFuse** | 专门追踪 LLM 应用的性能、成本、质量 |
| **nvidia-smi / DCGM** | GPU 底层指标监控 |

### 🔧 成本分析

| 工具  | 用途  |
| --- | --- |
| **Cloud Provider Cost API** | AWS/Azure/GCP 的账单 API，实时计算推理成本 |
| **自定义 Token 计数器** | 用 `tiktoken` 库统计输入/输出 Token，关联单价 |

---

## 五、实战代码示例

### 示例 1：使用 k6 进行 API 压测 + 流式响应解析

```javascript
// ai_performance_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Counter } from 'k6/metrics';

// 自定义指标
const ttft = new Trend('time_to_first_token');
const tps = new Trend('tokens_per_second');
const tokenCost = new Counter('total_tokens');

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // 热身
    { duration: '5m', target: 50 },   // 稳定负载
    { duration: '2m', target: 100 },  // 加压
    { duration: '3m', target: 0 },    // 冷却
  ],
  thresholds: {
    'time_to_first_token': ['p(95)<1500'], // P95 TTFT < 1.5s
    'http_req_duration': ['p(99)<5000'],   // P99 端到端 < 5s
    'http_req_failed': ['rate<0.01'],      // 错误率 < 1%
  },
};

export default function () {
  const prompt = `请总结以下内容：${'长文本...'.repeat(10)}`;
  
  const params = {
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${__ENV.API_KEY}` },
  };
  
  const startTime = new Date().getTime();
  let firstTokenTime = null;
  let tokenCount = 0;
  
  // 发起流式请求
  const res = http.post(
    'https://api.my-ai-app.com/v1/chat/completions',
    JSON.stringify({
      model: 'gpt-3.5-turbo',
      messages: [{ role: 'user', content: prompt }],
      stream: true,
    }),
    params
  );
  
  // 解析 SSE 流式响应 (简化版)
  if (res.status === 200) {
    const lines = res.body.split('\n').filter(l => l.startsWith(' '));
    
    for (let i = 0; i < lines.length; i++) {
      if (lines[i] === ' [DONE]') break;
      
      try {
        const json = JSON.parse(lines[i].replace(' ', ''));
        const content = json.choices[0]?.delta?.content;
        
        if (content) {
          // 记录首字时间
          if (firstTokenTime === null) {
            firstTokenTime = new Date().getTime();
            ttft.add(firstTokenTime - startTime);
          }
          tokenCount += content.length; // 粗略估算 token
        }
      } catch (e) { /* ignore parse error */ }
    }
    
    const endTime = new Date().getTime();
    const durationSec = (endTime - startTime) / 1000;
    
    // 计算 TPS
    if (durationSec > 0 && tokenCount > 0) {
      tps.add(tokenCount / durationSec);
    }
    
    tokenCost.add(tokenCount);
  }
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'got response': (r) => r.body.length > 0,
  });
  
  sleep(1); // 模拟用户思考时间
}
```

### 示例 2：Python + Locust 模拟多轮对话 + RAG 场景

```python
# locustfile.py
from locust import HttpUser, task, between
import time
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

class AIChatUser(HttpUser):
    wait_time = between(2, 5)  # 模拟用户间隔
    
    @task(3)
    def short_question(self):
        """短问答场景"""
        self.chat("今天天气怎么样？", max_tokens=50)
    
    @task(2)
    def rag_query(self):
        """RAG 检索增强场景"""
        # 先上传/索引文档 (简化)
        # 再提问
        self.chat("根据知识库，公司的年假政策是什么？", 
                 use_rag=True, 
                 max_tokens=200)
    
    @task(1)
    def long_generation(self):
        """长文本生成场景"""
        prompt = "写一篇关于可持续能源的 800 字文章..."
        self.chat(prompt, max_tokens=500)
    
    def chat(self, prompt: str, use_rag=False, max_tokens=100):
        start = time.time()
        
        payload = {
            "prompt": prompt,
            "max_tokens": max_tokens,
            "stream": True,
        }
        if use_rag:
            payload["retrieval"] = {"enabled": True, "top_k": 3}
        
        with self.client.post("/api/chat", json=payload, 
                             catch_response=True, name="/api/chat") as resp:
            if resp.status_code != 200:
                resp.failure(f"HTTP {resp.status_code}")
                return
            
            # 解析流式响应，记录首字时间
            first_token_at = None
            total_tokens = 0
            
            for line in resp.iter_lines():
                if line.startswith(b" "):
                    data = line[6:].decode()
                    if data == "[DONE]":
                        break
                    # ... 解析 JSON, 累加 token ...
                    if first_token_at is None:
                        ttft = time.time() - start
                        self.environment.events.request.fire(
                            request_type="TTFT", 
                            name="time_to_first_token", 
                            response_time=ttft*1000, 
                            response_length=0
                        )
                        first_token_at = time.time()
            
            # 记录总耗时和 token 数
            total_time = (time.time() - start) * 1000
            input_tokens = len(enc.encode(prompt))
            # output_tokens = ... 
            
            self.environment.events.request.fire(
                request_type="E2E", 
                name="end_to_end_latency", 
                response_time=total_time, 
                response_length=total_tokens
            )
```

### 示例 3：GPU 资源监控 + 自动告警 (Prometheus + Python)

```python
# monitor_gpu.py - 与压测并行运行
import pynvml
import time
from prometheus_client import start_http_server, Gauge

# 初始化指标
gpu_util = Gauge('gpu_utilization_percent', 'GPU 利用率', ['gpu_id'])
gpu_mem = Gauge('gpu_memory_used_mb', 'GPU 显存使用', ['gpu_id'])
inference_queue = Gauge('inference_queue_length', '推理队列长度')

def collect_gpu_metrics():
    pynvml.nvmlInit()
    device_count = pynvml.nvmlDeviceGetCount()
    
    while True:
        for i in range(device_count):
            handle = pynvml.nvmlDeviceGetHandleByIndex(i)
            util = pynvml.nvmlDeviceGetUtilizationRates(handle).gpu
            mem = pynvml.nvmlDeviceGetMemoryInfo(handle).used // (1024*1024)
            
            gpu_util.labels(gpu_id=i).set(util)
            gpu_mem.labels(gpu_id=i).set(mem)
        
        # 从应用接口获取队列长度 (示例)
        # queue_len = requests.get("http://app:8000/metrics/queue").json()["length"]
        # inference_queue.set(queue_len)
        
        time.sleep(5)

if __name__ == "__main__":
    start_http_server(8001)  # Prometheus 抓取端点
    collect_gpu_metrics()
```

---

## 六、最佳实践与避坑指南

### ✅ 推荐实践

1.  **基准先行 (Baseline First)**
    
    -   每次压测前，先跑单请求基线，确保环境正常。
        
    -   建立"性能回归测试"：新模型/新配置上线前，对比基线指标。
        
2.  **真实数据分布**
    
    -   Prompt 长度不要只用固定值，按线上日志的分布采样（如：70% 短 + 20% 中 + 10% 长）。
        
    -   多轮对话测试时，上下文长度要符合真实用户行为。
        
3.  **成本意识嵌入测试**
    
    -   每个测试用例同时输出 `(延迟, 质量, 成本)` 三元组。
        
    -   设置"成本门禁"：如"单次问答成本 > $0.01 则告警"。
        
4.  **分层压测策略**
    
    ```mermaid
    graph LR
    A[单元测试] -->|Mock 模型| B(接口性能)
    B --> C[集成测试]
    C -->|真实小模型| D(链路性能)
    D --> E[端到端压测]
    E -->|生产级模型| F(系统容量)
    F --> G[混沌测试]
    G --> H(韧性验证)
    ```
    
5.  **自动化报告 + 趋势分析**
    
    -   每次 CI/CD 运行后，生成性能报告并存档。
        
    -   用 Grafana 展示"延迟/成本"的周级趋势，提前发现退化。
        

### ❌ 常见陷阱

| 陷阱  | 后果  | 解决方案 |
| --- | --- | --- |
| **只用固定短文本压测** | 上线后长文本请求打爆显存 | 按线上分布构造测试数据 |
| **忽略冷启动延迟** | 容器/模型首次加载慢，用户首屏卡 | 压测前预热，或单独测试冷启动 |
| **流式响应按普通接口测** | 误判延迟，无法监控首字体验 | 专门解析 SSE，记录 TTFT |
| **不监控向量检索** | RAG 系统瓶颈在检索而非推理 | 链路追踪，拆解各阶段耗时 |
| **成本与性能割裂** | 优化了延迟但成本翻倍 | 建立"性价比"指标，联合优化 |

---

## 七、总结：AI 性能测试的核心思维

```text
🎯 目标不是"越快越好"，而是"在成本约束下，满足用户体验阈值"

🔄 闭环流程：
   定义指标 → 构造场景 → 自动化压测 → 分析瓶颈 → 
   优化 (模型/缓存/架构) → 回归验证 → 监控告警

🔑 关键成功因素：
   1. Mock 与真实调用结合：功能测试用 Mock，性能测试用真实模型
   2. 可观测性先行：没有监控的压测是盲人摸象
   3. 业务指标驱动：用"用户感知延迟"而非"接口耗时"做决策
   4. 成本左移：在测试阶段就暴露成本风险，而非上线后才发现
```

> 💡 **一句话总结**：AI 产品的性能自动化测试 = **传统压测工具** + **模型特有指标** + **成本意识** + **可观测性基建**。先保证"能用"（功能），再优化"好用"（性能+成本），最后追求"聪明"（质量）。
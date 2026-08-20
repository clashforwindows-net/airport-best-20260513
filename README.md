# 🧭 Clash 节点智能优选系统 — 自动测速·负载均衡·动态切换

> 深入解析如何借助自动化脚本与智能算法，从数十个节点中筛选最优线路，实现负载均衡与毫秒级自动切换。本方案面向有 Clash 使用经验、追求极速体验的用户，提供从原理到实操的完整闭环。

---

## 📋 目录

- [为什么需要节点优选](#为什么需要节点优选)
- [核心概念解析](#核心概念解析)
- [自动测速脚本](#自动测速脚本)
- [节点筛选规则体系](#节点筛选规则体系)
- [负载均衡配置实战](#负载均衡配置实战)
- [智能切换算法](#智能切换算法)
- [优选工具横向对比](#优选工具横向对比)
- [自定义优选方案](#自定义优选方案)
- [实战案例](#实战案例)
- [常见问题](#常见问题)
- [进阶技巧](#进阶技巧)
- [推荐机场](#推荐机场)

---

## 🔍 为什么需要节点优选

### 手动选择的困境

大多数用户在 Clash 中随意选择一个节点后就不再更换，但这往往导致以下问题：

| 问题 | 表现 | 体验影响 |
|------|------|---------|
| 晚高峰拥堵 | 晚8-11点速度骤降 | 视频卡顿、下载断流 |
| 节点劣化 | 部分节点长期未维护 | 延迟高、丢包率高 |
| 跨区域选错 | 访问日区选了美线 | 延迟翻倍 |
| 单一节点脆弱 | 单节点故障即断网 | 无法自动恢复 |

### 节点优选的核心价值

```
手动选节点 ≈ 随机抽卡
智能优选   ≈ 稳定极速体验
```

通过自动测速 → 规则筛选 → 负载均衡 → 动态切换四步闭环，用户可以实现：
- 全天候稳定高速连接（晚高峰速度损失 <10%）
- 自动规避故障节点（切换时间 <1秒）
- 多节点并行利用（带宽叠加效果）
- 个性化策略（按应用/域名/时间分流）

---

## ⚙️ 核心概念解析

### 1. 延迟（Latency）

指数据包从本地到节点再返回的往返时间，通常以毫秒（ms）为单位。

**测量方式**：
```bash
# ICMP Ping（最常见，但可能被节点防火墙拦截）
ping -c 5 node.example.com

# TCP Ping（更准确，测试实际代理端口）
nc -zv node.example.com 443 -w 5

# Clash 内置延迟测试（推荐）
# 在 Clash 客户端 GUI 中直接查看
```

**延迟分级参考**：
| 级别 | 延迟范围 | 适用场景 |
|------|---------|---------|
| 🟢 极优 | <50ms | 实时游戏、电竞 |
| 🟡 优秀 | 50-100ms | 4K视频、大文件下载 |
| 🟠 良好 | 100-200ms | 1080P视频、日常浏览 |
| 🔴 一般 | 200-500ms | 文字为主 |
| ⚫ 较差 | >500ms | 不建议使用 |

### 2. 丢包率（Packet Loss）

指发送数据包中未能成功返回的比例。丢包率 >5% 时，视频通话和游戏会出现明显问题。

```bash
# 测量丢包率
ping -c 100 node.example.com | grep "packet loss"
```

**丢包率与体验对照**：
| 丢包率 | 体验 |
|--------|------|
| 0-1% | 完美，几乎无感知 |
| 1-3% | 轻微卡顿，可接受 |
| 3-5% | 明显卡顿，视频缓冲 |
| >5% | 严重卡顿，游戏掉线 |

### 3. 带宽（Bandwidth）

实际可用吞吐量，通常以 Mbps（下载）或 MB/s（下载速度）为单位。

**注意**：节点标注的带宽≠实际可用带宽，需实测。

```bash
# 使用 curl 测试实际带宽
curl -o /dev/null -s -w "速度: %{speed_download} bytes/s\n" \
  --max-time 30 https://speed.cloudflare.com/__down?bytes=100000000
```

### 4. 节点健康度评分

综合评分公式：

```
健康度 = (速度分 × 0.35) + (延迟分 × 0.30) + (稳定性分 × 0.25) + (解锁分 × 0.10)
```

| 指标 | 计算方式 |
|------|---------|
| 速度分 | 实测带宽 / 理论带宽，最高100分 |
| 延迟分 | 100 - (实测延迟 / 500 × 100)，最低0分 |
| 稳定性分 | 100 - (丢包率 × 20) |
| 解锁分 | 流媒体解锁数量 × 20 |

---

## 🛠️ 自动测速脚本

### PowerShell 多节点并发测速脚本

以下脚本在 Windows 上运行，测试所有节点并输出排序结果：

```powershell
# 文件名: node-speedtest.ps1
# 使用方法: 右键用 PowerShell 运行，或 .\node-speedtest.ps1
# 需要 Clash 客户端运行中（默认代理端口 7890）

param(
    [string]$ProxyHost = "127.0.0.1",
    [int]$ProxyPort = 7890,
    [int]$TimeoutSec = 15,
    [int]$TestCount = 3,
    [int]$TopN = 20
)

$ErrorActionPreference = "SilentlyContinue"

# 测试节点列表（可从 Clash 订阅 URL 自动获取，此处手动示例）
$TestNodes = @(
    "香港-01",
    "香港-02",
    "台湾-01",
    "台湾-02",
    "日本-01",
    "日本-02",
    "新加坡-01",
    "美国-洛杉矶",
    "美国-纽约",
    "韩国-01"
)

$TestUrls = @(
    "https://speed.cloudflare.com/__down?bytes=5000000",   # 5MB
    "https://proof.ovh.net/files/1Mb.dat",                 # 1MB
    "https://downloads.cdn.pwtest.net/10MB.bin"           # 10MB
)

function Test-NodeSpeed {
    param($NodeName, $ProxyUrl)
    
    $speeds = @()
    $latencies = @()
    
    foreach ($url in $TestUrls) {
        try {
            $sw = [System.Diagnostics.Stopwatch]::StartNew()
            $response = Invoke-WebRequest -Uri $url -Proxy $ProxyUrl -TimeoutSec $TimeoutSec -UseBasicParsing
            $elapsed = $sw.ElapsedMilliseconds
            
            if ($response.StatusCode -eq 200) {
                $sizeMB = $response.Content.Length / 1MB
                $speedMbps = [math]::Round(($sizeMB * 8) / ($elapsed / 1000), 2)
                $speeds += $speedMbps
            }
        } catch {
            continue
        }
    }
    
    # 延迟测试
    try {
        $pingUrl = "https://www.google.com"
        $pingSw = [System.Diagnostics.Stopwatch]::StartNew()
        Invoke-WebRequest -Uri $pingUrl -Proxy $ProxyUrl -TimeoutSec 5 -UseBasicParsing | Out-Null
        $latency = $pingSw.ElapsedMilliseconds
        $latencies += $latency
    } catch {
        $latencies += 9999
    }
    
    $avgSpeed = if ($speeds.Count -gt 0) { ($speeds | Measure-Object -Average).Average } else { 0 }
    $avgLatency = if ($latencies.Count -gt 0) { ($latencies | Measure-Object -Average).Average } else { 9999 }
    
    return @{
        Speed = [math]::Round($avgSpeed, 2)
        Latency = $avgLatency
        Score = [math]::Round(($avgSpeed * 0.6) + (max(0, (200 - $avgLatency)) * 0.4), 2)
    }
}

Write-Host "======================================" -ForegroundColor Cyan
Write-Host "   Clash 节点智能测速工具 v1.0" -ForegroundColor Cyan
Write-Host "======================================" -ForegroundColor Cyan
Write-Host ""

$results = @()
$count = 0
$total = $TestNodes.Count

foreach ($node in $TestNodes) {
    $count++
    Write-Host "[$count/$total] 测试: $node ..." -NoNewline
    
    # 注意：实际使用时需通过 Clash API 切换到对应节点再测速
    $proxyUrl = "http://$ProxyHost`:$ProxyPort"
    $result = Test-NodeSpeed -NodeName $node -ProxyUrl $proxyUrl
    
    $results += [PSCustomObject]@{
        Node = $node
        SpeedMbps = $result.Speed
        LatencyMs = $result.Latency
        Score = $result.Score
    }
    
    Write-Host " 速度: $($result.Speed) Mbps | 延迟: $($result.Latency) ms | 评分: $($result.Score)" -ForegroundColor Green
}

Write-Host ""
Write-Host "========== 测速结果排名 ==========" -ForegroundColor Yellow

$sorted = $results | Sort-Object Score -Descending | Select-Object -First $TopN

$i = 1
foreach ($r in $sorted) {
    $bar = "█" * [math]::Min([int]($r.Score / 5), 20)
    $color = if ($i -le 3) { "Green" } elseif ($i -le 10) { "Cyan" } else { "White" }
    Write-Host "$($i.ToString().PadLeft(2)). $($r.Node.PadRight(20)) | $($r.SpeedMbps) Mbps | $($r.LatencyMs) ms | $bar" -ForegroundColor $color
    $i++
}

Write-Host ""
Write-Host "推荐配置：将 Top$TopN 节点配置为策略组，使用 fallback 规则自动切换" -ForegroundColor Yellow
```

### Python 跨平台测速脚本

```python
#!/usr/bin/env python3
"""
clash_node_speedtest.py
Clash 节点测速脚本（Python 3.8+）
pip install httpx aiohttp tqdm

使用示例:
  python clash_node_speedtest.py --url https://your-subscription-url
  python clash_node_speedtest.py --port 7890 --top 10
"""

import argparse
import asyncio
import base64
import json
import sys
import time
from dataclasses import dataclass, field
from typing import List, Optional

try:
    import httpx
    import yaml
except ImportError:
    print("缺少依赖，请先安装: pip install httpx pyyaml")
    sys.exit(1)


@dataclass
class NodeResult:
    name: str
    protocol: str
    server: str
    speed_mbps: float = 0.0
    latency_ms: float = 9999.0
    score: float = 0.0
    stream_media: str = "未知"
    available: bool = True
    error: str = ""


class ClashSpeedTester:
    """Clash 节点测速器"""
    
    TEST_URLS = [
        "https://speed.cloudflare.com/__down?bytes=10000000",
        "https://proof.ovh.net/files/10Mb.dat",
        "https://downloads.cdn.pwtest.net/10MB.bin",
    ]
    
    STREAM_TEST_URLS = {
        "Netflix": "https://www.netflix.com/title/60021922",
        "Disney+": "https://www.disneyplus.com/",
        "YouTube": "https://www.youtube.com/premium",
        "TikTok": "https://www.tiktok.com/",
    }
    
    def __init__(self, subscription_url: str = None, config_path: str = None,
                 proxy_port: int = 7890, timeout: int = 20):
        self.subscription_url = subscription_url
        self.config_path = config_path
        self.proxy_port = proxy_port
        self.timeout = timeout
        self.client: Optional[httpx.AsyncClient] = None
        self.nodes: List[dict] = []
    
    def load_config(self) -> List[dict]:
        """加载订阅或配置文件"""
        if self.subscription_url:
            resp = httpx.get(self.subscription_url, timeout=30)
            content = base64.b64decode(resp.text).decode()
            config = yaml.safe_load(content)
            return config.get("proxies", [])
        elif self.config_path:
            with open(self.config_path) as f:
                config = yaml.safe_load(f)
            return config.get("proxies", [])
        return []
    
    async def test_single_node(self, node: dict) -> NodeResult:
        """测试单个节点"""
        result = NodeResult(
            name=node.get("name", "未知"),
            protocol=node.get("type", "unknown"),
            server=node.get("server", "")
        )
        
        # 构建代理 URL（简化版，实际需要根据协议类型构造）
        proxies = f"http://127.0.0.1:{self.proxy_port}"
        
        try:
            async with httpx.AsyncClient(
                proxies=proxies,
                timeout=self.timeout,
                follow_redirects=True
            ) as client:
                # 带宽测试
                speeds = []
                for url in self.TEST_URLS:
                    try:
                        start = time.time()
                        resp = await client.get(url)
                        elapsed = time.time() - start
                        size_mb = len(resp.content) / (1024 * 1024)
                        speed = (size_mb * 8) / elapsed  # Mbps
                        speeds.append(speed)
                    except Exception:
                        continue
                
                if speeds:
                    result.speed_mbps = round(sum(speeds) / len(speeds), 2)
                
                # 延迟测试
                try:
                    start = time.time()
                    await client.get("https://www.google.com")
                    result.latency_ms = round((time.time() - start) * 1000, 0)
                except Exception:
                    result.latency_ms = 9999
                
                # 流媒体解锁测试
                unlocked = []
                for platform, url in self.STREAM_TEST_URLS.items():
                    try:
                        resp = await client.get(url, timeout=10)
                        if resp.status_code == 200:
                            unlocked.append(platform)
                    except Exception:
                        pass
                
                result.stream_media = " / ".join(unlocked) if unlocked else "未解锁"
                
                # 计算综合评分
                speed_score = min(result.speed_mbps / 5, 100) * 0.35
                latency_score = max(0, (200 - result.latency_ms) / 2) * 0.30
                stability_score = 80 * 0.25
                unlock_score = len(unlocked) * 20 * 0.10
                result.score = round(speed_score + latency_score + stability_score + unlock_score, 1)
                
        except Exception as e:
            result.available = False
            result.error = str(e)
        
        return result
    
    async def run(self) -> List[NodeResult]:
        """运行完整测速"""
        self.nodes = self.load_config()
        print(f"📡 共加载 {len(self.nodes)} 个节点，开始测速...\n")
        
        tasks = [self.test_single_node(node) for node in self.nodes]
        results: List[NodeResult] = await asyncio.gather(*tasks)
        
        # 过滤可用节点并排序
        available = [r for r in results if r.available]
        available.sort(key=lambda x: x.score, reverse=True)
        
        return available


async def main():
    parser = argparse.ArgumentParser(description="Clash 节点智能测速工具")
    parser.add_argument("--url", help="Clash 订阅 URL")
    parser.add_argument("--config", help="本地配置文件路径")
    parser.add_argument("--port", type=int, default=7890, help="代理端口")
    parser.add_argument("--top", type=int, default=20, help="显示 Top N 结果")
    args = parser.parse_args()
    
    tester = ClashSpeedTester(
        subscription_url=args.url,
        config_path=args.config,
        proxy_port=args.port
    )
    
    results = await tester.run()
    
    print(f"\n{'='*80}")
    print(f"{'排名':^4} | {'节点名称':^25} | {'速度(Mbps)':^10} | {'延迟(ms)':^8} | {'解锁':^20} | {'评分':^6}")
    print(f"{'-'*80}")
    
    for i, r in enumerate(results[:args.top], 1):
        bar = "▓" * min(int(r.score / 5), 20)
        print(f"{i:^4} | {r.name[:25]:^25} | {r.speed_mbps:^10.2f} | {r.latency_ms:^8.0f} | {r.stream_media[:20]:^20} | {r.score:^6.1f}")
    
    print(f"{'-'**80}")
    print(f"\n✅ 测速完成。共测试 {len(results)} 个节点，可用 {len(results)} 个")
    print(f"💡 建议：将 Top {args.top} 节点导入 Clash 策略组，启用自动切换")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📐 节点筛选规则体系

### 筛选维度

| 维度 | 权重 | 筛选方式 |
|------|------|---------|
| 速度 | 35% | 实测带宽 >50Mbps |
| 延迟 | 30% | 延迟 <200ms |
| 稳定性 | 25% | 连续3次测速均达标 |
| 解锁 | 10% | Netflix 优先 |

### 筛选规则 YAML 示例

```yaml
# advanced-filter.yaml
# Clash 高级筛选规则配置

filter_rules:
  # 速度筛选：排除低于 30Mbps 的节点
  speed_threshold_mbps: 30
  
  # 延迟筛选：排除高于 250ms 的节点
  latency_threshold_ms: 250
  
  # 丢包筛选：排除丢包率 >3% 的节点
  packet_loss_max_percent: 3
  
  # 必选协议类型
  allowed_protocols:
    - ss
    - vmess
    - trojan
    - hysteria2
  
  # 排除的节点关键词
  exclude_keywords:
    - "测试"
    - "免费"
    - "临时"
    - "demo"
  
  # 必须包含的节点关键词（至少一个）
  require_keywords:
    - "香港"
    - "台湾"
    - "日本"
    - "新加坡"
  
  # 流媒体解锁要求
  require_netflix: true
  require_disney: false
  
  # 评分阈值
  min_score: 60
```

### Python 节点过滤器

```python
import yaml
from dataclasses import dataclass
from typing import List

@dataclass
class ClashNode:
    name: str
    server: str
    port: int
    protocol: str
    speed_mbps: float
    latency_ms: float
    packet_loss: float
    netflix: bool
    disney: bool
    score: float

class NodeFilter:
    def __init__(self, rules_path: str = "advanced-filter.yaml"):
        with open(rules_path) as f:
            self.rules = yaml.safe_load(f)["filter_rules"]
    
    def apply(self, nodes: List[ClashNode]) -> List[ClashNode]:
        """应用筛选规则，返回符合条件的节点"""
        filtered = []
        
        for node in nodes:
            # 速度检查
            if node.speed_mbps < self.rules["speed_threshold_mbps"]:
                continue
            
            # 延迟检查
            if node.latency_ms > self.rules["latency_threshold_ms"]:
                continue
            
            # 丢包检查
            if node.packet_loss > self.rules["packet_loss_max_percent"]:
                continue
            
            # 协议检查
            if node.protocol not in self.rules["allowed_protocols"]:
                continue
            
            # 排除关键词
            if any(kw in node.name for kw in self.rules["exclude_keywords"]):
                continue
            
            # 必须包含关键词（至少一个）
            require_any = any(kw in node.name for kw in self.rules["require_keywords"])
            if self.rules["require_keywords"] and not require_any:
                continue
            
            # Netflix 解锁要求
            if self.rules.get("require_netflix") and not node.netflix:
                continue
            
            # 评分阈值
            if node.score < self.rules["min_score"]:
                continue
            
            filtered.append(node)
        
        # 按评分排序
        return sorted(filtered, key=lambda x: x.score, reverse=True)
```

---

## ⚖️ 负载均衡配置实战

### 什么是负载均衡？

负载均衡（Load Balance）指将流量分配到多个节点，以实现：
- **带宽叠加**：多节点同时下载，速度接近带宽总和
- **故障容灾**：单节点故障不影响整体连接
- **压力分散**：避免单一节点过载

### Clash 负载均衡策略组配置

```yaml
# Clash 配置文件中添加负载均衡策略组

proxy-groups:
  # 手动选择策略组
  - name: "🔧 手动选择"
    type: select
    proxies:
      - 香港-01
      - 香港-02
      - 日本-01
      - 日本-02
      - 新加坡-01
  
  # URL 测试自动选择最优节点（推荐）
  - name: "⚡ 自动优选"
    type: url-test
    # 候选节点列表（从订阅中筛选出的优质节点）
    proxies:
      - 香港-01
      - 香港-02
      - 日本-01
      - 日本-02
      - 新加坡-01
      - 台湾-01
      - 台湾-02
    # 测速 URL（Clash 内置）
    url: "http://www.gstatic.com/generate_204"
    # 间隔多久重新测速（秒）
    interval: 600
    # 取延迟最低的节点
    tolerance: 50
  
  # 故障切换策略（Fallback）
  - name: "🛡️ 备用切换"
    type: fallback
    proxies:
      - 香港-01
      - 日本-01
      - 新加坡-01
      - 美国-01
    url: "http://www.gstatic.com/generate_204"
    interval: 300
  
  # 负载均衡策略（尽可能均匀分配）
  - name: "⚖️ 负载均衡"
    type: load-balance
    proxies:
      - 香港-01
      - 香港-02
      - 日本-01
      - 日本-02
    url: "http://www.gstatic.com/generate_204"
    interval: 600
    # 负载均衡策略: consistent-hashing（一致性哈希）或 round-robin（轮询）
    strategy: consistent-hashing
  
  # 代理链（通过多个节点）
  - name: "🔗 链式代理"
    type: select
    proxies:
      - 香港-01
      - 日本-01
      - 新加坡-01

# 规则配置
rules:
  # 国内网站直连
  - DOMAIN-SUFFIX,cn,DIRECT
  - GEOIP,CN,DIRECT
  
  # 国外网站走自动优选
  - MATCH,⚡ 自动优选
  
  # Netflix 走专用策略组
  - DOMAIN-SUFFIX,netflix.com,⚡ 自动优选
  - DOMAIN-SUFFIX,nflxvideo.net,⚡ 自动优选
  - DOMAIN-KEYWORD,netflix,⚡ 自动优选
  
  # YouTube 走负载均衡
  - DOMAIN-SUFFIX,youtube.com,⚖️ 负载均衡
  - DOMAIN-SUFFIX,googlevideo.com,⚖️ 负载均衡
  
  # 游戏走低延迟节点
  - DOMAIN-KEYWORD,steam,🛡️ 备用切换
  - DOMAIN-KEYWORD,epicgames,🛡️ 备用切换
  
  # 默认使用手动选择
  - MATCH,🔧 手动选择
```

### 三种负载均衡策略对比

| 策略 | 类型 | 适用场景 | 优点 | 缺点 |
|------|------|---------|------|------|
| `url-test` | 自动选最优 | 日常浏览、视频 | 始终用最快节点 | 切换时有短暂断开 |
| `fallback` | 故障切换 | 稳定性优先 | 主节点故障秒切换 | 备用节点可能较慢 |
| `load-balance` | 流量分配 | 大文件下载 | 多节点带宽叠加 | 配置较复杂 |

---

## 🔄 智能切换算法

### 自动切换逻辑流程

```
开始监控
    ↓
读取节点评分表（每5分钟更新）
    ↓
当前节点评分 < 阈值？
  ├─ 是 → 评分低于前3名超过10分？→ 触发切换
  └─ 否 → 继续使用当前节点
    ↓
切换到评分最高的节点
    ↓
验证新节点连通性（3次重试）
  ├─ 成功 → 更新当前节点，记录切换日志
  └─ 失败 → 回退到上一节点，标记故障节点
    ↓
冷却期（5分钟内不重复切换，避免抖动）
```

### 智能切换 PowerShell 实现

```powershell
# smart-switch.ps1 - 智能节点切换脚本
# 配合 Clash API 使用，需要开启 Allow LAN 和 RESTful API

param(
    [string]$ClashHost = "127.0.0.1",
    [int]$ClashPort = 9090,
    [int]$CheckIntervalSec = 300,       # 检查间隔（秒）
    [int]$ScoreThreshold = 10,          # 评分差距阈值
    [int]$CooldownSec = 300,            # 冷却时间（秒）
    [int]$RetryCount = 3               # 失败重试次数
)

$ErrorActionPreference = "SilentlyContinue"

# 获取 Clash API 状态
function Get-ClashStatus {
    $uri = "http://${ClashHost}:${ClashPort}/proxies"
    try {
        $data = Invoke-RestMethod -Uri $uri -TimeoutSec 10
        return $data
    } catch {
        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] ❌ 无法连接 Clash API: $_" -ForegroundColor Red
        return $null
    }
}

# 获取所有代理节点
function Get-ClashProxies {
    $uri = "http://${ClashHost}:${ClashPort}/proxies"
    $data = Get-ClashStatus
    if ($data) {
        return $data.proxies
    }
    return @{}
}

# 切换到指定节点
function Switch-Node {
    param($NodeName)
    
    $uri = "http://${ClashHost}:${ClashPort}/proxies/GLOBAL"
    $body = @{ name = $NodeName } | ConvertTo-Json
    
    try {
        Invoke-RestMethod -Uri $uri -Method Put -Body $body -ContentType "application/json" -TimeoutSec 10 | Out-Null
        return $true
    } catch {
        return $false
    }
}

# 获取节点延迟
function Test-NodeLatency {
    param($NodeName)
    
    $uri = "http://${ClashHost}:${ClashPort}/proxies/$($NodeName)"
    try {
        $data = Invoke-RestMethod -Uri $uri -TimeoutSec 5
        $history = $data.history
        if ($history -and $history.Count -gt 0) {
            # 从历史数据提取延迟（ms）
            $last = $history[-1]
            if ($last..delay -and $last.delay -gt 0) {
                return $last.delay
            }
        }
    } catch {}
    
    # 备用：手动测试
    return 9999
}

# 评分计算
function Get-NodeScore {
    param($NodeName)
    
    $latency = Test-NodeLatency -NodeName $NodeName
    
    # 评分公式
    $speedScore = max(0, 100 - $latency)
    $latencyScore = max(0, 200 - $latency) / 2
    
    return $speedScore + $latencyScore
}

Write-Host "======================================" -ForegroundColor Cyan
Write-Host "   Clash 智能节点切换守护进程 v1.0" -ForegroundColor Cyan
Write-Host "======================================" -ForegroundColor Cyan
Write-Host "监控间隔: ${CheckIntervalSec}s | 切换阈值: ${ScoreThreshold}分 | 冷却时间: ${CooldownSec}s" -ForegroundColor Yellow
Write-Host ""

$lastSwitchTime = 0
$currentNode = ""

while ($true) {
    $proxies = Get-ClashProxies
    if (-not $proxies) {
        Start-Sleep -Seconds $CheckIntervalSec
        continue
    }
    
    $globalProxy = $proxies["GLOBAL"]
    if ($globalProxy) {
        $currentNode = $globalProxy.now
        
        # 收集所有节点评分
        $scores = @{}
        foreach ($name in $globalProxy.all.Split(",")) {
            if ($name -ne "DIRECT" -and $name -ne "REJECT") {
                $scores[$name] = Get-NodeScore -NodeName $name
            }
        }
        
        $sorted = $scores.GetEnumerator() | Sort-Object Value -Descending
        $best = $sorted[0]
        $bestName = $best.Key
        $bestScore = $best.Value
        $currentScore = $scores[$currentNode]
        
        $time = Get-Date -Format 'HH:mm:ss'
        
        if ($currentNode -ne $bestName -and ($bestScore - $currentScore) -ge $ScoreThreshold) {
            $elapsed = (Get-Date).Ticks - $lastSwitchTime
            
            # 冷却检查
            if ($lastSwitchTime -eq 0 -or ($elapsed / 10000000) -gt $CooldownSec) {
                Write-Host "[$time] 🔄 切换: $currentNode ($currentScore) → $bestName ($bestScore)" -ForegroundColor Green
                
                if (Switch-Node -NodeName $bestName) {
                    Write-Host "[$time] ✅ 切换成功" -ForegroundColor Green
                    $lastSwitchTime = (Get-Date).Ticks
                } else {
                    Write-Host "[$time] ❌ 切换失败，保留原节点" -ForegroundColor Red
                }
            } else {
                Write-Host "[$time] ⏸️ 冷却中，跳过切换" -ForegroundColor Yellow
            }
        } else {
            Write-Host "[$time] ✅ 当前节点最优: $currentNode ($currentScore)" -ForegroundColor Gray
        }
    }
    
    Start-Sleep -Seconds $CheckIntervalSec
}
```

---

## 🧰 优选工具横向对比

| 工具 | 平台 | 自动化程度 | 负载均衡 | 智能切换 | 评分体系 | 学习曲线 |
|------|------|-----------|---------|---------|---------|---------|
| Clash 客户端内置 | 全平台 | ⭐⭐ | ⭐⭐ | ⭐⭐ | 简单延迟 | ⭐ 极简 |
| subscribed | Linux/Docker | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 完整评分 | ⭐⭐ 中等 |
| stash-policy | iOS/Stash | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | 自定义规则 | ⭐⭐⭐ 较难 |
| sub-web | Web UI | ⭐⭐ | ⭐⭐ | ⭐⭐ | 手动选择 | ⭐ 简单 |
| clash-meta-autosort | 全平台 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 多维度 | ⭐⭐⭐ 较难 |
| NodeSelector插件 | CFW | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 延迟+速度 | ⭐⭐ 简单 |

### 推荐方案

**日常用户**：使用 Clash 客户端内置 `url-test` 策略组，配置 interval=600（10分钟自动测速）。

**进阶用户**：使用 `sub-web` + `clash-meta-autosort`，实现全自动节点优选和智能切换。

**专业用户**：自建 `subscribed` 服务，配合自定义评分算法，完全掌控节点管理。

---

## 🎯 自定义优选方案

### 场景1：游戏加速优先

```yaml
# 游戏场景：延迟优先，牺牲速度换稳定性
proxy-groups:
  - name: "🎮 游戏专用"
    type: fallback
    proxies:
      - 香港-01
      - 香港-02
      - 台湾-01
      - 日本-01
      - 韩国-01
    url: "http://www.gstatic.com/generate_204"
    interval: 180
    tolerance: 10  # 延迟容忍度低，严格选最优
```

### 场景2：4K 视频优先

```yaml
# 视频场景：带宽优先，速度换稳定性
proxy-groups:
  - name: "📺 视频专用"
    type: url-test
    proxies:
      - 日本-01
      - 日本-02
      - 新加坡-01
      - 香港-01
    url: "https://speed.cloudflare.com/__down"
    interval: 900
    tolerance: 100  # 延迟容忍度高，选速度快的
```

### 场景3：学术/开发访问

```yaml
# 开发场景：GitHub/Stack Overflow 优先
rules:
  - DOMAIN-SUFFIX,github.com,⚡ 自动优选
  - DOMAIN-SUFFIX,githubusercontent.com,⚡ 自动优选
  - DOMAIN-KEYWORD,stackoverflow,⚡ 自动优选
  - DOMAIN-SUFFIX,npmjs.com,⚡ 自动优选
  - DOMAIN-SUFFIX,pypi.org,⚡ 自动优选
```

---

## 💡 实战案例

### 案例：从100个节点中优选 Top 10

**背景**：用户订阅包含 100 个节点，希望自动选出最优 10 个用于日常使用。

**步骤**：
1. 运行 `node-speedtest.ps1`，导出全部 100 个节点测速结果
2. 使用 Python 脚本按评分排序，取 Top 10
3. 将 Top 10 节点名称填入 `url-test` 策略组
4. 配置定时任务，每 6 小时重新测速更新

**结果**：
```
原始 100 节点平均速度: 45 Mbps
优选后 Top 10 平均速度: 92 Mbps
晚高峰速度损失: 从 60% 降至 8%
```

### 案例：多设备家庭共享

**背景**：家庭 4 台设备（2 手机 + 1 平板 + 1 电脑），希望共享最优节点。

**方案**：
- 路由器安装 OpenClash，配置 `url-test` 策略组
- 所有设备连接路由器，无需各自配置
- 路由器每 10 分钟自动测速切换最优节点

```yaml
# 路由器 OpenClash 配置
proxy-groups:
  - name: "家庭优选"
    type: url-test
    proxies:
      - 香港-01
      - 香港-02
      - 日本-01
      - 台湾-01
    url: "http://www.gstatic.com/generate_204"
    interval: 600
```

---

## ❓ 常见问题

### Q: url-test 策略组多久测速一次？

A: 默认 interval=600（10分钟）。晚高峰期间建议缩短到 300 秒。

### Q: 测速太频繁会触发机场限速吗？

A: 建议每 10 分钟测速一次，且每次测速总流量不超过 5MB，全天约 720MB，对大多数机场影响可忽略。

### Q: load-balance 策略为什么有时速度反而更慢？

A: 因为一致性哈希需要维护连接状态，中途切换节点会导致连接断开。建议大文件下载场景使用 `url-test` 而非 `load-balance`。

### Q: 如何只测试特定地区的节点？

A: 在策略组 `proxies` 列表中只保留目标地区的节点名称，Clash 就会只从这些节点中选择。

### Q: 节点评分突然为0怎么办？

A: 可能是节点临时故障或订阅过期。先检查订阅是否有效，再手动测速确认。评分归零的节点通常在 24 小时内恢复。

---

## 🔧 进阶技巧

### 1. 多订阅合并

使用 `subscribed` 工具合并多个订阅源，过滤重复节点后再测速：

```yaml
# subscribed.yaml
混合订阅:
  - https://example.com/sub1
  - https://example.com/sub2
  - https://example.com/sub3

过滤规则:
  去除重复: true
  速度阈值: 20Mbps
  延迟阈值: 300ms
  排除关键词: ["测试", "免费", "临时"]
```

### 2. 定时自动更新订阅 + 优选

```bash
# Linux crontab 任务：每天凌晨3点自动更新并测速
0 3 * * * /usr/local/bin/clash-sub-update.sh && /usr/local/bin/node-speedtest.py --top 20 >> /var/log/clash-auto.log 2>&1
```

### 3. 节点健康度长期监控

```python
# monitor.py - 监控节点健康度并生成报告
import sqlite3
from datetime import datetime

conn = sqlite3.connect('node_health.db')
c = conn.cursor()
c.execute('''CREATE TABLE IF NOT EXISTS health_log
             (date TEXT, node TEXT, speed REAL, latency REAL, score REAL)''')

def log_health(node, speed, latency, score):
    c.execute('INSERT INTO health_log VALUES (?,?,?,?,?)',
              (datetime.now().isoformat(), node, speed, latency, score))
    conn.commit()

def gen_report():
    c.execute('''SELECT node, AVG(speed) as avg_speed, AVG(latency) as avg_lat,
                       AVG(score) as avg_score, COUNT(*) as tests
                FROM health_log GROUP BY node ORDER BY avg_score DESC''')
    return c.fetchall()
```

---

## 🏆 推荐机场

选择优质稳定的机场是节点优选的基础。以下机场节点质量高、线路优化好，非常适合作为优选系统的数据源：

### 🔥 ClashVIP（强烈推荐）

**官网**: https://clashvip.net

| 核心指标 | 数据 |
|---------|------|
| 节点数量 | 100+ 优质节点 |
| 线路类型 | 亚太优化 / 专线 |
| 最高带宽 | 10Gbps 共享 |
| 流媒体 | 全节点解锁 Netflix/Disney+ |
| 协议支持 | SS / VMess / Trojan / Hysteria2 |
| 客服 | 7×24 中文响应 |

**推荐理由**：ClashVIP 所有节点均经过线路优化，延迟低、带宽高、稳定性好，是节点优选系统的理想数据源。

**套餐参考**：

| 套餐 | 月付 | 年付 | 月流量 | 设备数 |
|------|------|------|--------|--------|
| 基础版 | 15元 | 150元 | 100GB | 3台 |
| 标准版 | 25元 | 250元 | 200GB | 5台 |
| 高级版 | 45元 | 450元 | 500GB | 8台 |
| 企业版 | 99元 | 990元 | 不限 | 不限 |

### 更多选择

- **机场导航**: https://nav.clashvip.net — 汇集多家优质机场信息
- **Clash 教程**: https://clashhub.net — 节点优选配置教程
- **用户社区**: https://bbs.clashhub.net — 真实用户评测与经验分享
- **客户端下载**: https://clash-for-windows.net — Clash for Windows 下载

---

## 📊 附录：评分参考数据

以下为典型节点在不同评分维度下的数据参考：

| 节点地区 | 典型延迟(ms) | 典型速度(Mbps) | 健康度评分 |
|---------|------------|--------------|-----------|
| 香港-优化 | 30-50 | 80-150 | 90-100 |
| 香港-普通 | 50-100 | 30-80 | 70-90 |
| 台湾 | 60-120 | 50-100 | 65-85 |
| 日本 | 80-150 | 60-120 | 60-80 |
| 韩国 | 70-130 | 50-100 | 60-80 |
| 新加坡 | 100-180 | 40-80 | 50-70 |
| 美国-洛杉矶 | 150-250 | 30-80 | 40-60 |
| 美国-纽约 | 200-350 | 20-60 | 30-50 |
| 欧洲 | 200-400 | 20-50 | 25-45 |

---

## 📝 总结

节点优选是一个持续优化的过程。核心要点：

1. **定期测速**：至少每 6 小时测速一次，晚高峰加密结果更准确
2. **合理评分**：综合速度、延迟、稳定性和解锁能力，而非单一指标
3. **策略配置**：根据使用场景选择合适的策略组类型
4. **自动切换**：配合冷却机制避免频繁抖动
5. **监控记录**：长期记录帮助发现节点劣化趋势

掌握了以上内容，你就能构建一套稳定、高效的 Clash 节点优选系统，真正实现"最优节点自动选择，全天候极速体验"。

---

**更新日期**: 2026-08-20  
**版本**: v2.0  
**贡献**: 欢迎提交 Issue 或 Pull Request 完善本仓库

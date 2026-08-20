# 2026年度 机场节点优选完全指南

> 本文深度解析节点优选工具的原理、配置与实战，覆盖自动测速、规则筛选、负载均衡、智能切换等全链路技术栈。

---

## 目录

- [一、自动测速脚本](#一自动测速脚本)
- [二、节点筛选规则](#二节点筛选规则)
- [三、负载均衡配置](#三负载均衡配置)
- [四、智能切换算法](#四智能切换算法)
- [五、优选工具对比](#五优选工具对比)
- [六、自定义优选方法](#六自定义优选方法)
- [七、综合实战方案](#七综合实战方案)
- [推广资源](#推广资源)

---

## 一、自动测速脚本

节点测速是优选的基础。手动测速效率低、误差大，自动化脚本是规模化运营的必备能力。

### 1.1 Python 多协议并发测速脚本

```python
#!/usr/bin/env python3
"""
节点多协议并发测速工具
支持 ss、ssr、vmess、trojan、hysteria2 五种协议
按延迟、下载速度、稳定性三维评分输出排名
"""

import asyncio
import aiohttp
import base64
import json
import re
import sys
import time
from dataclasses import dataclass, field
from typing import List, Optional
from urllib.parse import urlparse

@dataclass
class NodeResult:
    name: str
    protocol: str
    server: str
    port: int
    latency_ms: float = 0.0
    download_speed_mbps: float = 0.0
    stability_score: float = 0.0
    packet_loss: float = 0.0
    final_score: float = 0.0
    error: Optional[str] = None

@dataclass
class SpeedTestConfig:
    test_url: str = "https://speed.cloudflare.com/__down?bytes=10485760"
    test_count: int = 3
    timeout_seconds: float = 10.0
    latency_threshold_ms: float = 300.0
    min_speed_mbps: float = 10.0

class NodeParser:
    """节点链接解析器，支持多种格式"""

    @staticmethod
    def parse_subscription(url: str) -> List[dict]:
        """解析订阅链接，返回节点列表"""
        protocols = {
            "ss": NodeParser._parse_ss,
            "ssr": NodeParser._parse_ssr,
            "vmess": NodeParser._parse_vmess,
            "trojan": NodeParser._parse_trojan,
            "hysteria2": NodeParser._parse_hysteria2,
        }
        results = []
        # 实际生产中应使用 aiohttp 获取订阅内容并解码
        return results

    @staticmethod
    def _parse_ss(data: str) -> Optional[dict]:
        match = re.match(r"ss://([A-Za-z0-9+/=]+)@([^:]+):(\d+)/?", data)
        if match:
            return {"protocol": "ss", "server": match.group(2), "port": int(match.group(3))}
        return None

    @staticmethod
    def _parse_ssr(data: str) -> Optional[dict]:
        match = re.match(r"ssr://([A-Za-z0-9+/=]+)", data)
        if match:
            try:
                decoded = base64.b64decode(match.group(1)).decode()
                parts = decoded.split("/?")
                server_info = parts[0].split(":")
                return {"protocol": "ssr", "server": server_info[0], "port": int(server_info[1])}
            except Exception:
                return None
        return None

    @staticmethod
    def _parse_vmess(data: str) -> Optional[dict]:
        match = re.match(r"vmess://([A-Za-z0-9+/=]+)", data)
        if match:
            try:
                decoded = json.loads(base64.b64decode(match.group(1)))
                return {"protocol": "vmess", "server": decoded.get("add"), "port": int(decoded.get("port"))}
            except Exception:
                return None
        return None

    @staticmethod
    def _parse_trojan(data: str) -> Optional[dict]:
        match = re.match(r"trojan://([^@]+)@([^:]+):(\d+)/?", data)
        if match:
            return {"protocol": "trojan", "server": match.group(2), "port": int(match.group(3))}
        return None

    @staticmethod
    def _parse_hysteria2(data: str) -> Optional[dict]:
        match = re.match(r"hysteria2://([^@]+)@([^:]+):(\d+)/?", data)
        if match:
            return {"protocol": "hysteria2", "server": match.group(2), "port": int(match.group(3))}
        return None

class LatencyTester:
    """延迟测试模块"""

    @staticmethod
    async def measure_tcp_latency(host: str, port: int, timeout: float = 5.0) -> float:
        """TCP 直连测延迟（不经过代理）"""
        try:
            start = time.perf_counter()
            reader, writer = await asyncio.wait_for(
                asyncio.open_connection(host, port),
                timeout=timeout
            )
            latency = (time.perf_counter() - start) * 1000
            writer.close()
            await writer.wait_closed()
            return round(latency, 2)
        except Exception:
            return 99999.0

    @staticmethod
    async def measure_http_latency(url: str, timeout: float = 5.0) -> float:
        """HTTP HEAD 请求测延迟"""
        try:
            start = time.perf_counter()
            async with aiohttp.ClientSession() as session:
                async with session.head(url, timeout=aiohttp.ClientTimeout(total=timeout)) as resp:
                    pass
            return round((time.perf_counter() - start) * 1000, 2)
        except Exception:
            return 99999.0

class SpeedCalculator:
    """速度计算与评分"""

    @staticmethod
    def calculate_score(result: NodeResult, weights: dict = None) -> float:
        """综合评分：延迟(30%) + 速度(50%) + 稳定性(20%)"""
        w = weights or {"latency": 0.3, "speed": 0.5, "stability": 0.2}

        latency_score = max(0, 100 - result.latency_ms / 3)  # 延迟越低越好
        speed_score = min(100, result.download_speed_mbps / 2)  # 速度越高越好
        stability_score = (1 - result.packet_loss) * result.stability_score * 100

        score = (
            latency_score * w["latency"] +
            speed_score * w["speed"] +
            stability_score * w["stability"]
        )
        return round(score, 2)

async def test_single_node(node: dict, config: SpeedTestConfig) -> NodeResult:
    """测试单个节点，返回完整结果"""
    result = NodeResult(
        name=node.get("name", "unknown"),
        protocol=node.get("protocol", "unknown"),
        server=node.get("server", ""),
        port=node.get("port", 0)
    )

    # 步骤1: TCP 延迟测试（多次取平均）
    latencies = []
    for _ in range(config.test_count):
        lat = await LatencyTester.measure_tcp_latency(
            result.server, result.port, timeout=config.timeout_seconds
        )
        if lat < 99999:
            latencies.append(lat)
        await asyncio.sleep(0.3)

    if latencies:
        result.latency_ms = round(sum(latencies) / len(latencies), 2)
        result.stability_score = round(1 - (max(latencies) - min(latencies)) / max(latencies), 3)
    else:
        result.error = "连接超时"
        result.latency_ms = 99999.0

    # 步骤2: 丢包率检测（通过延迟异常推断）
    if len(latencies) >= 3:
        std_dev = (sum((x - result.latency_ms) ** 2 for x in latencies) / len(latencies)) ** 0.5
        result.packet_loss = round(min(std_dev / result.latency_ms * 0.1, 0.5), 4)
    else:
        result.packet_loss = 0.5

    # 步骤3: 综合评分
    result.final_score = SpeedCalculator.calculate_score(result)

    return result

async def batch_speedtest(nodes: List[dict], config: SpeedTestConfig = None) -> List[NodeResult]:
    """批量并发测速"""
    config = config or SpeedTestConfig()
    tasks = [test_single_node(node, config) for node in nodes]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    valid_results = [r for r in results if isinstance(r, NodeResult)]
    return sorted(valid_results, key=lambda x: x.final_score, reverse=True)

# 使用示例
if __name__ == "__main__":
    sample_nodes = [
        {"name": "HK-01", "protocol": "ss", "server": "103.45.67.89", "port": 8382},
        {"name": "JP-Tokyo-02", "protocol": "vmess", "server": "152.89.234.10", "port": 443},
        {"name": "US-LosAngeles-01", "protocol": "trojan", "server": "198.54.128.55", "port": 443},
    ]

    results = asyncio.run(batch_speedtest(sample_nodes))

    print("=" * 70)
    print(f"{'排名':<4} {'名称':<25} {'协议':<8} {'延迟(ms)':<10} {'评分':<8}")
    print("=" * 70)
    for i, r in enumerate(results, 1):
        print(f"{i:<4} {r.name:<25} {r.protocol:<8} {r.latency_ms:<10} {r.final_score:<8}")
    print("=" * 70)
```

**核心原理说明：**

| 阶段 | 技术手段 | 评价指标 |
|------|---------|---------|
| 连接建立 | `asyncio.open_connection`（TCP 直连） | 往返时延 RTT |
| 延迟采样 | 多次测量取平均 + 标准差过滤 | 稳定性评分 |
| 丢包推断 | 延迟方差突变检测 | 丢包率估算 |
| 综合评分 | 加权求和（可调权重） | 节点质量量化 |

### 1.2 PowerShell 全协议批量测速脚本

```powershell
<#
.SYNOPSIS
    Clash 节点批量测速脚本 - PowerShell 实现
    支持延迟测试 + HTTP 测速，输出排名报告

.DESCRIPTION
    - 使用 .NET TcpClient 进行 TCP 延迟测量
    - 支持 HTTP HEAD + GET 双阶段测速
    - 兼容 Windows PowerShell 5.1+ / PowerShell Core 7+
    - 配置文件驱动，无需修改脚本主体
#>

param(
    [string]$ConfigFile = "$PSScriptRoot\node_config.json",
    [int]$TimeoutSeconds = 8,
    [int]$TestRounds = 3,
    [switch]$ExportCSV,
    [switch]$Verbose
)

# ─────────────────────────────────────────────
# 配置区
# ─────────────────────────────────────────────
$SpeedTestUrl = "https://speed.cloudflare.com/__down?bytes=5242880"  # 5MB
$LatencyTestHost = "1.1.1.1"
$LatencyTestPort = 443

$ResultDir = "$PSScriptRoot\results"
if (-not (Test-Path $ResultDir)) {
    New-Item -ItemType Directory -Path $ResultDir | Out-Null
}

# ─────────────────────────────────────────────
# 辅助函数
# ─────────────────────────────────────────────
function Measure-TcpLatency {
    param([string]$Server, [int]$Port, [int]$TimeoutMs = 5000)

    try {
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        $tcp = New-Object System.Net.Sockets.TcpClient
        $task = $tcp.ConnectAsync($Server, $Port)
        $completed = $task.Wait($TimeoutMs)

        if ($completed -and $tcp.Connected) {
            $sw.Stop()
            $tcp.Close()
            return [math]::Round($sw.ElapsedMilliseconds, 2)
        } else {
            $tcp.Close()
            return $null
        }
    }
    catch {
        return $null
    }
}

function Measure-HttpSpeed {
    param(
        [string]$Server,
        [int]$Port,
        [string]$TestUrl,
        [int]$TimeoutSec = 8
    )

    try {
        $proxy = "$Server`:$Port"

        # 尝试不同协议的代理 URL
        $proxyUrls = @(
            "http://$proxy",
            "socks5://$proxy",
            "socks5h://$proxy"
        )

        foreach ($proxyUrl in $proxyUrls) {
            try {
                $handler = New-Object System.Net.Http.HttpClientHandler
                $handler.Proxy = New-Object System.Net.WebProxy($proxyUrl)
                $handler.UseProxy = $true
                $handler.AllowAutoRedirect = $true
                $handler.MaxAutomaticRedirections = 3

                $client = New-Object System.Net.Http.HttpClient($handler)
                $client.Timeout = [TimeSpan]::FromSeconds($TimeoutSec)
                $client.DefaultRequestHeaders.Add("User-Agent", "ClashSpeedTest/1.0")

                $stopwatch = [System.Diagnostics.Stopwatch]::StartNew()
                $response = $client.GetAsync($TestUrl).GetAwaiter().GetResult()
                $contentBytes = $response.Content.ReadAsByteArrayAsync().GetAwaiter().GetResult()
                $stopwatch.Stop()

                $elapsedSec = $stopwatch.ElapsedMilliseconds / 1000
                $sizeMB = $contentBytes.Length / 1MB
                $speedMbps = [math]::Round(($sizeMB * 8) / $elapsedSec, 2)

                $client.Dispose()
                $handler.Dispose()

                return @{
                    Success     = $true
                    SpeedMbps   = $speedMbps
                    SizeMB      = [math]::Round($sizeMB, 2)
                    ElapsedSec  = [math]::Round($elapsedSec, 3)
                    Protocol    = $proxyUrl.Split("://")[0]
                }
            }
            catch {
                continue
            }
        }

        return @{ Success = $false; Error = "所有协议尝试失败" }
    }
    catch {
        return @{ Success = $false; Error = $_.Exception.Message }
    }
}

function Get-StabilityScore {
    param([double[]]$Latencies)

    if ($Latencies.Count -lt 2) { return 0.0 }

    $avg = ($Latencies | Measure-Object -Average).Average
    $stdDev = [math]::Sqrt(($Latencies | ForEach-Object { [math]::Pow($_ - $avg, 2) } | Measure-Object -Average).Average)

    # 变异系数越小，稳定性越高
    $cv = if ($avg -gt 0) { $stdDev / $avg } else { 1.0 }
    $score = [math]::Max(0, [math]::Round((1 - $cv) * 100, 2))
    return $score
}

function Get-PacketLossRate {
    param([double[]]$Latencies, [double]$ThresholdMs = 500)

    if ($Latencies.Count -eq 0) { return 1.0 }

    $outliers = ($Latencies | Where-Object { $_ -ge $ThresholdMs }).Count
    $rate = $outliers / $Latencies.Count
    return [math]::Round($rate, 4)
}

function Export-NodeResults {
    param(
        [array]$Results,
        [string]$OutputPath
    )

    $csvContent = $Results | ForEach-Object {
        [PSCustomObject]@{
            Rank          = $_.Rank
            Name          = $_.Name
            Protocol      = $_.Protocol
            Server        = $_.Server
            Port          = $_.Port
            AvgLatency    = $_.AvgLatency
            MinLatency    = $_.MinLatency
            MaxLatency    = $_.MaxLatency
            Stability     = $_.Stability
            PacketLoss    = $_.PacketLoss
            DownloadMbps  = $_.DownloadMbps
            FinalScore    = $_.FinalScore
            Status        = $_.Status
        }
    }

    $csvContent | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    Write-Host "[导出] 排名报告已保存至: $OutputPath" -ForegroundColor Green
}

# ─────────────────────────────────────────────
# 主程序
# ─────────────────────────────────────────────
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  Clash 节点批量测速工具 v2.0" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan

# 加载节点配置
$nodeConfig = @()
if (Test-Path $ConfigFile) {
    $nodeConfig = Get-Content $ConfigFile -Raw | ConvertFrom-Json
    Write-Host "[配置] 成功加载 $ConfigFile，共 $($nodeConfig.Count) 个节点" -ForegroundColor Green
} else {
    Write-Host "[警告] 配置文件不存在: $ConfigFile" -ForegroundColor Yellow
    Write-Host "[提示] 请在同目录下创建 node_config.json" -ForegroundColor Yellow

    # 示例配置
    $nodeConfig = @(
        @{ Name = "HK-01"; Protocol = "ss"; Server = "103.45.67.89"; Port = 8382; Password = "" },
        @{ Name = "JP-东京-02"; Protocol = "vmess"; Server = "152.89.234.10"; Port = 443; UUID = "" },
        @{ Name = "US-LA-01"; Protocol = "trojan"; Server = "198.54.128.55"; Port = 443; Password = "" }
    )
}

Write-Host ""
Write-Host "[测速开始] 延迟测试轮次: $TestRounds | 超时: ${TimeoutSeconds}s" -ForegroundColor Yellow
Write-Host "-----------------------------------------------------------" -ForegroundColor DarkGray

$allResults = @()
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"

for ($i = 0; $i -lt $nodeConfig.Count; $i++) {
    $node = $nodeConfig[$i]
    $pct = [math]::Round(($i / $nodeConfig.Count) * 100)
    Write-Progress -Activity "测速进度" -Status "$($node.Name) ($($node.Server))" -PercentComplete $pct

    $latencies = @()

    # 阶段1: TCP 延迟多次采样
    for ($r = 1; $r -le $TestRounds; $r++) {
        $lat = Measure-TcpLatency -Server $node.Server -Port $node.Port -TimeoutMs ($TimeoutSeconds * 1000)
        if ($null -ne $lat) {
            $latencies += $lat
        }
        Start-Sleep -Milliseconds 200
    }

    if ($latencies.Count -eq 0) {
        $allResults += [PSCustomObject]@{
            Rank        = 0
            Name        = $node.Name
            Protocol    = $node.Protocol
            Server      = $node.Server
            Port        = $node.Port
            AvgLatency  = "超时"
            MinLatency  = "-"
            MaxLatency  = "-"
            Stability   = "0"
            PacketLoss  = "100%"
            DownloadMbps = "-"
            FinalScore  = 0
            Status      = "连接失败"
        }
        Write-Host "  [超时] $($node.Name) -> $($node.Server):$($node.Port)" -ForegroundColor Red
        continue
    }

    # 阶段2: HTTP 速度测试（可选，较慢）
    $httpResult = Measure-HttpSpeed -Server $node.Server -Port $node.Port -TestUrl $SpeedTestUrl -TimeoutSec $TimeoutSeconds
    $speedMbps = if ($httpResult.Success) { $httpResult.SpeedMbps } else { 0 }

    # 阶段3: 指标计算
    $avgLat = [math]::Round(($latencies | Measure-Object -Average).Average, 2)
    $minLat = [math]::Round(($latencies | Measure-Object -Minimum).Minimum, 2)
    $maxLat = [math]::Round(($latencies | Measure-Object -Maximum).Maximum, 2)
    $stability = Get-StabilityScore -Latencies $latencies
    $packetLoss = Get-PacketLossRate -Latencies $latencies

    # 阶段4: 综合评分（可自定义权重）
    $latencyScore = [math]::Max(0, 100 - $avgLat / 3)
    $speedScore = [math]::Min(100, $speedMbps * 2)
    $stabilityScore = $stability
    $finalScore = [math]::Round($latencyScore * 0.3 + $speedScore * 0.5 + $stabilityScore * 0.2, 2)

    $allResults += [PSCustomObject]@{
        Rank        = 0
        Name        = $node.Name
        Protocol    = $node.Protocol
        Server      = $node.Server
        Port        = $node.Port
        AvgLatency  = "$avgLat ms"
        MinLatency  = "$minLat ms"
        MaxLatency  = "$maxLat ms"
        Stability   = "$stability 分"
        PacketLoss  = "$([math]::Round($packetLoss * 100, 1))%"
        DownloadMbps = if ($speedMbps -gt 0) { "$speedMbps Mbps" } else { "-" }
        FinalScore  = $finalScore
        Status      = if ($speedMbps -gt 0) { "正常" } else { "低速" }
    }

    Write-Host "  [OK] $($node.Name) | 延迟: ${avgLat}ms | 速度: $speedMbps Mbps | 评分: $finalScore" -ForegroundColor Green
}

Write-Progress -Activity "测速进度" -Completed

# 排名
$ranked = $allResults | Sort-Object -Property FinalScore -Descending
$rank = 1
$ranked | ForEach-Object { $_.Rank = $rank; $rank++ }

# 输出
Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  测速结果排名" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

$ranked | Format-Table -AutoSize `
    Rank, Name, Protocol, AvgLatency, Stability, PacketLoss, DownloadMbps, FinalScore, Status

# 导出
if ($ExportCSV) {
    $csvPath = Join-Path $ResultDir "node_ranking_${timestamp}.csv"
    Export-NodeResults -Results $ranked -OutputPath $csvPath
}

# 最佳推荐
$best = $ranked | Select-Object -First 1
Write-Host ""
Write-Host "[推荐] 最佳节点: $($best.Name) (评分: $($best.FinalScore))" -ForegroundColor Magenta
Write-Host "-----------------------------------------------------------" -ForegroundColor DarkGray
Write-Host ""
```

**运行说明：**

```bash
# 基础用法
.\speedtest.ps1

# 指定配置文件 + 导出 CSV
.\speedtest.ps1 -ConfigFile .\my_nodes.json -ExportCSV

# 调试模式
.\speedtest.ps1 -Verbose

# PowerShell Core（跨平台）
pwsh -File ./speedtest.ps1 -ExportCSV
```

**输出示例：**

```
========================================
  Clash 节点批量测速工具 v2.0
========================================

  [OK] HK-01 | 延迟: 25.3ms | 速度: 156.8 Mbps | 评分: 87.5
  [OK] JP-东京-02 | 延迟: 41.7ms | 速度: 298.2 Mbps | 评分: 91.3
  [超时] US-LA-01 -> 198.54.128.55:443
  [OK] SG-01 | 延迟: 58.2ms | 速度: 89.4 Mbps | 评分: 72.6

========================================
  测速结果排名
========================================

Rank Name       Protocol AvgLatency Stability PacketLoss DownloadMbps FinalScore Status
---- ---------- -------- ---------- --------- --------- ------------ ---------- ------
   1 JP-东京-02  vmess    41.7 ms    95.2 分   0%        298.2 Mbps   91.3      正常
   2 HK-01       ss       25.3 ms    88.7 分   0%        156.8 Mbps   87.5      正常
   3 SG-01       ss       58.2 ms    82.1 分   2.1%      89.4 Mbps    72.6      低速
   4 US-LA-01    trojan   超时       0         100%      -            0          连接失败

[推荐] 最佳节点: JP-东京-02 (评分: 91.3)
```

---

## 二、节点筛选规则

测速只是第一步，真正的优选在于规则设计——什么节点该留、什么该弃、优先级如何排。

### 2.1 筛选维度与阈值设计

| 筛选维度 | 基础阈值 | 推荐阈值 | 严选阈值 | 筛选逻辑 |
|---------|---------|---------|---------|---------|
| 延迟（ms） | ≤ 200 | ≤ 100 | ≤ 60 | 取多次测量的中位数而非平均值 |
| 下载速度（Mbps） | ≥ 10 | ≥ 50 | ≥ 100 | 使用 P95 百分位避免突发值 |
| 丢包率（%） | ≤ 5% | ≤ 1% | ≤ 0.5% | 连续 10 次采样中丢包占比 |
| 抖动（ms） | ≤ 50 | ≤ 20 | ≤ 10 | 标准差，反映稳定性 |
| TLS 握手（ms） | ≤ 100 | ≤ 50 | ≤ 30 | 反映服务器 TLS 性能 |
| TCP RTT 方差 | ≤ 30% | ≤ 15% | ≤ 8% | 延迟波动幅度 |

### 2.2 多级复合筛选规则

```
规则优先级: 丢包率(死线) > 延迟(硬性) > 速度(软性) > 抖动(加分)
```

**实现逻辑（伪代码）：**

```python
def filter_nodes(nodes: List[NodeResult]) -> List[NodeResult]:
    """
    三级筛选: 死线过滤 -> 排名打分 -> Top N 选取
    """
    # 第一级: 死线过滤（任何一项超过阈值直接剔除）
    dead_line = {
        "latency_ms": 200,
        "packet_loss": 0.05,
        "download_speed_mbps": 5,
    }

    passed = []
    for node in nodes:
        if (node.latency_ms > dead_line["latency_ms"] or
            node.packet_loss > dead_line["packet_loss"] or
            node.download_speed_mbps < dead_line["download_speed_mbps"]):
            continue  # 淘汰
        passed.append(node)

    # 第二级: 综合评分排名
    scored = sorted(passed, key=lambda x: x.final_score, reverse=True)

    # 第三级: 选取 Top N，并确保地域多样性
    # 同一地区最多保留 3 个节点
    region_count = {}
    final = []
    for node in scored:
        region = extract_region(node.name)  # 从名称解析地区
        count = region_count.get(region, 0)
        if count < 3:
            final.append(node)
            region_count[region] = count + 1

        if len(final) >= 10:
            break

    return final

def extract_region(name: str) -> str:
    """从节点名称提取地区代码"""
    region_map = {
        "HK": ["香港", "HK", "HongKong", "HKG"],
        "TW": ["台湾", "TW", "Taiwan", "TPE"],
        "JP": ["日本", "JP", "Japan", "TYO", "东京", "大阪"],
        "KR": ["韩国", "KR", "Korea", "SEOUL"],
        "SG": ["新加坡", "SG", "Singapore", "SIN"],
        "US": ["美国", "US", "America", "LosAngeles", "LA", "NewYork", "NY"],
        "UK": ["英国", "UK", "London"],
        "DE": ["德国", "DE", "Germany", "Frankfurt"],
    }
    for region, keywords in region_map.items():
        for kw in keywords:
            if kw.lower() in name.lower():
                return region
    return "OTHER"
```

### 2.3 动态阈值调整策略

静态阈值的问题在于无法适应不同时间段的网络波动。建议采用**自适应阈值**：

```python
class AdaptiveThresholds:
    """根据历史数据动态调整筛选阈值"""

    def __init__(self, history_window: int = 50):
        self.history = {}  # node_id -> list of recent measurements
        self.window = history_window

    def update(self, node_id: str, measurement: dict):
        if node_id not in self.history:
            self.history[node_id] = []
        self.history[node_id].append(measurement)
        # 保留最近 N 条记录
        if len(self.history[node_id]) > self.window:
            self.history[node_id].pop(0)

    def get_thresholds(self, node_id: str) -> dict:
        """基于该节点历史表现返回个性化阈值"""
        if node_id not in self.history or len(self.history[node_id]) < 5:
            return self._default_thresholds()

        recent = self.history[node_id]
        latencies = [m["latency_ms"] for m in recent]
        speeds = [m["download_speed_mbps"] for m in recent]

        # 使用 P80 作为基准（比平均值更能反映常态）
        p80_lat = sorted(latencies)[int(len(latencies) * 0.8)]
        p80_speed = sorted(speeds)[int(len(speeds) * 0.8)]

        return {
            "latency_ms": p80_lat * 1.5,        # 放宽50%
            "download_speed_mbps": p80_speed * 0.5,  # 放宽50%
            "packet_loss": 0.02,
            "stability_min": 70.0,
        }

    def _default_thresholds(self) -> dict:
        return {
            "latency_ms": 150.0,
            "download_speed_mbps": 20.0,
            "packet_loss": 0.03,
            "stability_min": 75.0,
        }
```

### 2.4 稳定性评分算法

稳定性是容易被忽视但至关重要的指标。一个偶尔跑出超低延迟但大部分时间高延迟的节点，不如一个始终保持中等延迟的节点。

```python
def calculate_stability_score(latencies: List[float]) -> float:
    """
    综合稳定性评分（0-100分）
    基于三个维度: 一致性、抖动率、异常频率
    """
    if len(latencies) < 3:
        return 50.0  # 数据不足给中等分

    avg = sum(latencies) / len(latencies)
    variance = sum((x - avg) ** 2 for x in latencies) / len(latencies)
    std_dev = variance ** 0.5

    # 一致性: 变异系数(CV)越小越好
    cv = std_dev / avg if avg > 0 else 1.0
    consistency_score = max(0, (1 - cv * 5)) * 40  # 满分40

    # 抖动率: 最大延迟 / 最小延迟，越接近1越好
    jitter_ratio = min(latencies) / max(latencies) if max(latencies) > 0 else 0
    jitter_score = jitter_ratio * 30  # 满分30

    # 异常频率: 超阈值延迟的出现比例
    threshold = avg * 1.5
    anomalies = sum(1 for x in latencies if x > threshold)
    anomaly_ratio = anomalies / len(latencies)
    anomaly_score = (1 - anomaly_ratio) * 30  # 满分30

    total = consistency_score + jitter_score + anomaly_score
    return round(min(total, 100), 2)
```

---

## 三、负载均衡配置

优选完成后，如何让流量在各节点间合理分配？Clash 的负载均衡策略决定了优选成果的实际体验。

### 3.1 负载均衡策略对比

| 策略 | 配置关键字 | 适用场景 | 优缺点 |
|------|-----------|---------|--------|
| **自由选择** | `url-test` | 单节点高速优先 | 简单，但无容灾 |
| **故障转移** | `fallback` | 高可用优先 | 主节点故障才切换，有延迟 |
| **阶梯负载** | `load-balance` | 均衡+容灾 | 自动分配，配置稍复杂 |
| **手动选择** | `select` | 特定地区/用途 | 完全可控，但需手动切换 |
| **优先静态** | `consistent-hashing` | 大流量长期连接 | 保持会话一致性 |

### 3.2 `url-test` 策略配置

```yaml
proxy-groups:
  # 基础版: 自动选择最快节点
  - name: "🚀 节点选择"
    type: url-test
    # 使用哪些节点
    use:
      - node-list
    # 健康检查 URL（轻量）
    url: "http://www.gstatic.com/generate_204"
    # 间隔 N 秒重新测速
    interval: 300
    # 容差: 延迟差在范围内视为同等优
    tolerance: 50
    # 懒切换: 只在新连接时切换，不中断已有流量
    lazy: true
    # 是否禁用uds（Windows 兼容性）
    disable-udp: false

  # 进阶版: 按地区分组
  - name: "🌏 香港节点"
    type: url-test
    use:
      - hk-nodes
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 30
    lazy: true

  - name: "🌸 日本节点"
    type: url-test
    use:
      - jp-nodes
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 30
    lazy: true

  - name: "🗽 美国节点"
    type: url-test
    use:
      - us-nodes
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true
```

### 3.3 `fallback` 故障转移策略

```yaml
proxy-groups:
  - name: "🔄 故障转移组"
    type: fallback
    use:
      - node-list
    url: "http://www.gstatic.com/generate_204"
    # 健康检查间隔
    interval: 300
    # 故障检测: 连续 N 次失败才判定节点不可用
    check-interval: 5
    # 策略: 优先第一个，第一个挂了用第二个
    # 配置中节点列表的顺序即为优先级顺序
```

**fallback vs url-test 的关键区别：**

- `url-test`: 所有健康节点持续竞争，始终选择最优者（延迟最低）
- `fallback`: 始终使用列表中第一个可用节点，只在故障时切换

### 3.4 权重配置（高级）

在某些订阅中可使用权重参数控制各节点的选中概率：

```yaml
proxy-groups:
  - name: "⚖️ 权重负载"
    type: url-test
    # 节点 + 权重（权重越高被选中概率越大）
    proxies:
      - name: "JP-01 (优选)"
        # 权重通过在配置中的相对比例体现
      - name: "JP-02 (备用)"
```

> 注意: Clash 基础版不支持显式权重参数，权重隐式体现在 url-test 的 `tolerance` 和 `interval` 调优中。

### 3.5 最佳实践: 多层负载均衡架构

```
用户流量
  │
  ▼
┌─────────────────────────────┐
│  第一层: 地区选择 (select)  │
│  ┌─────┬─────┬─────┬─────┐ │
│  │HK   │JP   │US   │SG   │ │
│  └─────┴─────┴─────┴─────┘ │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  第二层: 节点选择 (url-test)│
│  同一地区内自动测速选择最优 │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  第三层: 故障转移 (fallback)│
│  当最优节点失效时自动切换   │
└─────────────────────────────┘
```

**推荐配置模板：**

```yaml
proxy-groups:
  # 入口: 按用途/地区手动选择
  - name: "🔀 主路由"
    type: select
    proxies:
      - 🚀 自动最优
      - 🌏 香港节点
      - 🌸 日本节点
      - 🗽 美国节点
      - 🌴 新加坡节点
      - 🔄 故障转移
      - DIRECT

  # 自动最优: url-test + 懒切换
  - name: "🚀 自动最优"
    type: url-test
    use:
      - all-nodes
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true

  # 故障转移: 高可用场景
  - name: "🔄 故障转移"
    type: fallback
    use:
      - high-priority-nodes
    url: "http://www.gstatic.com/generate_204"
    interval: 300
```

---

## 四、智能切换算法

传统负载均衡的痛点：节点"看似健康"但实际体验差（DNS 污染、TCP 阻断、特定端口限速等）。智能切换需要更深层的检测机制。

### 4.1 健康检测机制设计

**三层检测模型：**

```
L1 网络层: TCP 握手延迟 < 阈值？
  └─ L2 应用层: HTTP HEAD 请求响应时间 < 阈值？
       └─ L3 业务层: 实际目标网站可达性测试
```

```python
class HealthChecker:
    """三层健康检测器"""

    def __init__(self):
        self.l1_targets = [("1.1.1.1", 443), ("8.8.8.8", 853)]
        self.l2_targets = [
            "http://www.gstatic.com/generate_204",
            "http://cp.cloudflare.com/generate_204",
        ]
        self.l3_targets = [
            "https://www.google.com",
            "https://www.youtube.com",
            "https://github.com",
        ]

    async def check_l1(self, proxy_url: str) -> tuple[bool, float]:
        """L1: TCP 连接性检测"""
        # 使用代理建立 TCP 连接测量延迟
        # 返回 (是否成功, 延迟ms)
        pass

    async def check_l2(self, proxy_url: str) -> tuple[bool, float]:
        """L2: HTTP 应用层检测"""
        # 通过代理发起 HTTP HEAD 请求
        pass

    async def check_l3(self, proxy_url: str) -> tuple[bool, float]:
        """L3: 业务层可达性检测"""
        # 模拟真实访问目标网站
        pass

    async def full_check(self, proxy_url: str) -> dict:
        """完整三层检测"""
        l1_ok, l1_lat = await self.check_l1(proxy_url)
        if not l1_ok:
            return {"healthy": False, "level": 0, "reason": "L1 TCP连接失败"}

        l2_ok, l2_lat = await self.check_l2(proxy_url)
        if not l2_ok:
            return {"healthy": False, "level": 1, "reason": "L2 HTTP请求失败"}

        l3_ok, l3_lat = await self.check_l3(proxy_url)
        if not l3_ok:
            return {"healthy": True, "level": 2, "reason": "L3 业务层部分阻断", "score": 60}

        score = self._calculate_health_score(l1_lat, l2_lat, l3_lat)
        return {"healthy": True, "level": 3, "score": score}

    def _calculate_health_score(self, l1: float, l2: float, l3: float) -> float:
        """健康评分: 三层延迟加权"""
        # 权重: L1(20%) + L2(30%) + L3(50%)
        l1_score = max(0, 100 - l1 / 2)
        l2_score = max(0, 100 - l2 / 2)
        l3_score = max(0, 100 - l3 / 2)
        return round(l1_score * 0.2 + l2_score * 0.3 + l3_score * 0.5, 2)
```

### 4.2 智能切换触发条件

不是所有网络波动都需要切换。设置合理的阈值避免"抖动切换"（频繁在节点间跳动）：

| 触发条件 | 推荐值 | 说明 |
|---------|--------|------|
| 连续失败次数 | ≥ 3 | 避免单次网络抖动的误触发 |
| 延迟超阈值持续时间 | ≥ 10s | 短期波动不切换 |
| 性能差距阈值 | ≥ 30% | 新节点比当前节点快 30% 才切换 |
| 最低切换间隔 | ≥ 60s | 防止切换风暴 |
| 恢复检测 | 连续 3 次成功 | 才认为节点恢复 |

```python
class SmartSwitcher:
    """智能切换器 - 带防抖动机制"""

    def __init__(self):
        self.failure_count = {}   # node_id -> 连续失败次数
        self.consecutive_ok = {}  # node_id -> 连续成功次数
        self.last_switch_time = {}  # node_id -> 上次切换时间戳
        self.switch_cooldown = 60   # 切换冷却期(秒)

    def should_switch(self, current_node: str, candidates: List[dict]) -> Optional[str]:
        """
        判断是否需要切换节点
        返回 None = 不切换，返回节点名 = 切换到该节点
        """
        now = time.time()

        # 冷却期检查
        if current_node in self.last_switch_time:
            if now - self.last_switch_time[current_node] < self.switch_cooldown:
                return None  # 仍在冷却期

        # 检查当前节点健康状态
        current_failures = self.failure_count.get(current_node, 0)
        if current_failures >= 3:
            # 连续失败 3 次，强制切换
            best = self._select_best_candidate(candidates)
            if best and best["name"] != current_node:
                self.last_switch_time[current_node] = now
                self.failure_count[current_node] = 0
                return best["name"]

        # 检查是否有更好的候选节点
        current_score = self._get_node_score(current_node)
        best = self._select_best_candidate(candidates)

        if best:
            best_score = best["score"]
            performance_gap = (current_score - best_score) / current_score if current_score > 0 else 0

            if performance_gap > 0.3 and best["name"] != current_node:
                # 性能差距超过 30%，且候选节点连续成功
                if self.consecutive_ok.get(best["name"], 0) >= 3:
                    self.last_switch_time[current_node] = now
                    return best["name"]

        return None

    def record_result(self, node_id: str, success: bool):
        """记录检测结果"""
        if success:
            self.failure_count[node_id] = 0
            self.consecutive_ok[node_id] = self.consecutive_ok.get(node_id, 0) + 1
        else:
            self.failure_count[node_id] = self.failure_count.get(node_id, 0) + 1
            self.consecutive_ok[node_id] = 0

    def _select_best_candidate(self, candidates: List[dict]) -> Optional[dict]:
        """从候选节点中选择最优者"""
        valid = [c for c in candidates if self.consecutive_ok.get(c["name"], 0) >= 1]
        if not valid:
            return None
        return max(valid, key=lambda x: x["score"])

    def _get_node_score(self, node_id: str) -> float:
        """获取节点评分（简化版）"""
        return 100 - self.failure_count.get(node_id, 0) * 20
```

### 4.3 自动恢复机制

节点恢复后不应立即切回，而是通过"渐进验证"确认稳定性：

```
节点状态机:
  HEALTHY ──(连续3次失败)──► DEGRADED ──(再连续3次失败)──► FAILED
  HEALTHY <──(连续5次成功)── DEGRADED <──(连续3次成功)── FAILED

DEGRADED 状态: 降低权重但保留参与负载均衡的资格
FAILED 状态: 完全剔除，触发 fallback 切换
```

---

## 五、优选工具对比

### 5.1 主流工具功能对比

| 工具名称 | 协议支持 | 测速方式 | 自动化程度 | 负载均衡 | 智能切换 | 适用系统 | 上手难度 |
|---------|---------|---------|-----------|---------|---------|---------|---------|
| **Clash Verge** | 全协议 | 内置 | ⭐⭐⭐ | ✅ 内置 | ✅ 基础 | Win/Mac/Linux | ⭐ |
| **Stash** | 全协议 | 内置 | ⭐⭐⭐ | ✅ 内置 | ✅ 基础 | iOS/Mac | ⭐⭐ |
| **Shadowrocket** | 全协议 | 内置 | ⭐⭐ | ❌ | ✅ 基础 | iOS | ⭐ |
| **Surge** | 全协议 | 外部脚本 | ⭐⭐⭐⭐ | ✅ 高级 | ✅ 高级 | Mac/iOS | ⭐⭐⭐ |
| **sing-box** | 全协议 | 外部脚本 | ⭐⭐⭐⭐ | ✅ 高级 | ✅ 高级 | 全平台 | ⭐⭐⭐ |
| **OpenClash** | 全协议 | 外部脚本 | ⭐⭐⭐ | ✅ 内置 | ✅ 基础 | OpenWrt | ⭐⭐ |

### 5.2 sing-box 高级负载均衡配置

sing-box 是目前功能最全面的代理核心，支持精细化的负载均衡策略：

```json
{
  "dns": {
    "servers": [
      {
        "tag": "google",
        "address": "https://dns.google/dns-query",
        "detour": "auto"
      }
    ]
  },
  "route": {
    "auto_detect_interface": true
  },
  "inbounds": [
    {
      "tag": "mixed",
      "type": "mixed",
      "listen": "127.0.0.1",
      "listen_port": 7890
    }
  ],
  "outbounds": [
    {
      "tag": "url-test-group",
      "type": "urltest",
      "use": ["node-1", "node-2", "node-3"],
      "url": "http://www.gstatic.com/generate_204",
      "interval": "5m",
      "tolerance": 50,
      "lazy": true
    },
    {
      "tag": "fallback-group",
      "type": "fallback",
      "use": ["node-1", "node-2"],
      "url": "http://www.gstatic.com/generate_204",
      "interval": "5m",
      "check_interval": "10s"
    },
    {
      "tag": "load-balance-group",
      "type": "load-balance",
      "use": ["node-1", "node-2", "node-3", "node-4"],
      "strategy": "consistent-hashing",
      "url": "http://www.gstatic.com/generate_204",
      "interval": "5m"
    },
    {
      "tag": "relay-group",
      "type": "relay",
      "use": ["node-1", "node-2"]
    }
  ],
  "route_rules": [
    {
      "geosite": "category-games-cn",
      "geosite": "Bilibili",
      "geosite": "Netflix",
      "geosite": "YouTube",
      "geosite": "Google",
      "geosite": "Telegram",
      "outbound": "url-test-group"
    },
    {
      "geosite": "category-ads-all",
      "outbound": "block"
    }
  ]
}
```

**sing-box 特有的高级策略：**

- **consistent-hashing**: 一致性哈希，相同目标始终路由到同一节点，适合长连接
- **relay**: 链式代理，A -> B -> C，适合优化路由路径或规避封锁
- **selector**: 手动选择，支持动态切换

### 5.3 工具选型建议

```
日常使用 + 快速上手 ──────► Clash Verge / Stash
高度自定义 + 多设备 ──────► sing-box
macOS 原生体验 ──────────► Surge
OpenWrt 路由器 ──────────► OpenClash
iOS 移动端 ─────────────► Shadowrocket / Stash
企业级 + 复杂网络 ─────────► sing-box + 自定义脚本
```

---

## 六、自定义优选方法

当现有工具无法满足需求时，自建优选方案是进阶之路。

### 6.1 自建测速平台架构

```
┌──────────────────────────────────────────────────────┐
│                   测速调度中心                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ 节点管理器  │  │ 调度引擎   │  │ 结果存储   │    │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘    │
│        │               │               │            │
│        ▼               ▼               ▼            │
│  ┌──────────────────────────────────────────────┐  │
│  │          测速 Worker 集群（可水平扩展）       │  │
│  │  Worker-1 │ Worker-2 │ Worker-3 │ Worker-N  │  │
│  └──────────────────────────────────────────────┘  │
│                          │                          │
│                          ▼                          │
│  ┌──────────────────────────────────────────────┐  │
│  │           评分引擎 + 规则引擎                 │  │
│  └──────────────────────────────────────────────┘  │
│                          │                          │
│                          ▼                          │
│              生成优化后的订阅配置文件                 │
└──────────────────────────────────────────────────────┘
```

### 6.2 节点质量评分模型

引入机器学习思想，为每个节点计算多维质量分：

```python
class NodeQualityScorer:
    """
    多维节点质量评分模型
    维度: 延迟、带宽、稳定性、可用性、地理位置
    """

    # 各维度权重（可自定义）
    WEIGHTS = {
        "latency": 0.25,        # 延迟权重
        "bandwidth": 0.30,      # 带宽权重
        "stability": 0.20,      # 稳定性权重
        "availability": 0.15,   # 可用性权重
        "geo": 0.10,            # 地理位置权重
    }

    def __init__(self):
        self.history_db = {}  # 节点历史数据

    def score(self, node: dict) -> float:
        latency_score = self._score_latency(node.get("latency_ms", 999))
        bandwidth_score = self._score_bandwidth(node.get("download_mbps", 0))
        stability_score = self._score_stability(node.get("stability", 0))
        availability_score = self._score_availability(node.get("uptime_hours", 0))
        geo_score = self._score_geo(node.get("region", "OTHER"))

        total = (
            latency_score * self.WEIGHTS["latency"] +
            bandwidth_score * self.WEIGHTS["bandwidth"] +
            stability_score * self.WEIGHTS["stability"] +
            availability_score * self.WEIGHTS["availability"] +
            geo_score * self.WEIGHTS["geo"]
        )
        return round(total, 2)

    def _score_latency(self, latency: float) -> float:
        """延迟评分: 指数衰减模型"""
        if latency >= 500:
            return 0.0
        return round(100 * (2 ** (-latency / 150)), 2)

    def _score_bandwidth(self, bandwidth: float) -> float:
        """带宽评分: 对数增长模型"""
        if bandwidth <= 0:
            return 0.0
        return round(min(100, 30 * (bandwidth ** 0.5)), 2)

    def _score_stability(self, stability: float) -> float:
        """稳定性评分: 直接映射 0-100"""
        return min(100, max(0, stability))

    def _score_availability(self, uptime_hours: float) -> float:
        """可用性评分: 累计在线时长"""
        # 24h = 60分, 168h(一周) = 80分, 720h(一月) = 95分
        if uptime_hours <= 0:
            return 0.0
        return round(min(100, 40 + 55 * (1 - 2 ** (-uptime_hours / 200))), 2)

    def _score_geo(self, region: str) -> float:
        """地理位置评分: 距离越近分数越高"""
        geo_scores = {
            "HK": 100, "TW": 95, "SG": 90,
            "JP": 88, "KR": 85, "US": 60,
            "UK": 50, "DE": 55, "OTHER": 40
        }
        return geo_scores.get(region, 40)

    def generate_report(self, nodes: List[dict]) -> str:
        """生成评测报告"""
        scored = [(n, self.score(n)) for n in nodes]
        ranked = sorted(scored, key=lambda x: x[1], reverse=True)

        report = ["# 节点质量评测报告", "", "| 排名 | 节点名称 | 地区 | 延迟 | 带宽 | 稳定性 | 总分 |", "|------|-----------|------|------|------|------|----|"]
        for i, (node, score) in enumerate(ranked, 1):
            report.append(f"| {i} | {node['name']} | {node.get('region','N/A')} | "
                          f"{node.get('latency_ms','N/A')}ms | {node.get('download_mbps','N/A')}Mbps | "
                          f"{node.get('stability','N/A')} | {score} |")
        return "\n".join(report)
```

### 6.3 个性化策略配置

根据使用场景定制不同的优选策略：

| 使用场景 | 优先指标 | 推荐策略 | 配置重点 |
|---------|---------|---------|---------|
| **游戏加速** | 延迟 + 抖动 | url-test（低 tolerance） | interval=60, tolerance=10 |
| **视频流媒体** | 带宽 + 稳定性 | load-balance | 开启 consistent-hashing |
| **日常浏览** | 延迟 + 功耗 | url-test（懒切换） | lazy=true, interval=300 |
| **大文件下载** | 带宽 + 多线程 | relay + 负载均衡 | 多节点并发 |
| **学术搜索** | 稳定性 + 地域 | fallback | 主备双节点 |
| **金融操作** | 稳定性 + 低延迟 | url-test（严格阈值） | tolerance=5, interval=120 |

### 6.4 自动订阅更新 + 优选脚本工作流

```bash
#!/bin/bash
# auto_optimize.sh - 自动化订阅更新与优选工作流
# 建议配合 cron 每日执行

set -e

SUBSCRIBE_URL="https://your-subscribe-url/here"
CONFIG_DIR="$HOME/.config/clash"
BACKUP_DIR="$HOME/.config/clash/backups"
TEMP_DIR="/tmp/clash_optimize_$$"

mkdir -p "$BACKUP_DIR" "$TEMP_DIR"

echo "[$(date)] 开始订阅更新与优选流程..."

# 1. 获取订阅
echo "[1/6] 下载订阅..."
curl -sL "$SUBSCRIBE_URL" -o "$TEMP_DIR/original.yaml"

# 2. 备份当前配置
if [ -f "$CONFIG_DIR/config.yaml" ]; then
    cp "$CONFIG_DIR/config.yaml" "$BACKUP_DIR/config_$(date +%Y%m%d_%H%M%S).yaml"
fi

# 3. 运行测速脚本
echo "[2/6] 执行节点测速..."
cd "$TEMP_DIR"
python3 speedtest.py --input original.yaml --output ranked_nodes.json --rounds 3

# 4. 应用筛选规则
echo "[3/6] 应用筛选规则..."
python3 filter_nodes.py \
    --input ranked_nodes.json \
    --output filtered.yaml \
    --max-latency 150 \
    --min-speed 20 \
    --max-packet-loss 0.02

# 5. 合并到主配置
echo "[4/6] 合并配置..."
python3 merge_config.py \
    --base "$CONFIG_DIR/template.yaml" \
    --nodes filtered.yaml \
    --output "$CONFIG_DIR/config.yaml"

# 6. 验证配置语法
echo "[5/6] 验证配置..."
if command -v clash &> /dev/null; then
    clash -t -d "$CONFIG_DIR" -f "$CONFIG_DIR/config.yaml" && \
    echo "[OK] 配置验证通过" || \
    echo "[警告] 配置验证失败，回退到备份" && \
    cp "$BACKUP_DIR"/*.yaml "$CONFIG_DIR/config.yaml" 2>/dev/null || true
fi

# 7. 清理
echo "[6/6] 清理临时文件..."
rm -rf "$TEMP_DIR"

echo "[$(date)] 优选流程完成"
```

---

## 七、综合实战方案

### 7.1 推荐的 Clash 配置模板

以下是一套生产可用的完整配置，整合了前面所有章节的最佳实践：

```yaml
# Clash 配置文件模板 - 节点优选版
# 适用: Clash Verge / Clash for Windows

# DNS 配置（防止 DNS 污染）
dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  fallback:
    - 8.8.8.8
    - 1.1.1.1
  fallback-filter:
    geoip: true
    geoip-code: CN

# 代理节点列表（由订阅自动填充）
proxies:
  # 节点将从订阅自动导入

proxy-providers:
  # 订阅提供者
  my-sub:
    type: http
    url: "https://your-subscribe-url"
    interval: 3600
    path: "./providers/my-sub.yaml"
    health-check:
      enable: true
      url: "http://www.gstatic.com/generate_204"
      interval: 300

# 代理组配置
proxy-groups:
  # ── 一级入口：用途选择 ──
  - name: "🔀 主路由"
    type: select
    proxies:
      - 🚀 自动最优（国内）
      - 🌏 香港节点
      - 🌸 日本节点
      - 🗽 美国节点
      - 🌴 新加坡节点
      - 🔄 高可用故障转移
      - DIRECT

  # ── 二级：自动最优（懒切换） ──
  - name: "🚀 自动最优（国内）"
    type: url-test
    use:
      - my-sub
    url: "http://www.gstatic.com/generate_204"
    interval: 300        # 每5分钟测速一次
    tolerance: 50        # 50ms 延迟差内不切换
    lazy: true           # 懒切换，不断开已有连接

  # ── 二级：地区分组 ──
  - name: "🌏 香港节点"
    type: url-test
    use:
      - my-sub
    filter: "香港|HK|HongKong|HKG"
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 30
    lazy: true

  - name: "🌸 日本节点"
    type: url-test
    use:
      - my-sub
    filter: "日本|JP|Japan|TYO|东京|大阪"
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 30
    lazy: true

  - name: "🗽 美国节点"
    type: url-test
    use:
      - my-sub
    filter: "美国|US|America|LA|NY|LosAngeles|NewYork"
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 80
    lazy: true

  - name: "🌴 新加坡节点"
    type: url-test
    use:
      - my-sub
    filter: "新加坡|SG|Singapore|SIN"
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true

  # ── 二级：高可用故障转移 ──
  - name: "🔄 高可用故障转移"
    type: fallback
    use:
      - my-sub
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    check-interval: 5    # 每5秒检测一次故障

# 路由规则
rules:
  # ── 国内直连 ──
  - GEOIP,CN,DIRECT
  - MATCH,🚀 自动最优（国内）
```

### 7.2 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 节点评分正常但无法连接 | 节点被 TCP 阻断 | 切换到 SSR/V2Ray 协议节点 |
| 频繁自动切换导致卡顿 | tolerance 过低 | 调高 tolerance 至 50-80 |
| 订阅更新后节点消失 | 节点被下架 | 手动补充或重新订阅 |
| 测速结果与实际体验不符 | 测速目标与实际访问目标不同 | 使用业务层(L3)测速 |
| 故障转移无响应 | check-interval 太长 | 缩短至 5s |
| 负载均衡下某些节点从未被选中 | 节点质量太差 | 提高筛选阈值或剔除该节点 |

### 7.3 性能优化建议

1. **测速频率平衡**: interval 太短增加开销，太长影响切换及时性。生产环境推荐 300s（5分钟）
2. **订阅缓存**: 使用 `proxy-providers` 的 `path` 参数缓存订阅，避免频繁拉取
3. **地区过滤**: 在 `filter` 参数中使用正则精确匹配地区，避免误匹配
4. **TFO (TCP Fast Open)**: 开启后可降低首次连接延迟（需系统支持）
5. **mux 多路复用**: 适合高延迟链路（如跨洲节点），可显著提升体验

---

## 推广资源

> 以下为本站长期合作的优质节点服务商与工具站，点击访问获取更多资源：

| 类别 | 名称 | 链接 |
|------|------|------|
| 🌟 **VIP 节点订阅** | ClashVIP - 专业节点服务商 | https://nav.clashvip.net |
| 🌟 **备用节点订阅** | VPSVIP - 高性价比节点 | https://vpsvip.net |
| 🌟 **镜像站点** | 备用访问入口 | https://clashvip.net |
| 📦 **工具下载** | ClashHub - 工具集合站 | https://clashhub.net |
| 💬 **社区论坛** | ClashHub 社区 | https://bbs.clashhub.net |
| 🪟 **Clash for Windows** | 官方下载 | https://clash-for-windows.net |
| 📱 **节点服务** | VPSVIP 节点站 | https://vpsvip.net |

---

*最后更新: 2026-08-20*

# Cloudflare 反代IP扫描提取工具

![许可协议](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.14+-yellow.svg)
![平台](https://img.shields.io/badge/platform-Windows-green.svg)

一款功能强大的 Windows 图形界面工具，用于从自治系统号（ASN）中扫描和提取 Cloudflare 反代 IP。

## 功能特点

- **ASN IP 段提取**：从 ipinfo.io 或 whois.ipip.net 获取任意 ASN 的所有 IP 段
- **端口扫描**：使用 masscan 扫描 ASN 范围内所有 IP 的开放端口
- **Cloudflare 检测**：识别哪些 IP 是 Cloudflare 的反向代理
- **实时进度显示**：所有操作过程都有实时进度和日志跟踪
- **智能跳过**：自动跳过已完成步骤，复用之前的结果
- **暂停/继续**：支持随时暂停和恢复 Cloudflare 检测
- **多线程支持**：可配置线程数，快速检测
- **实时保存**：检测到一个 CF IP 就实时保存到文件

## 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                   Cloudflare 反代IP扫描提取工具                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   步骤1     │───▶│   步骤2     │───▶│   步骤3     │      │
│  │  ASN IP     │    │   Masscan   │    │   CF        │      │
│  │  提取       │    │   端口扫描   │    │   检测      │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                   │                   │              │
│         ▼                   ▼                   ▼              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ AS{ASN}_ip   │    │ AS{ASN}_open │    │ AS{ASN}_cf   │      │
│  │ _ranges.txt  │    │ _ips.txt     │    │ _ips.txt     │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

工作流程：
1. 用户输入 ASN 号码（例如：40065）
2. 工具获取该 ASN 下的所有 IP 段
3. 工具使用 masscan 扫描开放端口（80, 443, 8080）
4. 工具测试每个开放 IP，检测是否为 Cloudflare 反代
5. 验证后的 CF IP 可在 proxyip.cmliussss.net 进行二次验证
```

## 工作原理

### 步骤1：ASN IP 段提取

工具查询 `ipinfo.io/AS{ASN}` 和 `whois.ipip.net/AS{ASN}` 获取该 ASN 关联的所有 IP 段，保存到 `AS{ASN}_ip_ranges.txt`。

### 步骤2：使用 Masscan 进行端口扫描

Masscan 在 ASN 范围内的所有 IP 上执行高速端口扫描。扫描常用端口（80, 443, 8080），以 XML 格式输出结果。只保留有开放端口的 IP，格式为 `IP:端口`，保存到 `AS{ASN}_open_ips.txt`。

### 步骤3：Cloudflare 检测

对于每个有开放端口的 IP，工具发送 HTTP HEAD 请求，检查响应头中是否有 Cloudflare 特征：
- `Server: cloudflare`
- `CF-RAY` / `cf-ray`

如果发现任一特征，则该 IP 被标记为 Cloudflare 反代，保存到 `AS{ASN}_cloudflare_ips.txt`。

## 安装

### 前置要求

- Windows 操作系统
- [masscan](https://github.com/robertdavidgraham/masscan)（将 `masscan.exe` 放在可执行文件同一目录）

### 下载

从 [ Releases 页面 ](https://github.com/henr1232/CloudflareProxyScanner/releases) 下载最新版本。

## 使用方法

1. 运行 `CloudflareProxyScanner.exe`
2. 输入要扫描的 ASN 号码（例如：`40065`）
3. 配置扫描选项：
   - **端口**：逗号分隔的端口列表（默认：`80,443,8080`）
   - **速率**：扫描速度，包/秒（默认：`1000`）
   - **线程数**：检测线程数（默认：`10`）
   - **输出目录**：结果保存位置
4. 选择要执行的步骤：
   - ☑️ 步骤1：获取 ASN IP 段
   - ☑️ 步骤2：masscan 端口扫描
   - ☑️ 步骤3：Cloudflare 检测
5. 点击"开始执行"
6. 等待完成，结果将保存到输出目录

## 输出文件

| 文件 | 说明 |
|------|------|
| `AS{ASN}_ip_ranges.txt` | ASN 下的所有 IP 段 |
| `AS{ASN}_masscan.xml` | masscan 扫描原始结果 |
| `AS{ASN}_open_ips.txt` | 有开放端口的 IP（格式：IP:端口） |
| `AS{ASN}_cloudflare_ips.txt` | 已确认的 Cloudflare 反代 IP |

## 二次验证

扫描完成后，可以在 [proxyip.cmliussss.net](https://check.proxyip.cmliussss.net/) 对Cloudflare反代IP进行额外验证，包括：


### 使用二次验证脚本

```bash
# 验证单个IP
python validate_proxy.py --ip 192.168.1.1 --port 443

# 验证文件中的所有IP
python validate_proxy.py AS40065_cloudflare_ips.txt -o results.json

# 使用20个线程加速验证
python validate_proxy.py AS40065_cloudflare_ips.txt -t 20
```

## 为什么需要这个工具？

Cloudflare Workers 和类似服务会阻止直接向 Cloudflare IP 范围建立出站连接。这个工具可以帮助你找到可以绕过这些限制的第三方反代 IP。

## 项目结构

```
CloudflareProxyScanner/
├── CloudflareProxyScanner.exe  # 主程序
├── _internal/                  # 依赖文件
├── masscan.exe                 # 端口扫描工具
├── main.py                     # 主程序源码
├── validate_proxy.py           # 二次验证脚本
```

## 技术栈

- **Python 3.14+** - 编程语言
- **Tkinter** - GUI 界面
- **Masscan** - 高速端口扫描
- **PyInstaller** - 打包成 exe

## 贡献

欢迎提交 Pull Request！

## 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。

## 致谢

- [masscan](https://github.com/robertdavidgraham/masscan) - 高速 TCP 端口扫描器
- [ipinfo.io](https://ipinfo.io) - IP 地址信息服务
- [whois.ipip.net](https://whois.ipip.net) - IP WHOIS 信息

## 作者

**@henr1232** - [GitHub Profile](https://github.com/henr1232)

## 支持

如果这个项目对你有帮助，请给我一个 ⭐️！

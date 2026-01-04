# AI API Certificate Hash Fetcher (AI API 证书哈希自动获取工具)

这是一个自动化工具，用于定期获取和更新主流 AI 服务（如 OpenAI, 阿里云, Volcengine, Minimax, Google Gemini 等）API 接口的 TLS 证书公钥哈希值。

该项目主要用于配合客户端（如 iOS/Android 应用）实施 **SSL Pinning (证书锁定)** 安全策略，确保应用只信任合法的服务端证书，防止中间人攻击 (MITM)。

## 核心功能

*   **自动监控**: 通过 GitHub Actions 定时（每 6 小时）扫描目标 API。
*   **证书提取**: 建立 TLS 连接并获取服务器的 Leaf（叶子）证书。
*   **哈希计算**: 提取证书公钥并计算 SHA256 哈希值 (Base64 编码)，格式兼容 iOS/Apple 的原生 SSL Pinning 实现。
*   **自动更新**: 如果发现新的证书哈希值（例如服务商更换了证书），会自动更新到 `ValidAICertificatesHash.json` 并提交到仓库。

## 项目结构

*   `leaf_cert_public_fetcher.py`: 核心 Python 脚本。
    *   定义了需要监控的 API URL 列表 (`api_urls`)。
    *   实现了 SSL 连接、证书提取、iOS 兼容的公钥格式化及哈希计算逻辑。
*   `ValidAICertificatesHash.json`: 数据存储文件。
    *   包含一个 `hashs` 列表，存储所有捕获到的有效证书哈希。
*   `.github/workflows/cert_hash_updater.yml`: 自动化工作流配置。
    *   配置为北京时间每日 00:00, 06:00, 12:00, 18:00 自动运行。

## 如何使用

### 1. 数据接入
客户端应用应通过网络请求读取本仓库的 `ValidAICertificatesHash.json` Raw 文件，或在构建时将其打包进应用中，用作 SSL Pinning 的白名单验证依据。

### 2. 本地运行 (开发与调试)
如果你需要手动运行或调试脚本：

1.  环境准备：
    ```bash
    pip install cryptography
    ```
2.  运行脚本：
    ```bash
    python leaf_cert_public_fetcher.py
    ```
    脚本会输出每个域名的检查结果，如果有新哈希，会自动修改本地的 JSON 文件。

### 3. 添加新的 API 监控
如果需要监控新的 AI 服务商：

1.  编辑 `leaf_cert_public_fetcher.py`。
2.  在 `main` 函数中的 `api_urls` 列表里添加新的 API 地址。
    ```python
    api_urls = [
        "https://api.openai.com/v1/chat/completions",
        # 新增地址
        "https://api.new-service.com/v1/..."
    ]
    ```
3.  提交更改。GitHub Actions 会在下一次运行时自动获取新地址的证书哈希。

## GitHub Actions 配置 (关键)

为了让自动化脚本能够成功提交哈希值的更新，你需要在 Fork 或导入此项目后，对 GitHub 仓库的 Actions 权限进行配置：

1.  进入项目仓库的 **Settings** (设置) 页面。
2.  在左侧菜单栏找到 **Actions** -> **General**。
3.  向下滚动到 **Workflow permissions** (工作流权限) 区域。
4.  勾选 **Read and write permissions** (读写权限)。
5.  点击 **Save** (保存)。

> **⚠️ 注意**: 默认情况下 GitHub 可能只授予 Read 权限。如果不开启此权限，GitHub Action 在尝试 `git push` 时会报错 `Permission denied`，导致哈希值无法自动更新。

## 维护说明 (交接必读)

*   **哈希算法**: 脚本中 `get_ios_style_ec_public_key_bytes` 函数专门处理了 Elliptic Curve (EC) 公钥，确保生成的二进制格式与 iOS 系统 API (`SecKeyCopyExternalRepresentation`) 的输出一致。这是为了确保存储的哈希值能直接被 iOS 客户端校验。
*   **触发频率**: 目前设置为每 6 小时一次。如果 API 服务商频繁更换证书（极其罕见），可以调整 `.github/workflows/cert_hash_updater.yml` 中的 cron 表达式。

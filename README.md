# Cloudflare代理IP DNS更新器

本项目自动使用找到的最快代理IP地址更新Cloudflare DNS记录。它被设计为在Ubuntu免费运行器上作为计划任务运行的GitHub Actions工作流。

## 工作原理

1. **发现代理IP（`scripts/getIPs.py`）：** 获取Oracle Cloud公共CIDR范围，连接到每个IP的443端口，TLS SNI设置为`speed.cloudflare.com`，并检查响应是否来自Cloudflare的边缘节点。使用asyncio并行机制无限制地扫描所有IPv4和IPv6范围。结果保存到`result/ips.csv`。

2. **IP测试（`scripts/cfSpeedTest.py`）：** 读取`result/ips.csv`，按区域对IP进行分组，通过真实的TLS套接字连接测试每个代理IP的ping、下载和上传速度。支持IPv4和IPv6。结果保存到`result/tested-ips.csv`。

3. **域名IP映射（`scripts/mapDomain.py`）：** 将`result/ips.csv`（区域查询）和`result/tested-ips.csv`（速度数据）按IP进行关联，将表现最佳的IP按区域映射到域名，按下载速度排序。结果保存到`result/domains-ips.csv`。

4. **Cloudflare记录更新（`scripts/cfRecUpdate.py`）：** 读取`result/domains-ips.csv`，使用IP地址更新Cloudflare DNS记录。自动检测IP版本——为IPv4创建A记录，为IPv6创建AAAA记录。原地更新现有记录，创建新记录，并删除多余的记录。

5. **工作流自动化：** 一个GitHub Actions工作流（`daily_update.yml`）每十二小时调度运行整个过程。

## GitHub设置

1. **仓库：** 将此仓库克隆/复刻到您的GitHub账户。

2. **编辑配置：** 编辑`config.ini`为您的所需配置。

3. **工作流配置：**
   - 在您仓库的设置中（Settings > Secrets and variables > Actions > Secrets），添加一个名为`CLOUDFLARE_API_TOKEN`的密钥，值为您的Cloudflare API令牌。

4. **工作流手动触发（可选）：** 如果需要，您可以从仓库的"Actions"选项卡手动触发工作流。

## 本地设置

1. **仓库：** 使用git克隆此仓库。

2. **编辑配置：** 编辑`config.ini`为您的所需配置。

3. **设置环境变量：**
   - 设置名为`CLOUDFLARE_API_TOKEN`的环境变量，值为您的Cloudflare API令牌。

4. **运行：**
   - 获取代理IP，运行`python "scripts/getIPs.py"`
   - 测试代理IP，运行`python "scripts/cfSpeedTest.py"`
   - 将IP映射到域名，运行`python "scripts/mapDomain.py"`
   - 最后，更新Cloudflare记录，运行`python "scripts/cfRecUpdate.py"`

## 配置指南

### 1. **获取IP**
- **用途：** 通过扫描所有公共CIDR范围，在Oracle Cloud基础设施中发现Cloudflare代理IP。
- **设置：**
  - `ranges_url`：Oracle Cloud公共IP范围JSON URL（默认：`https://docs.oracle.com/en-us/iaas/tools/public_ip_ranges.json`）。
  - `region_url`：Cloudflare colo到区域映射CSV URL（默认：Netrvin/cloudflare-colo-list）。
  - `timeout`：代理扫描的每次连接超时时间，单位秒（默认：`1`）。
  - `max_concurrent`：最大并发asyncio任务数（默认：`2000`）。
  - `fetch_cidrs_timeout`：获取Oracle CIDR的HTTP超时时间，单位秒（默认：`30`）。
  - `fetch_colo_timeout`：获取colo CSV的HTTP超时时间，单位秒（默认：`4`）。
  - `output_file`：保存发现的代理IP的路径（默认：`result/ips.csv`）。

### 2. **Cloudflare速度测试（cfSpeedTest）**
- **用途：** 测试IP的ping、下载和上传性能的速度和质量。
- **设置：**
  - `file_ips`：包含发现的IP和区域的输入CSV（默认：`result/ips.csv`）。
  - `max_ips`：每个区域最多测试的IP数量（默认：`24`）。
  - `max_ping`：最大可接受ping值，单位ms（默认：`896`）。
  - `min_download_speed`：最小可接受下载速度，单位Mbps（默认：`20.0`）。
  - `min_upload_speed`：最小可接受上传速度，单位Mbps（默认：`20.0`）。
  - `test_size`：测试下载/上传速度的数据大小，单位KB（默认：`10240`）。
  - `timeout`：所有速度测试操作的每次连接超时时间，单位秒（默认：`4`）。
  - `ping_workers`：并行ping测试的线程池大小（默认：`20`）。
  - `output_file`：保存测试结果的文件（默认：`result/tested-ips.csv`）。

### 3. **映射域名**
- **用途：** 将已测试的IP分配给特定的区域和域名。
- **设置：**
  - `file_ips`：IP到区域映射的CSV（默认：`result/ips.csv`）。
  - `file_tests`：已测试IP性能数据的CSV（默认：`result/tested-ips.csv`）。
  - `output_file`：域名到IP映射的输出CSV（默认：`result/domains-ips.csv`）。
- **映射规则（`[mapDomain.map]`）：**
  - 每行将一个区域映射到一个域名和最大IP数量。
  - 格式：`{区域} = {域名},{最大IP数量}` 例如：
    - `Europe = proxy.farel.is-a.dev,5`
    - `Asia_Pacific = proxy.farel.is-a.dev,10`

### 4. **Cloudflare记录更新（cfRecUpdate）**
- **用途：** 根据映射的域名和IP更新Cloudflare DNS记录。自动为IPv4创建A记录，为IPv6创建AAAA记录。
- **设置：**
  - `api_url`：Cloudflare API基础URL（默认：`https://api.cloudflare.com/client/v4`）。
  - `zone_id`：用于更新的Cloudflare区域ID。
  - `file_domains`：包含域名及其对应IP的CSV（默认：`result/domains-ips.csv`）。

每个部分对应流程中的特定步骤，允许模块化使用和配置。每个配置键都有合理的默认值——只有`zone_id`和`CLOUDFLARE_API_TOKEN`环境变量是必需的。

## 许可证

本项目采用GNU通用公共许可证v3.0授权——详见[LICENSE](LICENSE)文件。

## 免责声明

本项目按现状提供。使用风险自负。请确保您理解其工作原理，并根据您的具体需求正确配置。作者不对因使用本项目而产生的任何问题或损害负责。

## 贡献

欢迎贡献！请随时提出issue或提交pull request。

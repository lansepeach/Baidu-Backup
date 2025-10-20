# Baidu-Backup：高性能百度网盘自动备份与轮替解决方案

Baidu-Backup 是一个功能强大、经过实战考验的自动化备份方案。它通过高度优化的 Shell 脚本与 Python 引擎配合，将您的服务器数据以高性能并行方式备份到云端，并智能管理历史备份。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 为什么选择 Baidu-Backup？

- 极致性能，并行上传：分卷并行上传，充分利用带宽，显著缩短备份时间。
- 流式处理，零中间大文件：tar 与 split 管道化，不占用额外磁盘空间。
- 智能轮替，空间无忧：自动仅保留最新 N 个备份“组”。
- 无人值守，长达十年：一次授权，自动刷新 Token，长期稳定运行。
- 数据校验，可靠可信：双重 MD5 校验，服务器端最终校验。
- 配置清晰，部署简单：配置集中在 start_backup.sh，Python 引擎免改动。
- 日志清晰，易于排错：并行模式下依然清晰，适合自动化任务。

## 支持与规划

- 已支持后端：百度网盘（Baidu Netdisk）
- 规划/预备：123pan（123 云盘）。本文档包含 123pan 凭证准备指南，便于后续切换或扩展。

## 架构概览

- start_backup.sh（总指挥与配置）
  - 本地打包（流式）→ 分卷 → 并行调度上传引擎
  - 您主要修改的文件
- Baidu-Backup.py（上传引擎）
  - 云端操作：鉴权、分片上传、合并、备份轮替
  - 读取环境变量 BAIDU_APP_KEY、BAIDU_SECRET_KEY

## 快速上手

### 第 1 步：准备云盘应用凭证

- 百度网盘（已支持）
  1. 访问 https://pan.baidu.com/union/console 并登录。
  2. 完成开发者认证，创建“个人应用-存储”类型应用。
  3. 记录 AppKey 与 SecretKey。
  4. 运行时以环境变量提供：
     - BAIDU_APP_KEY
     - BAIDU_SECRET_KEY

- 123pan（规划/预备）
  1. 访问 123pan 开放平台（请根据官方文档与入口操作）。
  2. 注册开发者并创建应用，获取 Client ID/Secret。
  3. 建议预留环境变量（当前版本未使用）：
     - PAN123_CLIENT_ID
     - PAN123_CLIENT_SECRET

### 第 2 步：部署项目

1. 克隆仓库
   git clone https://github.com/lansepeach/Baidu-Backup.git
   cd Baidu-Backup

2. 准备 SDK（按百度官方指引）
   从百度开放平台下载 Python SDK（openapi_client），解压后将 openapi_client 文件夹复制到 Baidu-Backup 目录。

3. 安装依赖
   pip install -r requirements.txt

### 第 3 步：配置 start_backup.sh

打开 start_backup.sh 并按注释修改：

- 必填环境变量（用于 Python 引擎鉴权）
  - export BAIDU_APP_KEY="你的AppKey"
  - export BAIDU_SECRET_KEY="你的SecretKey"
- 关键参数
  - LIVE_DATA_DIR：需要备份的本地目录（例如 /opt）
  - REMOTE_DIR：云端备份目录（百度需以 /apps/ 开头）
  - MAX_BACKUPS：仅保留最近 N 个备份组（0 表示不清理）
  - SPLIT_SIZE：分卷大小（推荐 1G）
  - PARALLEL_UPLOADS：并行上传任务数（建议 2~3 起步）
  - PYTHON_EXECUTABLE：Python 解释器路径
  - BACKUP_SCRIPT_PATH：Baidu-Backup.py 的绝对路径

示例配置可参考仓库根目录的 config.example.yaml（用于映射与记录，不会被程序直接读取）。

### 第 4 步：首次授权与运行

1. 赋予执行权限
   chmod +x start_backup.sh

2. 首次运行（交互式授权）
   ./start_backup.sh
   根据提示在浏览器打开授权链接，完成登录授权后粘贴 code。成功后会生成 baidu_token.json，后续将自动刷新。

3. 后续运行（全自动）
   再次执行 ./start_backup.sh 即可静默运行。

### 第 5 步：自动化运行

- 使用 cron
  参考 scripts/cron_example.txt，将任务设为每日 03:00 执行并写日志。

- 使用 systemd（推荐）
  1) 安装示例单元
     sudo cp scripts/systemd/baidu-backup.service /etc/systemd/system/
     sudo cp scripts/systemd/baidu-backup.timer /etc/systemd/system/
  
  2) 可选：环境文件（存放密钥等）
     sudo cp scripts/systemd/baidu-backup.env.example /etc/default/baidu-backup
     sudo chmod 600 /etc/default/baidu-backup
  
  3) 启用并启动定时器
     sudo systemctl daemon-reload
     sudo systemctl enable --now baidu-backup.timer
  
  4) 查看状态与日志
     systemctl status baidu-backup.service
     journalctl -u baidu-backup.service --no-pager -n 200

## 配置指南（要点与环境变量映射）

- BAIDU_APP_KEY / BAIDU_SECRET_KEY：百度开放平台凭证（必需）
- LIVE_DATA_DIR：备份源目录（绝对路径）
- REMOTE_DIR：云端备份目录；百度需以 /apps/<应用名> 开头
- MAX_BACKUPS：仅保留最近 N 个备份“组”；按组删除所有分卷
- SPLIT_SIZE：分卷大小，单位如 1G、512M
- PARALLEL_UPLOADS：并行上传任务数（xargs -P）
- PYTHON_EXECUTABLE：Python 解释器路径
- BACKUP_SCRIPT_PATH：Baidu-Backup.py 的绝对路径

说明：当前版本程序不直接读取 YAML；config.example.yaml 仅作为“配置清单与环境映射”示例，便于生成 /etc/default/baidu-backup 或对照 start_backup.sh。

## 加密备份（可选）

出于安全考虑，您可以在管道中插入对称加密步骤。推荐两种方式：

- OpenSSL（AES-256-CTR）
  将 start_backup.sh 中 tar → split 的管道替换为：
  tar --ignore-failed-read -czf - -C "$(dirname "$LIVE_DATA_DIR")" "$(basename "$LIVE_DATA_DIR")" \
  | openssl enc -aes-256-ctr -pbkdf2 -salt -pass env:BACKUP_PASSWORD \
  | split -b "$SPLIT_SIZE" -d - "$SPLIT_DIR/$TAR_FILENAME_BASE."

  恢复解密：
  cat your_backup.tar.gz.* | openssl enc -d -aes-256-ctr -pbkdf2 -pass env:BACKUP_PASSWORD | tar -xz -C /restore/path

- GPG（对称加密）
  tar --ignore-failed-read -czf - -C "$(dirname "$LIVE_DATA_DIR")" "$(basename "$LIVE_DATA_DIR")" \
  | gpg --batch --yes --symmetric --cipher-algo AES256 --passphrase "$BACKUP_PASSWORD" \
  | split -b "$SPLIT_SIZE" -d - "$SPLIT_DIR/$TAR_FILENAME_BASE."

  恢复解密：
  cat your_backup.tar.gz.* | gpg --batch --yes --decrypt --passphrase "$BACKUP_PASSWORD" | tar -xz -C /restore/path

请将 BACKUP_PASSWORD 放入安全的环境文件（如 /etc/default/baidu-backup），并限制权限（600）。

## 监控与退出码

- 成功：start_backup.sh 最终返回 0，Python 引擎返回 0
- 失败：任何阶段出现致命错误返回非 0（通常为 1）
- systemd 监控：
  - Restart=on-failure（如需）
  - journalctl -u baidu-backup.service 查看日志
- cron 监控：
  - 将输出重定向到日志文件并由外部监控采集
  - 或使用 MAILTO 接收失败通知

## 常见问题排查（Troubleshooting）

- 首次授权失败 / 无法获取 token
  - 确认已设置 BAIDU_APP_KEY/BAIDU_SECRET_KEY
  - 按提示在浏览器打开授权 URL，粘贴 code
  - 删除损坏的 baidu_token.json 后重试

- ImportError: 无法导入 openapi_client
  - 请按 README 指引下载 SDK 并将 openapi_client 目录置于项目根目录

- 远程目录无效
  - 对于百度，REMOTE_DIR 必须以 /apps/ 开头

- 并行上传失败或偶发网络错误
  - 程序会自动重试分片；可适当降低 PARALLEL_UPLOADS

- 本地文件不一致错误（双 MD5 校验未通过）
  - 可能为底层存储或文件被改写，请先确保源目录稳定

- Streaming process failed. No volumes were created.
  - 检查 LIVE_DATA_DIR 是否存在、权限足够、磁盘空间是否充足

## 恢复备份

1. 下载同一批次的所有分卷（例如 .00、.01、.02 ...）到同一目录
2. 合并并解压：
   cat your_backup_timestamp.tar.gz.* > complete_backup.tar.gz
   tar -xzf complete_backup.tar.gz

若启用了加密，请先按上文“加密备份（可选）”中的解密命令合并解密，再解压。

## 发布说明

请查看 CHANGELOG.md 获取 v0.1.0 的发布摘要。

## 贡献

欢迎提交 Issues 或 Pull Requests。如果这个项目对您有帮助，欢迎 Star ⭐！

## 许可证

本项目采用 MIT License 授权。

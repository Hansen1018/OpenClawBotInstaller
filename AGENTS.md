# AGENTS.md - OpenClaw 安装工具项目指南

## 项目概述

**项目类型**: Bash Shell 安装配置工具
**主要语言**: Bash
**目标**: 自动化部署和配置 OpenClaw AI 助手

## 构建和测试命令

```bash
# 无传统构建命令，项目使用纯 Shell 脚本

# 验证脚本语法
bash -n install.sh
bash -n config-menu.sh

# 运行安装脚本（测试模式）
# 脚本支持 dry-run 或交互模式，需查看脚本具体实现

# 检查 Shell 兼容性
shellcheck install.sh config-menu.sh  # 如果安装了 shellcheck

# 测试 Docker 构建
docker build -t openclaw-installer .
docker-compose up --build

# 本地测试安装脚本
chmod +x install.sh
./install.sh

# 本地测试配置菜单
chmod +x config-menu.sh
./config-menu.sh
```

**注意**: 项目没有自动化测试框架（如 pytest, npm test 等）。测试主要通过：
1. 手动运行脚本验证功能
2. Shell 语法检查 (`bash -n`, `shellcheck`)
3. Docker 镜像构建测试

---

## 代码风格指南

### 1. 文件命名

- **Shell 脚本**: 小写 + 连字符或下划线（推荐小写 + `.sh` 扩展名）
  - 正例: `install.sh`, `config-menu.sh`, `docker-entrypoint.sh`
  - 反例: `Install.sh`, `configMenu.sh`

- **配置文件**: 
  - 环境变量: `env`
  - JSON 配置: `openclaw.json`
  - YAML 示例: `config.example.yaml`

### 2. 导入和模块

- Shell 脚本使用 `source` 或 `.` 导入其他脚本
  ```bash
  source ~/.openclaw/env
  . /path/to/other-script.sh
  ```

### 3. 命名约定

**变量命名**: 全大写 + 下划线（蛇形命名）
```bash
OPENCLAW_VERSION="latest"
CONFIG_DIR="$HOME/.openclaw"
BACKUP_DIR="$CONFIG_DIR/backups"
```

**函数命名**: 小写 + 下划线（蛇形命名）
```bash
check_dependencies()
print_banner()
backup_config()
restart_gateway_for_channel()
```

**局部变量**: 使用 `local` 关键字
```bash
local prompt="$1"
local var_name="$2"
local response=${response:-$default}
```

### 4. 代码格式

- **缩进**: 4 个空格（无 tab）
- **行宽**: 建议不超过 100 字符（长命令可用反斜杠换行）
- **函数定义**: 使用 `()` 花括号
  ```bash
  function_name() {
      local var="$1"
      # 函数体
  }
  ```

- **条件语句**: 使用 `if/then/fi`
  ```bash
  if [ -t 0 ]; then
      TTY_INPUT="/dev/stdin"
  else
      TTY_INPUT="/dev/tty"
  fi
  ```

### 5. 错误处理

**全局错误处理**:
```bash
set -e  # 遇到错误立即退出
set -u  # 使用未定义变量时报错
set -o pipefail  # 管道中任何命令失败都返回错误
```

**命令执行检查**:
```bash
# 使用 || true 忽略失败
openclaw gateway stop 2>/dev/null || true

# 检查命令是否存在
if ! command -v yq &> /dev/null; then
    USE_YQ=false
else
    USE_YQ=true
fi
```

**错误日志**:
```bash
log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 使用示例
if [ ! -f "$OPENCLAW_ENV" ]; then
    log_error "配置文件不存在: $OPENCLAW_ENV"
    exit 1
fi
```

### 6. 用户交互

**TTY 检测**（支持 `curl | bash` 模式）:
```bash
if [ -t 0 ]; then
    TTY_INPUT="/dev/stdin"
else
    TTY_INPUT="/dev/tty"
fi

# 读取输入
read_input() {
    local prompt="$1"
    local var_name="$2"
    echo -en "$prompt"
    read $var_name < "$TTY_INPUT"
}
```

**确认对话框**:
```bash
confirm() {
    local message="$1"
    local default="${2:-y}"
    local prompt="[Y/n]"
    echo -en "${YELLOW}$message $prompt: ${NC}"
    read response < "$TTY_INPUT"
    response=${response:-$default}
    case "$response" in
        [yY][eE][sS]|[yY]) return 0 ;;
        *) return 1 ;;
    esac
}
```

### 7. 日志和输出

**颜色定义**:
```bash
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
GRAY='\033[0;90m'
NC='\033[0m' # 无颜色
```

**日志函数**:
```bash
log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1"; }
log_step() { echo -e "${BLUE}[STEP]${NC} $1"; }
```

**横幅输出**:
```bash
print_header() {
    echo -e "${CYAN}"
    cat << 'EOF'
    ╔═══════════════════════════════════════════════════════════════╗
    ║   🦞 OpenClaw 配置中心                                         ║
    ╚═══════════════════════════════════════════════════════════════╝
EOF
    echo -e "${NC}"
}
```

### 8. 配置管理

**环境变量文件格式** (`~/.openclaw/env`):
```bash
# OpenClaw 环境变量配置
export ANTHROPIC_API_KEY=sk-ant-xxxxx
export ANTHROPIC_BASE_URL=https://your-api-proxy.com

# 或 OpenAI
export OPENAI_API_KEY=sk-xxxxx
export TELEGRAM_BOT_TOKEN=xxx
```

**读取环境变量**:
```bash
get_env_value() {
    local key=$1
    if [ -f "$OPENCLAW_ENV" ]; then
        grep "^export $key=" "$OPENCLAW_ENV" 2>/dev/null | sed 's/.*=//' | tr -d '"'
    fi
}

# 加载环境变量
source "$OPENCLAW_ENV"
```

**备份配置**:
```bash
backup_config() {
    mkdir -p "$BACKUP_DIR"
    local backup_file="$BACKUP_DIR/env_$(date +%Y%m%d_%H%M%S).bak"
    if [ -f "$OPENCLAW_ENV" ]; then
        cp "$OPENCLAW_ENV" "$backup_file"
        echo "$backup_file"
    fi
}
```

### 9. 进程管理

**后台启动服务**:
```bash
# 使用 setsid 创建新会话（完全脱离终端）
if command -v setsid &> /dev/null; then
    setsid bash -c "source $OPENCLAW_ENV && exec openclaw gateway" > /tmp/openclaw-gateway.log 2>&1 &
else
    # 备用方案: nohup + disown
    nohup bash -c "source $OPENCLAW_ENV && exec openclaw gateway" > /tmp/openclaw-gateway.log 2>&1 &
    disown 2>/dev/null || true
fi
```

**停止服务**:
```bash
openclaw gateway stop 2>/dev/null || true
pkill -f "openclaw.*gateway" 2>/dev/null || true
```

### 10. Docker 相关

**Dockerfile 最佳实践**:
- 使用轻量级基础镜像
- 复制文件前使用 WORKDIR
- 设置正确的文件权限
- 使用 ENTRYPOINT 而非 CMD（便于接收参数）

**docker-compose.yml**:
- 使用版本 3.8+
- 定义健康检查
- 挂载配置目录
- 设置环境变量

### 11. 注释和文档

**文件头注释**:
```bash
#!/bin/bash
#
# ╔═══════════════════════════════════════════════════════════════╗
# ║   🦞 OpenClaw 一键部署脚本 v1.0.0                            ║
# ║   智能 AI 助手部署工具 - 支持多平台多模型                      ║
# ╚═══════════════════════════════════════════════════════════════╝
#
# 使用方法:
#   curl -fsSL <url> | bash
#   或本地执行: chmod +x install.sh && ./install.sh
#
```

**函数注释**:
```bash
# ================================ 工具函数 ================================

# 检查依赖工具是否可用
check_dependencies() {
    # 实现细节
}
```

---

## 项目目录结构

```
OpenClawBotInstaller/
├── install.sh           # 主安装脚本
├── config-menu.sh        # 交互式配置菜单
├── docker-entrypoint.sh  # Docker 入口脚本
├── Dockerfile            # Docker 镜像定义
├── docker-compose.yml    # Docker Compose 配置
├── README.md             # 项目文档
├── .gitignore            # Git 忽略规则
├── .gitattributes        # Git 属性配置（LF 换行）
├── examples/             # 示例配置
│   ├── config.example.yaml
│   └── skills/           # 技能示例
└── photo/                # 文档图片
```

**安装后目录** (`~/.openclaw/`):
```
~/.openclaw/
├── env                   # 环境变量文件
├── openclaw.json         # OpenClaw 配置
├── config-menu.sh        # 配置菜单脚本
├── backups/              # 配置备份
└── logs/                 # 日志文件
```

---

## 常用命令参考

**服务管理**:
```bash
openclaw gateway start    # 后台启动
openclaw gateway stop     # 停止服务
openclaw gateway restart  # 重启服务
openclaw gateway status   # 查看状态
openclaw logs             # 查看日志
```

**配置管理**:
```bash
openclaw config           # 打开配置文件
openclaw doctor           # 诊断配置问题
openclaw health           # 健康检查
```

**数据管理**:
```bash
openclaw export --format json
openclaw memory clear
openclaw backup
```

---

## 安全注意事项

1. **API Key 安全**: 永远不要将 `~/.openclaw/env` 提交到版本控制
2. **输入验证**: 对所有用户输入进行验证
3. **权限控制**: 配置中限制允许的用户 ID
4. **沙箱模式**: 生产环境建议启用沙箱模式
5. **禁止危险命令**: 默认禁用 shell 命令和文件访问功能

---

## 其他说明

- 项目使用 UTF-8 编码
- 换行符格式：LF（通过 `.gitattributes` 配置）
- 脚本兼容 Bash 4.0+（部分功能可能需要更高版本）
- 主要测试平台：macOS 12+, Ubuntu 20.04+, Debian 11+

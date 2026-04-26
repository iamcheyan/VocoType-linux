# Fedora 43 + GNOME + Wayland 安装指南

本文档记录 VoCoType IBus 完整版（语音 + Rime 拼音）在 Fedora 43 Workstation (GNOME/Wayland) 上的完整安装流程。

## 环境信息

- **OS**: Fedora Linux 43 (Workstation Edition)
- **桌面**: GNOME + Wayland
- **Python**: 3.14.3（系统默认，但 onnxruntime 不支持）
- **输入法框架**: IBus（GNOME 默认）

## 前置条件

系统已安装：
- `ibus` — GNOME 默认自带
- `portaudio` — 音频输入
- `gcc-c++` — 编译依赖

## 1. 安装系统依赖

```bash
sudo dnf install -y \
  ibus-rime \
  librime-devel \
  cairo-devel \
  cairo-gobject-devel \
  gobject-introspection-devel \
  pkgconfig
```

> **注意**: `cairo-gobject-devel` 容易被遗漏，但它是编译 `pygobject` 必需的。

## 2. 安装 uv 并创建 Python 3.12 环境

Fedora 43 默认 Python 3.14 不支持 onnxruntime，需用 uv 管理 Python 3.12：

```bash
# 安装 uv（如未安装）
pip install uv

# 下载 Python 3.12 并创建虚拟环境
uv python install 3.12
uv venv --python 3.12 .venv
```

## 3. 安装 Python 依赖

```bash
# 核心依赖
uv pip install -r requirements.txt

# Rime Python 绑定（完整版需要）
uv pip install pyrime

# SLM 本地模型依赖（Shift+F9 长句润色）
uv pip install torch transformers sentencepiece socksio
```

## 4. 配置音频设备

创建音频配置文件（跳过交互式向导）：

```bash
mkdir -p ~/.config/vocotype
cat > ~/.config/vocotype/audio.conf << 'EOF'
[audio]
device_name = default
sample_rate = 44100
EOF
```

> `device_name` 比 `device_id` 更稳定，设备编号变化不会导致配置失效。

## 5. 运行安装脚本

使用项目虚拟环境（选项 1），完整版（选项 2），启用 SLM 本地模型（选项 2→1）：

```bash
printf '2\n2\n1\n\nY\n1\n0\n' | bash scripts/install-ibus.sh \
  --device default --sample-rate 44100
```

选项含义：
- `2` → 完整版（语音 + Rime 拼音）
- `2` → 启用 SLM 润色
- `1` → 本地一次性加载（推荐）
- `\n` → 使用默认模型 `Qwen/Qwen3.5-0.8B`
- `Y` → 安装 torch/transformers 等 SLM 依赖
- `1` → 使用项目虚拟环境 `.venv`
- `0` → 使用默认 `luna_pinyin` 方案

## 6. 修复 IBus 组件路径（GNOME 必需）

GNOME 要求 IBus 组件 XML 在系统目录：

```bash
sudo cp ~/.local/share/ibus/component/vocotype.xml \
  /usr/share/ibus/component/
```

安装脚本已自动处理此步骤，如未成功可手动执行。

## 7. 重启 IBus

```bash
ibus restart
```

## 8. 添加输入法

1. 打开 **设置 → 键盘 → 输入源**
2. 点击 `+` 添加输入源
3. 滑到最底下点三个点 (⋮)
4. 搜索 `voco` → 中文 → **VoCoType Voice Input**

## 使用方法

| 快捷键 | 功能 |
|--------|------|
| `F9` | 极速语音输入（仅 ASR） |
| `Shift+F9` | 长句模式（ASR + SLM 润色） |
| `Ctrl+F9` | 语音编辑模式（读取 surrounding text） |
| 直接打字 | Rime 拼音输入 |

## 配置文件

| 文件 | 说明 |
|------|------|
| `~/.config/vocotype/audio.conf` | 音频设备配置 |
| `~/.config/vocotype/ibus.json` | SLM / 润色配置 |
| `~/.config/vocotype/rime/user.yaml` | Rime 输入方案 |
| `~/.config/ibus/rime/` | Rime 用户配置目录 |
| `~/.local/share/vocotype/ibus.log` | 运行日志 |

## 常见问题

### Python 版本不兼容

Fedora 43 默认 Python 3.14，但 onnxruntime 仅支持 3.11-3.12。必须使用 uv 或 pyenv 安装 3.12。

### `pygobject` 编译失败

错误：`cairo-gobject.h: 没有那个文件`

解决：
```bash
sudo dnf install cairo-gobject-devel
```

### 输入法列表中找不到 VoCoType

GNOME 要求组件 XML 在 `/usr/share/ibus/component/`。安装脚本已自动复制，如失败请手动执行：

```bash
sudo cp ~/.local/share/ibus/component/vocotype.xml /usr/share/ibus/component/
ibus restart
```

### 首次使用下载模型

- FunASR 模型：约 500MB，首次语音输入时自动下载
- SLM 模型（Qwen3.5-0.8B）：约 1.5GB，首次 `Shift+F9` 时自动下载

## 卸载

```bash
./scripts/uninstall-ibus.sh
```

选择：
- **快速卸载**：保留 `.venv` 和模型，方便重装
- **完全卸载**：删除所有内容

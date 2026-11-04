# 说明

- MAVLink 仓库保存的是 XML 协议定义和 `pymavlink` 代码生成器，是协议的源码仓库。
- `c_library_v2` 保存由这些 XML 生成的 MAVLink 2 C 头文件，供QGC直接引用。
- QGC 通过 CPM 拉取 `c_library_v2` 的指定提交，并使用其中的 `all` dialect 编译 MAVLink 通信代码。

## 拉取 MAVLink 和 c_library_v2 库
### 1. 拉取 MAVLink 协议库

克隆仓库：

```bash
export MAVLINK_DIR="/path/to/mavlink"
git clone --recurse-submodules https://github.com/Glados07/mavlink.git "$MAVLINK_DIR"
```

拉取完成后，执行以下命令 fetch分支到本地：

```sh
# feature功能开发分支
git fetch SecDev/feature
git checkout SecDev/feature
```

初始化子模块：

```bash
git -C "$MAVLINK_DIR" submodule update --init --recursive
```

后续开发：QGC 使用 `all` dialect，新增独立 dialect 时，将它加入 `message_definitions/v1.0/all.xml`。

### 2. 拉取 c_library_v2 头文件仓库

克隆头文件仓库：

```bash
export CLIB_DIR="/path/to/c_library_v2"
git clone https://github.com/Glados07/c_library_v2.git "$CLIB_DIR"
```

拉取完成后，执行以下命令 fetch分支到本地：

```sh
# feature功能开发分支
git fetch SecDev/feature
git checkout SecDev/feature
```

### 3. 开发约定：添加新功能

从`SecDev/feature`分支创建新分支开发，命名为 `SecDev/ft/<new_feature>`。比如开发新功能'3D_map'，执行

```sh
git switch SecDev/feature
git checkout -b SecDev/ft/3D_map
```

当这一子功能测试完毕后，merge到`SecDev/feature`分支

---

## 新开发 MAVLink 协议到 QGC 头文件库的同步流程

**概述**：

```text
MAVLink XML ──pymavlink/mavgen──> c_library_v2 C 头文件 ──CPM 固定提交──> QGC
```

### 1. 准备MAVLink生成环境

在**MAVLink仓库**：

```bash
python3 -m venv "$MAVLINK_DIR/build-venv"
MAVLINK_PYTHON="$MAVLINK_DIR/build-venv/bin/python"
"$MAVLINK_PYTHON" -m pip install -r "$MAVLINK_DIR/pymavlink/requirements.txt"
```

### 2. 生成 MAVLink 2 C 头文件

找一个临时目标目录存取mavlink的生成文件：

```bash
export TMPDIR="path/to/temporaryFolder"
```

mavgen 生成 dialect 的头文件。生成完成后，保存本次协议 XML：

```bash
MAVLINK_OUT="$(mktemp -d "${TMPDIR:-/tmp}/mavlink-v2.XXXXXX")"

(
	set -e
    cd "$MAVLINK_DIR"

    for dialect in \
        all test storm32 ardupilotmega csAirLink uAvionix \
        cubepilot ualberta icarous loweheiser paparazzi \
        ASLUAV AVSSUAS matrixpilot python_array_test \
        development fuel_cell marsh common standard minimal
    do
        "$MAVLINK_PYTHON" pymavlink/tools/mavgen.py \
            --lang=C \
            --wire-protocol=2.0 \
            --output="$MAVLINK_OUT" \
            "message_definitions/v1.0/$dialect.xml"
    done

    mkdir -p "$MAVLINK_OUT/message_definitions"
    cp message_definitions/v1.0/*.xml "$MAVLINK_OUT/message_definitions/"
)
```

将临时目录下的生成结果同步到 `c_library_v2`仓库：

```bash
rsync -avi --delete \
    --exclude='/.git' --exclude='/.github/' \
    --include='*/' --include='*.h' --include='/message_definitions/*.xml' \
    --exclude='*' "$MAVLINK_OUT/" "$CLIB_DIR/"
```

### 3. 发布头文件版本到远程仓库

- 将更新后的头文件库提交git
- 发布到远程仓库
- 发布后，执行以下命令取新版本的hash版本号

```bash
git -C "$CLIB_DIR" rev-parse HEAD"
```

此版本号就是 QGC 应引用的头文件版本hash值。

### 4. 更新 QGC 并构建

在 `qgroundcontrol/custom/cmake/CustomOverrides.cmake` 中更新头文件仓库及版本：

```cmake
set(QGC_MAVLINK_GIT_REPO
    "https://github.com/Glados07/c_library_v2.git"
    CACHE STRING "URL to MAVLink Git Repo" FORCE
)

set(QGC_MAVLINK_GIT_TAG
    "<这里粘贴版本号hash值>"
    CACHE STRING "Tag of MAVLink Git Repo" FORCE
)
```

最后重新Run CMake，然后构建实际使用的 QGC Kit即可。

---

## c_library_v2 提交commit格式约定

```md
由mavlink库 <版本号> 版本自动生成
- <提交描述>
```

**示例**：

```md
由mavlink库 33af200d25ec6f0925b49b1ba82bbf1294ea5f72 版本自动生成
- 修改燃料电池状态协议
```

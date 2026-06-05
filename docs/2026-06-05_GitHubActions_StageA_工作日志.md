# GitHub Actions CI 演进记录 (Stage A)

> 接续 [`2026-06-05_GitHubActions构建wheel.md`](./2026-06-05_GitHubActions构建wheel.md)（早期 self-hosted ARM64 思路），本次把 CI 推进到 GitHub-hosted runner + 多 Python × 多 QNN matrix。
>
> 工作分支：`ci-stage-a`，已推送到个人 fork `https://github.com/shijunz/ai-engine-direct-helper/tree/ci-stage-a`，**未合入** `quic/ai-engine-direct-helper:main`。

---

## 1. 目标与最终形态

每次 push / PR 自动跑 lint + 多组合 wheel build + smoke test，全部跑在 **GitHub-hosted** runner 上（无 self-hosted 依赖）。

最终矩阵：**4 Python × 2 QNN SDK × 2 OS = 16 个 build job**。

| 维度 | 取值 |
|---|---|
| Python | 3.10 / 3.11 / 3.12 / 3.13 |
| QNN SDK | 2.46.0.260424 / 2.47.0.260601（Community） |
| OS | windows-latest (x64, ARM64EC binaries) / windows-11-arm (native ARM64) |

x64 job 只 build 不 test（ARM64EC 二进制不能在 x86-64 硬件上 import）；ARM64 job 在同一 runner 上 native build + smoke test。

---

## 2. 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| [`.github/workflows/ci.yml`](../.github/workflows/ci.yml) | 新增 | 主 workflow，3 个 job：lint + build-windows-x64 + build-windows-arm64 |
| [`.github/actions/setup-qnn-sdk/action.yml`](../.github/actions/setup-qnn-sdk/action.yml) | 新增 | Composite action：下载 + actions/cache 缓存 QNN Community SDK，自动探测解压结构，导出 `QNN_SDK_ROOT`；支持 `version` + `url` 两个 input |
| [`.gitignore`](../.gitignore) | 修改 | 补 `build/`, `lib/`, `*.egg-info/`, `*.whl`, `__pycache__/`, `*.pyc` |
| [`tests/test_package_smoke.py`](../tests/test_package_smoke.py) | 一并 commit | 仓库里已有但 untracked，3 个 smoke test：import + 公开符号 + 常量值 |
| [`.github/workflows/build-wheel.yml`](../.github/workflows/build-wheel.yml) | **未动** | 早期 self-hosted ARM64 workflow，按用户要求保留 |

---

## 3. Commit 链（在 `ci-stage-a` 上，基于 `origin/main`）

| commit | 内容 |
|---|---|
| `34fceb8` | 初版：lint + windows-x64 build + smoke test + setup-qnn-sdk composite + .gitignore |
| `36c665f` | 修 ARM64EC 在 x64 硬件上 access violation：x64 job 不再 smoke test，新增 windows-11-arm 原生 ARM64 job |
| `c45c1c2` | smoke test 前 `pip install numpy` |
| `f46f561` | 同上加 `pyyaml`（一次性 grep 全文 import 后补齐） |
| `f3b7047` | lint job 改 `continue-on-error: true`，存量 25 个 ruff 警告不阻塞构建 |
| `1da6b4b` | 4 Python × 2 QNN matrix 展开，artifact 命名带版本 |
| `(next)` | 加 `tests/test_cpu_runtime_smoke.py` ——「Stage B-lite」深度 CPU smoke：验证 pybind 层可调 + 绑定的 `QnnCpu.dll`/`QnnSystem.dll` 在 wheel 里可发现 |

---

## 4. 走过的坑（按时间序）

### 4.1 ARM64EC 二进制在 x64 hosted runner 上 access violation

**症状**：
```
Windows fatal exception: access violation
File ".../qai_appbuilder/qnncontext.py", line 13 in <module>
```

**根因**：[`setup.py:_detect_arch()`](../setup.py#L120-L136) 在 AMD64 主机上选 `ARM64EC`（专给 X Elite 上跑 x64 Python 的二进制）。该二进制**不能**在 x86-64 硬件上执行 → import 时 CPU 解码 ARM 指令失败。

**结论**：windows-x64 job 只能 build 不能 smoke test。真正的运行时验证留给 Stage B 的 X Elite 真机 runner。

### 4.2 `import qai_appbuilder` 缺 numpy / pyyaml

**症状**：
```
ModuleNotFoundError: No module named 'numpy'   # 然后是 'yaml'
File ".../qai_appbuilder/onnxwrapper.py", line 46
```

**根因**：[`qai_appbuilder/__init__.py`](../script/qai_appbuilder/__init__.py#L24-L26) 无条件 `from .onnxwrapper import *`；[`onnxwrapper.py`](../script/qai_appbuilder/onnxwrapper.py) 顶层 `import numpy / import yaml`；但 [`setup.py`](../setup.py#L642-L670) 没有 `install_requires`，所以 wheel 装上后干净环境必定 import 失败。

**当前修法**：CI 在 smoke test 前 `pip install numpy pyyaml`（绕过）。

**真正修法（待办，超出本次范围）**：在 `setup.py` 加
```python
install_requires=["numpy", "pyyaml"],
```

### 4.3 ruff 报 25 个存量问题

包括 2 个真正的 F821 undefined name bug：
- [`samples/python/utils/image_processing.py:205`](../samples/python/utils/image_processing.py#L205) — `numpy_image_to_torch`
- [`script/qai_appbuilder/onnxwrapper.py:681`](../script/qai_appbuilder/onnxwrapper.py#L681) — `exp_shape_by_name`

**修法**：lint job 标 `continue-on-error: true`（informational only）。值得另起 PR 单独清理。

---

## 5. 关键技术决定

1. **submodule 选择性 init** — 仓库有 11 个 submodule（包括 llama.cpp / MNN / curl，几百 MB），wheel 构建只需 `pybind/pybind11`。用 `actions/checkout submodules: false` + 手动 `git submodule update --init --depth 1 pybind/pybind11`。

2. **PEP517 隔离绕开** — `python -m build -w --no-isolation`，因为 `setup.py` 构建时要读 `QNN_SDK_ROOT` env，PEP517 隔离环境拿不到。依赖在前一步 `pip install` 显式装。

3. **QNN SDK 缓存 key** — `qnn-sdk-${version}-${runner.os}-v1`，含版本号，多版本并存不冲突。首次 cache miss ~25 min（含 ~800MB 下载），后续 ~15 min。

4. **SDK 解压结构自适配** — composite action 探测 `<dir>/lib/` 直接在根、或在 `qairt/<version>/` 下两种结构，避免不同 SDK 版本目录布局差异。

5. **2.46 vs 2.47 URL 域名不同** — 2.47 在 `softwarecenter.qualcomm.com`，2.46 在 `apigwx-aws.qualcomm.com`。通过 composite action 的 `url` input 注入，没有硬编码。

6. **artifact 命名带维度** — `wheel-win_arm64-py3.12-qnn2.47.0.260601`，16 个 job 互不冲突。

---

## 6. 当前状态

| 项目 | 状态 |
|---|---|
| ci-stage-a 分支 | 已推送到 fork (`shijunz/ai-engine-direct-helper`) |
| 上游 `quic/ai-engine-direct-helper:main` | **未推送**，未 PR |
| 单组合验证（py3.12 × qnn2.47） | ✅ x64 build / ✅ ARM64 build + smoke test 通过（commit `f3b7047`） |
| 16 组合 matrix 验证 | ⏳ commit `1da6b4b` 推送后**尚未独立验证**所有组合，需在 https://github.com/shijunz/ai-engine-direct-helper/actions 看结果 |
| 本地 `main` | 领先 origin 2 笔（`dc1c433` zipformer + `3b42c40` 早期 CI commit），未推送 |
| 工作树 stash | `stash@{0}` 含 zipformer 笔记 + samples/utils/install.py 改动，等切回 main 时 `git stash pop` |

---

## 7. 已知遗留 / 待办

### 7.1 包元数据缺失（影响所有用户，不只 CI）
[`setup.py`](../setup.py) 应加 `install_requires=["numpy", "pyyaml"]`。否则 `pip install qai-appbuilder` 后干净环境 `import qai_appbuilder` 必报 `ModuleNotFoundError`。

### 7.2 存量 lint 债务
script/ + samples/python/utils/ 共 25 个 ruff 错误，其中 2 个是真实 bug（见 4.3 节）。建议另起 PR 清理后把 lint job 改回硬性 gate (`continue-on-error: false`)。

### 7.3 16 组合可能有失败
Python 3.10/3.11/3.13 在 ARM64 hosted runner 上 setup-python 是否齐全、QNN 2.46 解压结构是否兼容当前自适配逻辑 —— 都还没 100% 验证过。若有失败，看 artifact `cmake-logs-*` 或 step 日志诊断。

---

## 8. Stage B 待办（不在本次范围）

> **重要前提**：GitHub-hosted `windows-11-arm` runner **没有 Snapdragon NPU/HTP 硬件**，它是 Azure Cobalt/Ampere 虚拟化 ARM 服务器。"在 ARM64 上跑 inception_v3" 不是真正的 NPU 端到端测试 —— 该 sample 用 `Runtime.HTP` + HTP 编译的 `.bin`，会在 `QNNContext(...)` 构造时直接报"找不到 HTP backend"。AI Hub 的 `.bin` 是 **backend-bound 的 context binary**，不能跨 runtime 加载。
>
> 已在 commit `(next)` 加了 [`tests/test_cpu_runtime_smoke.py`](../tests/test_cpu_runtime_smoke.py)，作为 **Stage B-lite**：在 ARM64 hosted runner 上**不**碰 NPU、**不**加载 `.bin`，但走完 pybind C++ 调用 + QNN backend DLL 发现路径。这能抓 ABI 不匹配、wheel 缺 DLL、pybind 层回归 —— 比纯 import 深一层、比 inception_v3 现实可行。

真正的 Stage B 需要：

- **真机 NPU 端到端测试** — self-hosted X Elite runner 跑 `samples/python/inception_v3/` 等轻量 sample。这才是 AI Hub `.bin` + HTP runtime 的真验证。
- Linux aarch64 wheel — composite action 加 bash 分支
- Tag `v*` 触发 GitHub Release，自动上传 wheel + zip
- 多 hexagon arch（73 / 75 / 79）matrix
- Android `.so` 构建（NDK r26d）

---

## 9. 相关链接

- 工作 fork：https://github.com/shijunz/ai-engine-direct-helper
- ci-stage-a 分支：https://github.com/shijunz/ai-engine-direct-helper/tree/ci-stage-a
- Actions tab：https://github.com/shijunz/ai-engine-direct-helper/actions
- 上游：https://github.com/quic/ai-engine-direct-helper
- 早期 self-hosted ARM64 思路（保留）：[`2026-06-05_GitHubActions构建wheel.md`](./2026-06-05_GitHubActions构建wheel.md)

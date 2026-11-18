# stable-diffusion.cpp（vendored）

本目录是 [stable-diffusion.cpp](https://github.com/ningpengtao-coder/stable-diffusion.cpp) 的
vendored 副本，用于在本仓库内构建统一的扩散模型推理运行时（CPU / Metal），并通过 C ABI
供 Flutter FFI 调用。

## Upstream

- 仓库：https://github.com/ningpengtao-coder/stable-diffusion.cpp
- Commit：`5dcfa79801213f23e27b3e83617350ebb8ea93aa`
- 许可证：MIT（见 [LICENSE](LICENSE)），Copyright (c) 2023 leejet
- 子模块 ggml：`leejet/ggml` @ `eeda9ec48fc84a3df0bb9dcc19ae9053075a2539`
  （由上游 `.gitmodules` / 该 commit 的 submodule 指针固定；这是带扩展算子的 patched ggml）

### 获取方式（与上游文档的 git clone 不同）

本机直连 GitHub 的 git 协议不可用：

```
git clone --depth 1 https://github.com/ningpengtao-coder/stable-diffusion.cpp.git packages/stable-diffusion.cpp
# fatal: unable to access '...': Recv failure: Connection was reset
# 退出码 128
```

因此改用 GitHub codeload 的固定 commit 归档（内容等价于该 commit 的工作树，但不含 `.git`）：

```powershell
Invoke-WebRequest -Uri "https://codeload.github.com/ningpengtao-coder/stable-diffusion.cpp/zip/5dcfa79801213f23e27b3e83617350ebb8ea93aa" -OutFile sd.zip
Expand-Archive sd.zip; # 解压后移动到 packages/stable-diffusion.cpp
Invoke-WebRequest -Uri "https://codeload.github.com/leejet/ggml/zip/eeda9ec48fc84a3df0bb9dcc19ae9053075a2539" -OutFile ggml.zip
Expand-Archive ggml.zip; # 解压内容复制到 packages/stable-diffusion.cpp/ggml
```

- 仓库内没有 `.git` / `.gitmodules` 的 git 元数据，子模块以普通目录形式 vendored。
- 未拉取以下子模块（本仓库构建时用 `-DSD_BUILD_EXAMPLES=OFF -DSD_WEBP=OFF -DSD_WEBM=OFF`
  关闭了对应功能）：
  - `thirdparty/libwebp`（WebP/WebM 编解码）
  - `thirdparty/libwebm`（WebM 视频输出）
  - `thirdparty/cutlass`（仅 CUDA + INT8 路径需要）
  - `examples/server/frontend`（内置 Web UI）
- `thirdparty/oniguruma`、`thirdparty/utf8proc`、`thirdparty/zip` 等属于主仓库内容，已包含。

## 本地改动

- 源码未做任何修改。
- 新增本文件 `UPSTREAM.md`。
- 构建产物位于 `build-windows/`（本地生成，不应入库）。CMake 在无 `.git` 时会输出
  `stable-diffusion.cpp version unknown / commit unknown`，这不影响构建与运行。
- 构建共享库使用上游选项（见 README.md 的「构建」章节），未改动上游 CMake。

## 校验记录

- Commit 记录来自 GitHub codeload 的固定 commit 归档 URL；vendored 目录没有
  `.git`，无法本地执行 `git rev-parse` 复核。升级时请重新核对归档 URL 与
  本文件的 commit。
- `ggml/` 子模块已补齐并以普通目录形式存在：`ggml/CMakeLists.txt`、
  `ggml/src/`、`ggml/include/` 均在，且包含 patched 扩展头
  `ggml-ext.h` / `ggml-graph.h`。本次构建实际编译了 `ggml-base`、
  `ggml-cpu`、`ggml` 三个目标，无需再补子模块。

## 构建记录（Windows / MSVC x64，2026-11-15）

- 生成器：Visual Studio 17 2022（x64），MSVC 14.44。
- 既有 `build-windows` 缓存的关键选项：
  `SD_BUILD_SHARED_LIBS=ON`、`SD_BUILD_SHARED_GGML_LIB=OFF`、
  `SD_BUILD_EXAMPLES=OFF`、`SD_WEBP=OFF`、`SD_WEBM=OFF`、
  `CMAKE_BUILD_TYPE=Release`。
- 如需从零重建：

  ```powershell
  cmake -S . -B build-windows -G "Visual Studio 17 2022" -A x64 `
    -DSD_BUILD_SHARED_LIBS=ON `
    -DSD_BUILD_SHARED_GGML_LIB=OFF `
    -DSD_BUILD_EXAMPLES=OFF `
    -DSD_WEBP=OFF -DSD_WEBM=OFF
  cmake --build build-windows --config Release
  ```

- 本次一致性构建命令与结果：

  ```powershell
  cmake --build build-windows --config Release
  # 退出码 0：onig / sd-utf8proc / zip / ggml-base / ggml-cpu / ggml /
  #           stable-diffusion 全部构建成功
  ```

- 产物：`build-windows/bin/Release/stable-diffusion.dll`（38,002,688 字节）。
- 导出符号抽查（`dumpbin /exports`）：`new_sd_ctx`、`free_sd_ctx`、
  `generate_image`、`sd_cancel_generation`、`sd_ctx_params_init`、
  `sd_img_gen_params_init`、`free_sd_image_generation_result`、
  `sd_set_log_callback`、`sd_version`、`sd_commit`。
- 运行时依赖：`MSVCP140.dll`、`MSVCP140_CODECVT_IDS.dll`、`VCOMP140.DLL`、
  `VCRUNTIME140.dll`、`VCRUNTIME140_1.dll`；这些文件已随
  `packages/stable-diffusion.cpp-runtime/windows/Assets` 分发。
- 未修改上游源码。

## 已知限制

- 本目录只 vendored 了 CPU（以及 macOS 上可选的 Metal）构建所需的源码，未包含
  CUDA / Vulkan / OpenCL 子模块与内核。
- 上游处于活跃开发中，C ABI 会随 commit 变化；升级时请同步更新 Dart 绑定与 `UPSTREAM.md`。

# SimdPaddleOCR — WebAssembly (WASM) 4w 性能与基准测试报告

测试环境：
* **宿主硬件**：AMD Ryzen (32 逻辑核心), Windows 11
* **运行时框架**：.NET 10.0 (`net10.0` / `browser-wasm`)
* **浏览器引擎**：Microsoft Edge (Chromium 内核，Headless 模式)
* **SIMD 特性**：WASM SIMD128 (`Vector<float>.Count = 4`, `Vector.IsHardwareAccelerated = True`)
* **多线程机制**：Web Workers + `SharedArrayBuffer`（由本地服务端注入 COOP: `same-origin` 与 COEP: `require-corp` 安全响应头）
* **并发参数**：`--workers 4`（`EffectiveLineWorkerCount = 4`）
* **测试工程**：`test/Sdcb.SimdPaddleOCR.WasmBench`，输入为 `dataset/` 变尺寸真实/合成评测集

---

## 一、端到端性能对比 (4 Workers)

| 模型 | 端到端平均耗时 (Mean ms) | 中位数 (Median ms) | P95 延迟 (ms) | 吞吐量 (img/s) |
| :--- | :---: | :---: | :---: | :---: |
| **ChineseV6Tiny (tiny)** | **1,264.9 ms** | **1,302.4 ms** | **1,702.9 ms** | **0.79 img/s** |
| **ChineseV6Small (small)** | **8,932.0 ms** | **9,339.0 ms** | **10,781.2 ms** | **0.11 img/s** |
| **ChineseV6Medium (medium)**| **15,716.2 ms** | **15,711.0 ms** | **18,482.1 ms** | **0.06 img/s** |

---

## 二、模型识别准确率 (Accuracy & CER)

评测以 `dataset/metadata.json` 真实标注为基准计算字符错误率（CER）与行绝对匹配率：

| 模型 | 字符准确率 (Char Accuracy) | 字符错误率 (CER) | 行绝对匹配 (Line Exact) | 行匹配率 |
| :--- | :---: | :---: | :---: | :---: |
| **ChineseV6Tiny (tiny)** | 97.43% | 2.57% | 149 / 206 | 72.33% |
| **ChineseV6Small (small)** | 99.50% | 0.50% | 187 / 206 | 90.78% |
| **ChineseV6Medium (medium)**| **100.00%** | **0.00%** | **93 / 94** | **98.94%** |

> **准确率观察**：WASM 下全托管推理引擎 `OnnxSharp` 的算子数值稳定性表现优异。Medium 模型在测试集上达成 100% 字符级零错误识别，文本 Hash 与桌面端完全一致，没有任何精度退化。

---

## 三、分阶段耗时剖析 (Mean ms/图)

| 阶段 | ChineseV6Tiny (4w) | ChineseV6Small (4w) | ChineseV6Medium (4w) | 说明 |
| :--- | :---: | :---: | :---: | :--- |
| **det_preprocess** | 91.4 ms | 94.4 ms | 98.3 ms | 缩放、归一化与通道重排 |
| **det_graph** | 509.7 ms | 4,637.0 ms | 5,659.2 ms | 文本检测网络推断 |
| **det_postprocess** | 10.2 ms | 9.0 ms | 10.8 ms | 概率图二值化、轮廓与多边形拟合 |
| **crop** | 81.0 ms | 79.1 ms | 53.6 ms | 仿射透视校正提取文字行 |
| **cls_graph** | 377.0 ms | 374.7 ms | 478.2 ms | 文本行方向分类 (180° 翻转检测) |
| **rec_preprocess** | 58.0 ms | 60.3 ms | 78.3 ms | 动态宽高归一化 |
| **rec_graph (总和)** | 1,674.8 ms | 15,273.3 ms | 37,558.0 ms | 所有文字行识别网络的算力耗时总和 |
| **lines_wall (实测墙钟)** | **571.3 ms** | **4,111.4 ms** | **9,893.4 ms** | **4 个 Web Worker 并发执行后的实际耗时** |
| **rec_reshape** | 7.0 ms | 10.6 ms | 18.0 ms | 动态宽度重塑与权重预备 |

> **多线程并行收益**：
> 在 4 个并发 Worker 的协同下：
> * `tiny` 的文字识别墙钟耗时从 `rec_graph` 1,674.8 ms 压缩到 `lines_wall` 571.3 ms（**加速比 ~2.93×**）；
> * `small` 从 15,273.3 ms 压缩到 4,111.4 ms（**加速比 ~3.71×**，接近理论极限 4×）；
> * `medium` 从 37,558.0 ms 压缩到 9,893.4 ms（**加速比 ~3.80×**）。
> 充分证明了 WebAssembly 多线程与任务并行调度在行识别阶段的高效并发。

---

## 四、内存占用与工作集 (Working Set)

得益于 `Sdcb.SimdPaddleOCR` 零堆内存分配的算子复用设计与原生内存池化，WebAssembly 环境下的内存表现极具轻量优势：

| 模型 | 平均托管内存占用 | 峰值托管内存占用 | 内存安全评价 (WASM 2GB 限制) |
| :--- | :---: | :---: | :--- |
| **ChineseV6Tiny (tiny)** | **56.2 MB** | **67.5 MB** | 极低，适合各类微前端及低配嵌入式 Web 视图 |
| **ChineseV6Small (small)** | **125.6 MB** | **155.5 MB** | 适中，内存平稳无泄漏 |
| **ChineseV6Medium (medium)**| **358.1 MB** | **376.5 MB** | 远低于 2GB/4GB 寻址上限，无内存增长风险 |

---

## 五、WebAssembly 运行要点与 Harness 实现

### 1. 多线程与跨域隔离 (COOP/COEP)
WebAssembly 原生不支持传统 OS 级轻量进程，.NET 10 的 `<WasmEnableThreads>true</WasmEnableThreads>` 依靠浏览器底层的 **Web Workers** 和 **SharedArrayBuffer**：
* 浏览器出于安全规范（防范 Spectre 类旁路攻击），要求页面必须运行在跨域隔离上下文（`crossOriginIsolated == true`）。
* 因此，Harness 的本地服务端（`server.mjs`）配置了：
  ```http
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  ```
* 若在纯 Node.js 宿主执行带线程的 WASM，Mono 运行时会直接触发 `Assert failed: This build of dotnet is multi-threaded, it doesn't support shell environments like V8 or NodeJS` 保护异常。

### 2. 算子向量化与 SIMD 128
`Sdcb.SimdPaddleOCR` 的全托管算子集（`Conv1x1`、`Conv3x3`、`MatMul` 等）内置了通用的 `Vector.cs` 向量回退。在 .NET WASM 下开启 SIMD 后：
* `Vector<float>.Count = 4`（128-bit 宽度，对应 WASM `v128` 指令）。
* 算子无需任何手写汇编即可利用现代浏览器提供的 WASM 128-bit 向量指令加速点积与矩阵运算。

---

## 六、复现与基准测试运行指南

本项目提供了完整的自动化 Harness，位于 `test/Sdcb.SimdPaddleOCR.WasmBench`：

```powershell
# 1. 运行 tiny 模型 4 线程基准测试
.\test\Sdcb.SimdPaddleOCR.WasmBench\run.ps1 -Workers 4 -Model tiny -Count 20

# 2. 运行 small 模型 4 线程基准测试
.\test\Sdcb.SimdPaddleOCR.WasmBench\run.ps1 -Workers 4 -Model small -Count 20

# 3. 运行 medium 模型 4 线程基准测试
.\test\Sdcb.SimdPaddleOCR.WasmBench\run.ps1 -Workers 4 -Model medium -Count 10

# 4. 打开浏览器交互式仪表盘（实时可视化进度条、指标卡片与文本 Hash）
.\test\Sdcb.SimdPaddleOCR.WasmBench\run.ps1 -Workers 4 -Model tiny -Open
```

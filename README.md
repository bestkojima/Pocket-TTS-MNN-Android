# Pocket TTS MNN Android

在 Android 手机上通过 `adb shell` 运行 Pocket TTS，输入英语文本，生成语音和性能记录。Release 提供模型、命令行程序及配套运行库，手机上无需 Python。

**当前发布的是 Android arm64 命令行测试包 `.tar.gz`，没有 APK，也没有应用界面。** 这是供真机测试的预发布版本：交叉编译、ELF 检查与 Linux 主机数值验证已完成，Android 真机精度和速度尚未验证。

- [下载 v0.1.0 测试包](https://github.com/bestkojima/Pocket-TTS-MNN-Android/releases/tag/v0.1.0)
- [完整 MNN 模型及实验报告：ModelScope](https://modelscope.cn/models/dr3334/Pocket-TTS-MNN)

## 支持范围与模型选择

| 项目 | 本版本支持范围 |
| --- | --- |
| 设备 | Android arm64-v8a，API 26 / Android 8.0 及以上；CPU 推理 |
| 语言 | **英语**，对应 `english_18m_2026-10`；其他语言没有在本版本接入或验证 |
| 音色 | 附带的预计算 **Alba** 音色；没有任意 WAV 声音克隆功能 |
| 输出 | 24 kHz、单声道、PCM16 WAV；可另存未裁剪的 float32 PCM |
| 默认模型 | **18.2M 原生 LLM 导出、原生 KV 的 W8/FP32 混合版本**，目录名 `18m-w8-mixed` |
| 默认设置 | 1 线程、1 次预热、1 次计时；`Precision_High` / `Memory_Normal` / 动态量化选项 `0` |

手机初次测试建议直接使用 Release 附带的 **18.2M 混合模型**和配套库。需要对照精度时，使用 ModelScope 中的 `18m-fp32` 作为基线。

“W8/FP32 混合”表示仅 Mimi 解码器末端 **6 个算子、26,720 个权重元素**使用 W8 存储，其他部分保留 FP32；当前 `Memory_Normal` 配置下计算仍为 FP32。它不是整个 18.2M 模型的 INT8 计算版，压缩范围很小，不能据此宣称 INT8 加速。Transformer 主干使用 MNN LLM 导出路径与原生 KV 缓存，Drifting 采样头、流式 Mimi 解码器作为独立子图调度。

命令行程序可通过 `--model` 切换到匹配的 18M / 7M / 2M 原生 KV 导出目录，但 Release **只附带 18.2M 混合版**。早期显式状态导出不适用。

## 下载与校验

电脑需准备 `adb`。以下下载步骤使用 GitHub CLI，也可以在 Release 页面下载同名文件。

```bash
gh release download v0.1.0 --repo bestkojima/Pocket-TTS-MNN-Android \
  --pattern 'pockettts-android-arm64.tar.gz' \
  --pattern 'pockettts-android-arm64.tar.gz.sha256'
sha256sum -c pockettts-android-arm64.tar.gz.sha256
tar -xzf pockettts-android-arm64.tar.gz
```

压缩包大小为 **74,432,761 字节**，SHA-256：

```text
b82a55ee3c6c87e1020a60202d780d9154aab86d798e0bb16d2b1c59ab2602d6
```

请保留解压后的电脑端目录，后面的精度比较程序需要其中的参考音频。

## 导入手机并运行

开启手机的 USB 调试，连接电脑，确认 `adb devices` 可以看到设备。

```bash
adb devices
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.build.version.sdk
adb push pockettts-android-arm64.tar.gz /data/local/tmp/
adb shell 'cd /data/local/tmp && tar -xzf pockettts-android-arm64.tar.gz && chmod 755 pockettts-android-arm64/bin/pockettts_cli_pockettts_nativekv'
```

ABI 应为 `arm64-v8a`，SDK 数字应至少为 `26`。请使用 `/data/local/tmp`；`/sdcard` 通常不能执行 ELF 文件。

先以 **1 线程**运行一句英语。默认先预热 1 次，再记录 1 次：

```bash
adb shell 'cd /data/local/tmp/pockettts-android-arm64 && sh run.sh --threads 1 --text "Hello world. This is Pocket TTS running locally."'
adb pull /data/local/tmp/pockettts-android-arm64/output.wav .
adb pull /data/local/tmp/pockettts-android-arm64/output.json .
```

再以 **4 线程**、预热 1 次、重复计时 3 次测试。指定不同文件名可保留前一组结果：

```bash
adb shell 'cd /data/local/tmp/pockettts-android-arm64 && sh run.sh --threads 4 --warmup 1 --repeat 3 --output /data/local/tmp/pockettts-android-arm64/test4.wav --text "Hello world. This is Pocket TTS running locally."'
adb pull /data/local/tmp/pockettts-android-arm64/test4.wav .
adb pull /data/local/tmp/pockettts-android-arm64/test4.json .
```

`--threads 4` 是推理时使用 4 线程，无需重新导出模型。`--repeat 3` 在同一个进程中执行三轮，每轮重置请求状态；JSON 保留全部轮次，WAV 保存最后一轮。同名输出文件会覆盖。

查看完整参数：

```bash
adb shell 'cd /data/local/tmp/pockettts-android-arm64 && sh run.sh --help'
```

## 怎样看性能记录

终端显示实际输入文本、音频时长、首块耗时、合成耗时、RTF 和峰值 RSS；相邻 JSON 文件还保存模型、线程数、精度设置、加载时间、预热时间及每一轮结果。

| 指标 | 含义 |
| --- | --- |
| `audio_seconds` | 输出 PCM 的音频时长 |
| `wall_seconds` | 本轮合成耗时，包含缓存重置/导入、文本 prefill、生成及 PCM 写盘与 flush |
| `rtf` | `wall_seconds / audio_seconds`，越小越快；低于 1 表示生成快于实时 |
| `first_audio_ms` | 到首块 PCM 生成的耗时，不包含加载和预热，不是实际扬声器播放延迟 |
| `peak_rss_mib` | 进程峰值内存，包括已加载模型；不是单轮新增内存 |
| `load_seconds` / `warmup_seconds` | 加载和预热耗时，单独记录，不计入各轮 RTF |

文本分句、规范化和 token 准备在预热与计时前完成。比较 1 / 4 线程时请保持文本、模型、噪声、设备电源模式和温度尽量一致，并查看多轮结果。当前没有 Android 实测 RTF；Linux 主机速度不能作为手机速度。

## 固定噪声精度检查

普通运行采用 C++ 随机数。即使 `seed` 相同，也不保证与 NumPy 或其他 C++ 标准库产生相同噪声。逐样本检查请使用随包提供的固定噪声和固定文本：

```bash
adb shell 'cd /data/local/tmp/pockettts-android-arm64 && sh run_compare.sh --threads 1 --warmup 1 --repeat 1'
adb pull /data/local/tmp/pockettts-android-arm64/compare.f32 compare-t1.f32
adb pull /data/local/tmp/pockettts-android-arm64/compare.json compare-t1.json
python3 pockettts-android-arm64/compare.py compare-t1.f32

adb shell 'cd /data/local/tmp/pockettts-android-arm64 && sh run_compare.sh --threads 4 --warmup 1 --repeat 3'
adb pull /data/local/tmp/pockettts-android-arm64/compare.f32 compare-t4.f32
adb pull /data/local/tmp/pockettts-android-arm64/compare.json compare-t4.json
adb pull /data/local/tmp/pockettts-android-arm64/compare.wav .
python3 pockettts-android-arm64/compare.py compare-t4.f32
```

比较脚本只需要电脑上的 Python 3 标准库，检查样本数相同、数值有限，以及**完整 float32 PCM 的相对 L2 误差低于 1%**，不做裁剪或时间对齐。`passed: true` 才表示该次测试通过。该门限是整段逐样本向量的相对 L2，不代表每一个采样点各自相对误差都低于 1%。

参考 PCM 来自同模型、同文本、同实际噪声的已验收 Linux Python/MNN 运行。修改 `run_compare.sh` 的文本、温度或模型后，随包参考便不再适用。手机结果仍需真机检查，不能由主机测试推定。

已有验证：600 例文本/token 对照、9 例空输入拒绝检查通过；Linux 原生 CLI 的 8 个完整波形用例与 Python/MNN 参考逐字节一致，包含五个模型变体短句及 18.2M 混合模型长句、1 / 4 线程测试。预热与重复请求检查通过。已保存的完整实验材料见 [ModelScope 模型仓库](https://modelscope.cn/models/dr3334/Pocket-TTS-MNN)。这些是主机移植一致性证据，不是 Android 性能或主观音质评分。

## 文件布局与兼容性

```text
pockettts-android-arm64/
├── bin/pockettts_cli_pockettts_nativekv
├── lib/libMNN_pockettts_nativekv.so
├── lib/libMNN_Express_pockettts_nativekv.so
├── lib/libpockettts_mnn_pockettts_nativekv.so
├── lib/libc++_shared.so
├── models/18m-w8-mixed/
├── fixtures/
├── run.sh
├── run_compare.sh
├── compare.py
├── validation/
├── build-records/
└── licenses/
```

请推送完整目录，单独拷贝可执行程序不能完成推理。SentencePiece 已静态链接在程序内，其他所需第三方运行库随包提供。

此程序依赖本项目适配过的 MNN 原生 KV 实现和匹配的 Core / Express / 桥接库，**不保证兼容所有 MNN 版本**。新编译的可执行文件与三份 MNN 库都采用 `_pockettts_nativekv` 后缀，不覆盖原有 MNN 库；NDK 标准运行库 `libc++_shared.so` 保留标准库名。库名后缀不等同于同进程符号隔离，请通过本包启动脚本在独立 shell 进程运行。集成到现有 App 需要另外验证 ABI 和链接关系。

构建采用 NDK r28b、Release / **`-O3 -DNDEBUG`**，AArch64 PIE、16 KB LOAD 对齐；没有开启会改变浮点归约顺序的全局 fast-math。请保留初始 `high / normal / dynamic0` 设置；切换 `--precision low`、`--memory low` 或动态 INT8 不继承现有精度结论。

## 来源与许可

- [上游 Pocket TTS 代码](https://github.com/kyutai-labs/pocket-tts)
- [实际使用的原模型：kyutai/pocket-tts-without-voice-cloning](https://huggingface.co/kyutai/pocket-tts-without-voice-cloning)
- [上游 Pocket TTS Hugging Face 主页](https://huggingface.co/kyutai/pocket-tts)
- [MNN](https://github.com/alibaba/MNN)
- [转换后的模型、FP32 基线与完整实验材料](https://modelscope.cn/models/dr3334/Pocket-TTS-MNN)

模型权重、tokenizer 和 Alba 派生状态采用 **CC BY 4.0**，Alba 音色归属 **Alba MacKenna**。MNN 及本项目新增 CLI 代码采用 **Apache-2.0**；上游 Pocket TTS 代码采用 **MIT**，SentencePiece、protobuf、Darts 与 Android NDK 保留各自许可。具体归属见 [NOTICE](NOTICE.md)，第三方许可和模型来源记录随 Release 包提供；分发模型时请一并保留。

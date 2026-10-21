# v0.1.0 — Android arm64 命令行测试包

包含 18.2M 原生 LLM / 原生 KV 的 W8/FP32 混合模型、Android arm64 CLI、配套运行库和固定噪声数值检查。最低 API 26 / Android 8.0，CPU 推理，英语与预计算 Alba 音色。

这是 `.tar.gz` 命令行包，没有 APK。下载后按 [README](https://github.com/bestkojima/Pocket-TTS-MNN-Android#readme) 通过 adb 导入 `/data/local/tmp`，可选择 1 / 4 线程、预热和多轮计时，输出 24 kHz WAV、float32 PCM 与 JSON 指标。

- NDK r28b、Release `-O3 -DNDEBUG`、AArch64 PIE、16 KB LOAD 对齐。
- 三份 MNN 库与 CLI 使用 `_pockettts_nativekv` 后缀，保持原有库不变。
- 仅 Mimi 末端六算子使用 W8 存储，当前计算仍为 FP32；不代表完整 INT8 加速。
- Linux 主机 600 例文本/token、9 例空输入检查通过；8 例完整 float32 波形与 Python/MNN 参考逐字节一致。
- **尚未在 Android 真机执行；本次以预发布测试版发布，手机精度、RTF 与设备兼容性需实测。**

完整包 `pockettts-android-arm64.tar.gz` 为 74,432,761 字节；SHA-256：

```text
b82a55ee3c6c87e1020a60202d780d9154aab86d798e0bb16d2b1c59ab2602d6
```

请同时下载 `.sha256` 文件并校验。权重/tokenizer/Alba 派生状态为 CC BY 4.0；软件与依赖各自许可见 [NOTICE](https://github.com/bestkojima/Pocket-TTS-MNN-Android/blob/main/NOTICE.md)。

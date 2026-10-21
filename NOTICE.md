# 来源与许可

本仓库发布第三方 Pocket TTS → MNN 的 Android 命令行测试包，非 Kyutai 或 MNN 官方发布。

- MNN 和新增集成代码采用 Apache-2.0，见 [LICENSE](LICENSE)。MNN 来源：<https://github.com/alibaba/MNN>。
- 上游 Pocket TTS 代码采用 MIT：<https://github.com/kyutai-labs/pocket-tts>。复用部分保留该许可，全文随 Release 的 `licenses/pocket-tts-MIT.txt` 提供。
- 权重、tokenizer 和派生预计算音色保留 **CC BY 4.0**，不由软件许可替代。原模型为 [kyutai/pocket-tts-without-voice-cloning](https://huggingface.co/kyutai/pocket-tts-without-voice-cloning)，18.2M 使用 `english_18m_2026-10` 基础权重。
- Alba 音色作者为 **Alba MacKenna**，由 Kyutai 发布：[来源说明](https://huggingface.co/kyutai/tts-voices/blob/main/README.md#alba-mackenna)、[原始录音](https://huggingface.co/kyutai/tts-voices/blob/main/alba-mackenna/casual.wav)。包内为转换后的预计算状态，没有附带原始录音。
- CC BY 4.0：[许可说明](https://creativecommons.org/licenses/by/4.0/)、[法律文本](https://creativecommons.org/licenses/by/4.0/legalcode.en)。
- 模型修改包括流式 MNN 子图、原生 KV 布局、外置 FP32 权重，以及 Mimi 末端六个算子的 W8 存储。其他部分保留 FP32，计算采用 High / Normal / dynamic0。
- SentencePiece、protobuf、Darts、FlatBuffers、half 和 Android NDK 的第三方声明随 Release 的 `licenses/` 提供。

Kyutai 与 Alba MacKenna 未为本次第三方适配背书。再次分发模型时应保留来源、修改说明和各自许可。

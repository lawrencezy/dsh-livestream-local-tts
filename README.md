# DSH 直播间 · 本地 TTS 教程与补丁

这是一个**免费分享**的实践记录：把原版 `dsh-livestream` 的语音调用接上本机 Qwen3-TTS，让舞台真的说话、显示字幕、返回结束回执。

不是完整发行版，也不是卖课。这里保留能复用的代码和步骤；角色、音色、字体等素材请各自确认授权。

## 上游与许可

- 上游项目：`luckui/dsh-livestream`，MIT（源码安装）。
- 本仓库的增量补丁和文档：MIT。
- **不包含**上游角色素材。上游角色为 CC BY-NC-SA 4.0（非商业），不能拿来做商业直播或售卖；本仓库不附带角色、音色、字体文件。
- 本地 TTS 模型（Qwen3-TTS）请按模型自己的许可使用。

## 已验证的范围

在本机 Windows 10、Python 3.12、Node 24、Qwen3-TTS 0.6B 的隔离副本里验证：

- 普通模式：本机现场生成 4.72 秒新台词，舞台播放到 4.72 秒，`ended`，字幕整句可见，`/said` 回执 1 次。
- 预览模式：0 个 Audio、0 次 `/said`，只显示字幕。
- 生产文件和配置前后 SHA256 一致；没有开播、没有推流、没有改生产包。

未验证：真实 B 站弹幕、OBS、推流、豆包线上 TTS、逐音素口型、并发/长时间稳定性。

## 目录

- `patches/local-tts.patch`：`runtime.ts` 的本地 TTS 分支、`ttsLocal.ts`、常驻 worker。
- `docs/TUTORIAL.md`：从跑原版到验证声音的完整步骤。
- `docs/VERIFICATION.md`：这次验证的数值和检查方法。
- `NOTICE.md`：上游、素材和模型许可说明。

## 快速开始

```bash
# 1) 先按上游 README 跑通原版，确认后台和舞台能打开
# 2) 在 dsh-livestream 包根目录应用补丁
patch -p1 < patches/local-tts.patch

# 3) 安装本地 TTS 依赖并准备模型；具体路径用环境变量给出
export DSH_LOCAL_TTS=1
export DSH_LOCAL_TTS_PYTHON=/path/to/python
export DSH_LOCAL_TTS_WORKER=/path/to/dsh-livestream/tools/local_tts_worker.py
export DSH_LOCAL_TTS_MODEL=/path/to/Qwen3-TTS-12Hz-0.6B-CustomVoice
export DSH_LOCAL_TTS_SPEAKER=Serena

# 4) 启动后先在预览模式确认字幕；再切普通模式听声音
# 5) 按 docs/VERIFICATION.md 检查 currentTime、ended、字幕和 /said
```

补丁默认关闭本地分支：不设置 `DSH_LOCAL_TTS=1` 时走原来的语音路径；本地失败会回落，不会静默改生产行为。

## 常见问题

- 第一句没声音：模型冷启动可能超过 15 秒，先预热再说话；补丁里有 `warmLocalTts()` 的接入点。
- 下一句突然没声音：检查上一句的超时监听是否在请求结束后移除，别让它误杀常驻 worker。
- 字幕和声音不一致：当前字幕按字数均分时间轴，不是逐音素对齐；不要把它宣传成逐音素口型。
- 角色素材能不能商用：不能直接把上游角色打包售卖或开播收礼；换自己的角色/音色/字体并确认授权。

## 相关

- 上游：https://github.com/luckui/dsh-livestream
- 教程视频：2 分 41 秒竖屏教程已完成（2026-11-15）；发布链接待补。

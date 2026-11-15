# 教程：让 DSH 直播间用自己的声音说话

这份教程记录一次真实跑通的路径。先跑原版，再加本地语音；每一步都能退回。

## 0. 先确认原版工作

按上游 `luckui/dsh-livestream` 的 README 安装，启动后确认：

1. 后台能打开；
2. 舞台能显示角色；
3. 预览模式能显示字幕；
4. 不改任何代码，原版能正常工作。

如果原版都没跑通，先别接 TTS。把安装问题单独解决，不要把它和语音问题混在一起。

## 1. 准备声音引擎

准备 Python、Qwen3-TTS 和模型目录，并确认能单独生成一条 WAV：

```bash
python -c "import your_tts_package; print('ok')"
```

先单独生成一条 3–5 秒的短句，确认采样率、时长和音量正常，再接入舞台。这样出问题能分清是引擎还是舞台。

## 2. 应用补丁

在 `dsh-livestream` 包根目录：

```bash
patch -p1 < /path/to/patches/local-tts.patch
```

补丁做三件事：

1. 在 `src/dsh/runtime.ts` 的 `speech()` 调用点加入本地分支；
2. 新增 `src/body/ttsLocal.ts`：生成本机 WAV、转 mp3、返回音频和时间轴；
3. 新增 `tools/local_tts_worker.py`：常驻 worker，模型只加载一次。

**默认关闭**：只有 `DSH_LOCAL_TTS=1` 才走本地；失败会回落到原来的语音路径。

## 3. 配置环境变量

```bash
export DSH_LOCAL_TTS=1
export DSH_LOCAL_TTS_PYTHON=/path/to/python
export DSH_LOCAL_TTS_WORKER=/path/to/dsh-livestream/tools/local_tts_worker.py
export DSH_LOCAL_TTS_MODEL=/path/to/Qwen3-TTS-12Hz-0.6B-CustomVoice
export DSH_LOCAL_TTS_SPEAKER=Serena
```

不要把真实密钥写进仓库、日志或截图。这个补丁只调用本机模型，不需要云端 TTS key。

## 4. 启动并验证

1. 先启动后台和舞台；
2. 用预览模式发一句话，确认字幕出现、没有音频请求；
3. 切普通模式，发一句短句；
4. 观察并记录：
   - `audio.currentTime` 是否从 0 走到音频时长；
   - 是否触发 `ended`；
   - 字幕是否可见；
   - `/said` 回执是否返回，id 是否和广播一致；
5. 再发第二句，确认常驻 worker 没被上一句的超时监听误杀。

## 5. 这次实测结果

- 本机生成 4.72 秒新台词，生成耗时 7.14 秒；
- 舞台播放到 4.72 秒，`ended`；
- 字幕整句可见；
- `/said` 回执 1 次，id 与广播 id 一致；
- 预览模式 0 个 Audio、0 次 `/said`；
- 生产文件和配置前后 SHA256 一致。

## 6. 两个真实坑

### 坑 1：超时信号误杀常驻 worker

上一句的 15 秒超时监听在调用结束后仍然存在，到点把 worker 杀了，下一句就没有声音。修法：每次请求结束就移除监听；取消只让本次请求失败，不杀 worker。

### 坑 2：冷启动被 15 秒超时截断

模型第一次加载约 18.7 秒，首句还没生成就被判超时。修法：启动后先 `warmLocalTts()` 预热，再让舞台说话。

## 7. 边界

- 只在本机隔离环境验证；真实 B 站弹幕、OBS、推流、线上豆包 TTS、逐音素口型都没验证。
- 字幕按字数均分时间轴，不是逐音素对齐。
- 上游角色是非商用素材；本仓库不附带角色、音色、字体。商用请换自己的资产并确认授权。

## 8. 回退

删掉补丁或恢复备份的 `src/dsh/runtime.ts`、`src/body/ttsLocal.ts`、`tools/local_tts_worker.py`，并取消 `DSH_LOCAL_TTS`；原语音路径不受影响。

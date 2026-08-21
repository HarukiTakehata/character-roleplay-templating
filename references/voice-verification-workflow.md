# B站语音验证工作流（角色卡交叉验证）

> 本文件描述如何通过 Bilibili 角色全语音视频提取语音文本，与已有角色数据交叉验证。
> **B站通用技术细节（API、下载、转写）见 `chinese-web-scraping` 技能的「3.9 Bilibili — API 直链下载 + Whisper 转写」章节。**

## 适用场景

- 已有角色故事/角色卡初稿，需要验证其性格描写、人际关系、说话风格是否准确
- Wiki 文本无法覆盖语音中的互动细节（如角色对其他角色的评价）
- 需要提取真实台词的经典语录补充角色卡

## 工作流

### 1. 搜索语音视频

```
https://search.bilibili.com/all?keyword={角色名} 全语音
```

优先找播放量最高、时长最长的全语音合集（通常10-20分钟）。完整的全语音视频比切片更适合验证。

通过 browser_console 提取 BV 号：
```js
const links = document.querySelectorAll('a[href*="video/BV"]');
links.forEach(a => {
  const m = a.href.match(/video\/(BV[a-zA-Z0-9]+)/);
  if (m) console.log(m[1], '-', a.textContent.trim().substring(0,60));
});
```

### 2. 下载 + 转写

B站 API 直链获取、ffmpeg 音频提取、Whisper 转写的完整模板见 **`chinese-web-scraping` 技能 `references/bilibili-video-extraction.md`**。注意：

- 不使用 `vad_filter=True`（导致生成器不可重入）
- 17分钟音频 small 模型约 8-10 分钟 CPU 转写
- 超时后 `librosa.load()` 切割续转

### 3. 交叉验证

提取的语音文本可分为几类，与已有角色数据对照：

| 语音类别 | 验证目标 |
|:---------|:---------|
| 自我介绍/开场白 | 角色核心身份、态度 |
| 关于其他角色的评价 | 人际关系描述准确性 |
| 闲聊/天气语音 | 日常生活态度、性格基调 |
| 战斗/受击语音 | 战斗中的性格反差 |
| 关于特定主题的独白 | 深层次世界观、价值观 |

### 4. 验证产出

- **确认项**：语音与已有数据一致 → 高可信度标记
- **深化项**：语音补充了已有关系的细节 → 更新角色卡
- **新发现项**：语音中提到的角色/事件在已有数据中未覆盖 → 补充
- **矛盾项**：语音与已有数据不一致 → 需进一步调查
- **新增语录**：语音中的经典句子 → 补充到 IDENTITY.md 的经典语录章节

## 常见问题

| 问题 | 解决方案 |
|:----|:---------|
| URL 获取后下载失败 | deadline 过期，重新获取 URL 立即下载（详见 chinese-web-scraping） |
| 中文专有名词误识别 | 角色名/地名常见误识别，手动校对后标注 `[听写存疑]` |
| vad_filter=True 错误 | 不加 VAD 直接转写（详见 chinese-web-scraping 陷阱#8） |

## 文件清理

```bash
rm -f /tmp/{角色名}_voice.mp4 /tmp/{角色名}_voice.mp3 /tmp/{角色名}_voice_transcript.txt
```

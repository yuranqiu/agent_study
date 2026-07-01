# Multi-Modal Agent · 多模态 Agent

> **核心问题**：Agent 如何感知和理解图像、音频、视频等多模态信息？
> **预计学时**：5h
> **对应 Phase**：Phase 5

---

## 学习目标

1. 能解释多模态 Agent 的架构设计（感知-规划-执行）
2. 能说出图文多模态的主流方案（CLIP、LLaVA、GPT-4V）
3. 能为 Agent 添加图像理解工具并正确调用
4. 能设计简单的多模态 ReAct 循环

---

## 1. 多模态 Agent：为什么需要

### 单模态的局限

纯文本 Agent 只能处理文字，但在真实世界：

- **用户拍了一张产品图**：需要识别商品
- **用户发了一段语音**：需要转文字再理解
- **用户给了一个表格截图**：需要提取数据
- **用户问视频里发生了什么**：需要理解动态内容

### 多模态融合的三个层次

| 层次 | 方式 | 代表工作 |
|------|------|---------|
| 早期融合 | 原始信号直接拼在一起 | Perceiver IO |
| 晚期融合 | 各模态独立编码后再融合 | CLIP + LLM |
| 跨模态对齐 | 通过对齐信号关联不同模态 | BLIP-2 |

---

## 2. 视觉编码器方案

### 2.1 CLIP 系列

CLIP（Contrastive Language-Image Pretraining）通过大规模图文对学习视觉表示：

```
图像 → ViT encoder → embedding
文本 → Transformer → embedding
对比学习拉近匹配对距离
```

**优点**：零样本能力强大，直接用文本描述查询图像
**局限**：无法精细定位，只能做整体理解

### 2.2 LLaVA 系列

LLaVA（Large Language-and-Vision Assistant）是开源多模态方案的代表：

```
图像 → CLIP ViT → feature
文本 + 图像 feature → LLM（Vicuna/Phi）
```

**LLaVA-1.5 改进**：
- 使用 MLP 投影层替代线性层
- 增加高质量数据（GCSD、LAION）
- 在 Science QA 等任务上接近 GPT-4V

### 2.3 GPT-4V / Gemini / Claude Vision

| 模型 | 图像输入 | 特点 |
|------|---------|------|
| GPT-4V | 单图/多图 | 文档理解强，指代清晰 |
| Gemini Pro | 多模态交错 | 支持视频帧序列 |
| Claude 3.5 | 单图/多图 | 视觉推理强 |

---

## 3. 音频与语音处理

### 3.1 语音识别（ASR）

Whisper 是主流的开源 ASR 模型：

```python
from transformers import WhisperProcessor, WhisperForConditionalGeneration

processor = WhisperProcessor.from_pretrained("openai/whisper-base")
model = WhisperForConditionalGeneration.from_pretrained("openai/whisper-base")

def transcribe(audio_path):
    import librosa
    audio, sr = librosa.load(audio_path, sr=16000)
    input_features = processor(audio, sampling_rate=16000, return_tensors="pt").input_features
    generated_ids = model.generate(input_features)
    transcript = processor.batch_decode(generated_ids, skip_special_tokens=True)
    return transcript[0]
```

### 3.2 语音合成（TTS）

主流方案：
- **Edge TTS**：微软开源，低延迟，支持多语言
- **VALL-E**：通过 3 秒音频克隆音色
- **CosyVoice**：阿里开源，支持零样本克隆

### 3.3 多模态 Agent 中的语音处理流程

```
用户语音 → ASR（Whisper） → 文本 → LLM 理解 → 响应
                                     ↓
用户期望语音 ← TTS（Edge） ← 文本响应
```

---

## 4. 多模态 Agent 架构设计

### 4.1 感知-规划-执行循环

```
┌─────────────────────────────────────┐
│  Perception（感知层）                │
│  - 文本解析                          │
│  - 图像编码（CLIP/ViT）              │
│  - 音频编码（Whisper/SpeechBERT）    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Planning（规划层）                  │
│  - 多模态理解（LLM）                 │
│  - 任务分解（ReAct/CoT）             │
│  - 工具选择                          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  Execution（执行层）                 │
│  - Tool 调用                        │
│  - 跨模态生成（图像/语音/文本）       │
└─────────────────────────────────────┘
```

### 4.2 多模态 ReAct 实现

```python
from transformers import CLIPProcessor, CLIPVisionModel
import torch

class MultimodalAgent:
    def __init__(self, llm, vision_model="openai/clip-vit-base-patch32"):
        self.llm = llm
        self.vision_model = CLIPVisionModel.from_pretrained(vision_model)
        self.processor = CLIPProcessor.from_pretrained(vision_model)
    
    def perceive_image(self, image_path: str) -> str:
        """将图像转为文本描述"""
        from PIL import Image
        image = Image.open(image_path)
        inputs = self.processor(images=image, return_tensors="pt")
        with torch.no_grad():
            image_features = self.vision_model(**inputs).last_hidden_state
        return f"[IMAGE_FEATURES: shape={image_features.shape}]"
    
    def run(self, user_input: str, image_path: str = None):
        # 1. 感知：处理多模态输入
        context = user_input
        if image_path:
            image_desc = self.perceive_image(image_path)
            context = f"User input: {user_input}\nImage context: {image_desc}"
        
        # 2. 规划：ReAct 循环
        max_iterations = 10
        for i in range(max_iterations):
            thought = self.llm.think(f"Task: {context}\nThink step {i+1}:")
            
            if self.llm.should_act(thought):
                action = self.llm.extract_action(thought)
                result = self.execute_tool(action)
                context += f"\nObservation: {result}"
            else:
                return self.llm.generate(context)
        
        return "Max iterations reached"
```

---

## 5. 工具调用：图像处理工具

### 5.1 图像理解工具

```python
import base64
from io import BytesIO
from PIL import Image

def encode_image_base64(image_path: str) -> str:
    """将图像转为 base64 供 API 调用"""
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

def describe_image(image_path: str, model: str = "gpt-4o") -> str:
    """使用视觉模型描述图像内容"""
    image_b64 = encode_image_base64(image_path)
    
    if model.startswith("gpt"):
        from openai import OpenAI
        client = OpenAI()
        response = client.chat.completions.create(
            model=model,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{image_b64}"}},
                    {"type": "text", "text": "描述这张图像的内容"}
                ]
            }]
        )
        return response.choices[0].message.content
```

### 5.2 图像生成工具（工具返回）

Agent 也可以生成图像作为响应：

```python
def generate_image(prompt: str, tool: str = "dall-e-3") -> str:
    """图像生成工具"""
    if tool == "dall-e-3":
        from openai import OpenAI
        client = OpenAI()
        response = client.images.generate(
            model="dall-e-3",
            prompt=prompt,
            size="1024x1024",
            n=1
        )
        return response.data[0].url
```

---

## 6. 视频理解：时序多模态

### 6.1 视频帧采样策略

视频包含时空信息，处理方式：

- **均匀采样**：每 N 帧取一帧，适合短视频
- **关键帧检测**：用场景检测算法挑重要帧
- **密集采样 + 压缩**：所有帧编码后用注意力融合

### 6.2 视频多模态 Agent 示例

```python
import cv2

def extract_keyframes(video_path: str, num_frames: int = 8) -> list:
    """提取关键帧"""
    cap = cv2.VideoCapture(video_path)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    frame_indices = [int(i * total_frames / num_frames) for i in range(num_frames)]
    frames = []
    
    for idx in frame_indices:
        cap.set(cv2.CAP_PROP_POS_FRAMES, idx)
        ret, frame = cap.read()
        if ret:
            frames.append(cv2.cvtColor(frame, cv2.COLOR_BGR2RGB))
    cap.release()
    return frames
```

---

## 练习清单

- [ ] 安装 transformers + PIL，用 CLIP 提取一张图片的 feature（shape 验证）
- [ ] 用 LLaVA 或 GPT-4V API 对一张图片做内容描述
- [ ] 用 Whisper 识别一段音频（>3秒）的文字内容
- [ ] 实现 extract_keyframes 函数，在本地视频上测试
- [ ] 组合：语音输入 → ASR → LLM → TTS 语音输出，跑通全流程

---

## 关联资源

- [LLaVA 论文](https://arxiv.org/abs/2304.08485)
- [CLIP 论文](https://arxiv.org/abs/2103.00020)
- [BLIP-2 论文](https://arxiv.org/abs/2301.12597)
- [Whisper 论文](https://arxiv.org/abs/2212.04356)
- [LangChain 多模态指南](https://python.langchain.com/docs/tutorials/multimodal/)

---

*最后更新：2026-05-27*
**IP-Adapter（Image Prompt Adapter，图像提示词适配器）让模型在生成时参考一张图片。** 例如，你给它一张漫画人物图，再用文字描述「站在雨夜的街头」，模型就可以参考图中的人物外观、配色或画风，画出新的场景。它输出的是供 `KSampler` 使用的 **`MODEL`**。[ip-adapter.github.io](https://ip-adapter.github.io/?utm_source=chatgpt.com)

以下按常见的 `ComfyUI_IPAdapter_plus` 插件说明。**基础版 `IPAdapter` 有 3 个必接输入、1 个可选输入、1 个输出、4 个可调参数。** 如果你的界面上还有 `combine_embeds` 和 `embeds_scaling`，你用的是 **`IPAdapter Advanced`**。[GitHub](https://github.com/cubiq/ComfyUI_IPAdapter_plus/blob/main/IPAdapterPlus.py?utm_source=chatgpt.com)

## 一、输入接口

| 接口             | 类型          | 适用版本   | 代表什么，通常接哪里                                                                                                                         |
| ---------------- | ------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `model`          | `MODEL`       | 两版必接   | **负责画图的基础模型**。使用 `IPAdapter Unified Loader` 时，接它的 `MODEL` 输出。                                                            |
| `ipadapter`      | `IPADAPTER`   | 两版必接   | **让基础模型读懂参考图的适配器**。基础版通常接同一个 `IPAdapter Unified Loader` 的 `ipadapter` 输出。                                        |
| `image`          | `IMAGE`       | 两版必接   | **参考哪张图**。接 `Load Image` 的 `IMAGE` 输出，例如人物立绘、照片或风格样图。                                                              |
| `attn_mask`      | `MASK`        | 两版可选   | **参考图影响成品的哪个区域**。白色区域影响最强，黑色区域关闭影响，灰色介于两者之间。例如，只希望左侧人物参考这张立绘，就把目标画面左侧涂白。 |
| `image_negative` | `IMAGE`       | 高级版可选 | **负向参考图**。用图片提供希望减少的视觉特征，通常可以留空。                                                                                 |
| `clip_vision`    | `CLIP_VISION` | 高级版可选 | **提取参考图特征的视觉编码器**。使用 Unified Loader 时一般可留空。若使用独立的 `IPAdapter Model Loader`，需要另接匹配的 `Load CLIP Vision`。 |

这里容易混淆的是 `image` 和 `attn_mask`：前者决定**参考什么**，后者决定**在成品的哪里使用这份参考**。遮罩按目标画面的布局制作。[GitHub](https://github.com/cubiq/ComfyUI_IPAdapter_plus/blob/main/IPAdapterPlus.py?utm_source=chatgpt.com)

## 二、输出接口

| 接口    | 输出内容                 | 下一步接哪里                                                                                                                                                                                                                    |
| ------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MODEL` | 已加入图像参考能力的模型 | 接 `KSampler.model`。KSampler 的 `positive`、`negative` 仍接提示词条件，`latent_image` 仍可接 `Empty Latent Image`。[GitHub](https://github.com/cubiq/ComfyUI_IPAdapter_plus/blob/main/IPAdapterPlus.py?utm_source=chatgpt.com) |

## 三、节点自身可以调节的参数

| 分类     | 参数             | 默认值                             | 具体作用                                                                                                                                                                                          |
| -------- | ---------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 参考强度 | `weight`         | `1.0`                              | 决定参考图影响有多强。人物外观偏离参考图，可以调高；想换场景，却连参考图的背景都被带进来，可以调低。这里的数值是权重，不能直接换算成相似度百分比。                                                |
| 参考方式 | `weight_type`    | 基础版 `standard`，高级版 `linear` | 决定参考图的影响怎样分配到模型的不同层。基础版有 `standard`（常规参考）、`prompt is more important`（让文字有更多影响）、`style transfer`（侧重画风）。高级版还可选 `composition`（侧重构图）等。 |
| 生效时间 | `start_at`       | `0.0`                              | 从去噪进程的哪个位置**开始**使用参考图。`0.0` 表示从头开始，`0.3` 表示到约 30% 的位置才开始。                                                                                                     |
| 生效时间 | `end_at`         | `1.0`                              | 到哪个位置**停止**使用参考图。若 `start_at=0`、`end_at=0.7`，参考图从开头生效，到约 70% 的位置停止。                                                                                              |
| 多图合并 | `combine_embeds` | `concat`，仅高级版                 | 输入多张参考图时，决定怎样合并它们的视觉特征。`concat` 拼接，`add` 相加，`average` 取平均，`norm average` 归一化后平均，`subtract` 用第一张的特征减去其余图的平均特征。单张参考图可以保留默认。   |
| 特征加权 | `embeds_scaling` | `V only`，仅高级版                 | 决定 `weight` 怎样作用到图像注意力。`V only` 主要调节传入的视觉信息量；`K+V` 同时改变特征匹配和传入的信息量。带 `C penalty` 的选项还会按通道维度调整权重。                                        |

区分 `weight_type` 和 `start_at / end_at` 最直观的方法是：**想让模型偏重画风或构图，改 `weight_type`；想让参考图只在生成的某一段生效，改 `start_at / end_at`。** [GitHub](https://github.com/cubiq/ComfyUI_IPAdapter_plus/blob/main/IPAdapterPlus.py?utm_source=chatgpt.com)

结合你前面问的 DWPose，假设目标是**让角色 A 做出照片 B 里的举手动作**：把 **A 的立绘**接到 `IPAdapter.image`，把 **B 的动作照片**接到 `DWPose.image`。角色外观偏了，先看 IP-Adapter 的参考图和 `weight`；手臂位置偏了，先预览 DWPose 骨架，骨架正确后再看 ControlNet 的 `strength`。[ip-adapter.github.io](https://ip-adapter.github.io/?utm_source=chatgpt.com)

**`InpaintModelConditioning` 用来为局部重绘准备输入。** 比如你想把照片里的红色外套换成蓝色，SAM3 负责选中外套，`InpaintModelConditioning` 把**原图、外套遮罩和“蓝色外套”的提示词**整理好，再交给 KSampler 重绘。这个节点有 **5 个接线输入、3 个输出、1 个可调开关**。[GitHub](https://github.com/Comfy-Org/ComfyUI/blob/master/nodes.py?utm_source=chatgpt.com)

### 一、输入接口

| 接口       | 类型           | 代表什么，通常接哪里                                                                                                                           |
| ---------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `positive` | `CONDITIONING` | **希望重绘后出现什么。** 接正向 `CLIP Text Encode`，例如「一件深蓝色牛仔外套」。                                                               |
| `negative` | `CONDITIONING` | **希望避免什么。** 接负向 `CLIP Text Encode`，例如「文字、水印」。实际影响取决于模型和 CFG 设置。                                              |
| `vae`      | `VAE`          | **把原图编码为模型使用的 latent。** 接 `Load Checkpoint` 的 `VAE` 输出，或接与基础模型匹配的 `Load VAE`。                                      |
| `pixels`   | `IMAGE`        | **要编辑的完整原图。** 接 `Load Image.IMAGE`。如果前面用了 SAM3，这里接送入 SAM3 的那张原图。                                                  |
| `mask`     | `MASK`         | **指定要修改的位置。** 白色表示希望重绘的区域，黑色表示希望保留的区域。接 SAM3 的 `masks` 输出；如果检测出多件外套，先选中目标外套对应的遮罩。 |

在这个例子里，SAM3 的 `text_prompt` 写 **`red jacket`**，用于找到原图里已有的红外套；重绘的 `positive` 写 **`blue denim jacket`**，描述希望生成的蓝外套。两处文字的任务不同。[github.com](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/segmentation.py?utm_source=chatgpt.com)

### 二、输出接口

| 输出       | 类型           | 里面有什么                                                                    | 接到哪里                |
| ---------- | -------------- | ----------------------------------------------------------------------------- | ----------------------- |
| `positive` | `CONDITIONING` | 正向提示条件，加上遮罩和遮住待改区域后的图像信息                              | `KSampler.positive`     |
| `negative` | `CONDITIONING` | 负向提示条件，也附带同一份重绘信息                                            | `KSampler.negative`     |
| `latent`   | `LATENT`       | **完整原图**编码后的 latent，作为采样起点。开启 `noise_mask` 时还携带采样遮罩 | `KSampler.latent_image` |

因此，这条工作流里，KSampler 的 `latent_image` 直接接本节点的 `latent`。KSampler 的 `model` 单独接兼容的重绘模型，采样结果再经 `VAE Decode` 得到图片。[GitHub](https://github.com/Comfy-Org/ComfyUI/blob/master/nodes.py?utm_source=chatgpt.com)

### 三、节点自身的可调参数

唯一的开关是 **`noise_mask`**，默认 `true`。它决定**是否把遮罩也交给采样器，用来限制修改范围**。[GitHub](https://github.com/Comfy-Org/ComfyUI/blob/master/nodes.py?utm_source=chatgpt.com)

| 设置            | 实际作用                                                           | 用外套举例                                             |
| --------------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| `true`（开启）  | 把遮罩附在输出的 `latent` 上，KSampler 按遮罩限制采样更新。        | 只想换外套、尽量保留脸和背景时，从默认值开始。         |
| `false`（关闭） | 重绘模型仍会收到原图信息和遮罩，但 KSampler 可以更新遮罩外的区域。 | 模型仍知道要换外套，周围的头发、光影或背景也可能变化。 |

所以，**决定“改哪件外套”用 `mask`，决定“采样时是否限制在这件外套内”用 `noise_mask`，决定“重画幅度有多大”则调 KSampler 的 `denoise`**。[GitHub](https://github.com/Comfy-Org/ComfyUI/blob/master/nodes.py?utm_source=chatgpt.com)

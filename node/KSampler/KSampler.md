ComfyUI 的普通版 **K 采样器（KSampler）** 有 **4 个接线输入、1 个接线输出**。节点内通常能看到 **7 个可调项**，其中 6 个参与本次采样，`control_after_generate` 管理下一次运行的种子。[ComfyUI](https://docs.comfy.org/zh/built-in-nodes/sampling/ksampler?utm_source=chatgpt.com)

## 一、输入接口：KSampler 需要什么

| 接口           | 类型           | 代表什么                                           | 通常接哪里                                                                                           |
| -------------- | -------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `model`        | `MODEL`        | **用哪个模型生成**                                 | 接 `Load Checkpoint` 的 `MODEL` 输出。使用 IP-Adapter 时，接 IP-Adapter 处理后的 `MODEL`。           |
| `positive`     | `CONDITIONING` | **希望画面出现什么**，例如「一个人在窗边举起右手」 | 接正向 `CLIP Text Encode` 的输出。使用姿势 ControlNet 时，接 `Apply ControlNet` 的 `positive` 输出。 |
| `negative`     | `CONDITIONING` | **希望画面减少什么**，例如「模糊、水印」           | 接负向 `CLIP Text Encode` 的输出。使用 ControlNet 时，接 `Apply ControlNet` 的 `negative` 输出。     |
| `latent_image` | `LATENT`       | **从什么画面基础开始**                             | 文生图接 `Empty Latent Image`，在上游设宽高。图生图接原图经过 `VAE Encode` 后的输出。                |

结合前面讲的 DWPose：它输出的**骨架图**接 `Apply ControlNet.image`，由 ControlNet 把姿势信息加入正、负条件，再送进 KSampler。这样，即使 `latent_image` 接的是空画布，生成的人物也能参考骨架摆姿势。[ComfyUI](https://docs.comfy.org/zh/built-in-nodes/sampling/ksampler?utm_source=chatgpt.com)

## 二、输出接口：KSampler 生成什么

| 接口                  | 输出内容                                               | 下一步接哪里                                                                              |
| --------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| `LATENT`（`samples`） | **采样后的潜空间图像**，也就是模型内部对画面的数字表示 | 接 `VAE Decode` 的 `samples` 输入，解码为 `IMAGE`，再接 `Preview Image` 或 `Save Image`。 |

因此，如果你想**预览最终图片**，要从 `VAE Decode` 的输出接预览。[ComfyUI](https://docs.comfy.org/zh/built-in-nodes/sampling/ksampler?utm_source=chatgpt.com)

## 三、节点内的 7 个可调项

| 分类       | 参数                                | 代表什么                              | 怎么理解或调整                                                                        |
| ---------- | ----------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------- |
| 随机起点   | `seed` 随机种子                     | 决定起始噪声的图案                    | 同一句提示词，换种子就可能换构图、姿势和细节。比较其他参数时先固定它。                |
| 种子管理   | `control_after_generate` 生成后控制 | 决定下一次运行怎么选种子              | `fixed` 保持，`randomize` 随机，`increment` 加 1，`decrement` 减 1。                  |
| 采样过程   | `steps` 步数                        | **整个过程走多少步**                  | 默认 `20`。增加步数通常更慢，画质收益会逐渐减小。适合的步数取决于模型。               |
| 采样过程   | `sampler_name` 采样器               | **每一步用什么算法更新画面**          | 例如 `euler`、`dpmpp_2m`。种子和步数相同，换算法也可能改变纹理和细节。                |
| 采样过程   | `scheduler` 调度器                  | **这些步分布在哪些噪声水平上**        | 例如 `normal`、`karras`。即使都是 20 步，也可以在高噪声和低噪声阶段采用不同的步距。   |
| 提示词引导 | `cfg` 引导强度                      | 控制正、负条件对生成方向的影响        | 标准 CFG 下，`1` 表示采用正向预测。继续调高会放大正、负预测的差异，过高可能损害画质。 |
| 原图重绘   | `denoise` 降噪强度                  | 控制从多高的噪声水平开始，范围 `0～1` | 常规文生图用 `1.0`。图生图里，值低通常更保留原图，值高则允许更大幅度重画。            |

例如，把同一张人物照片改成水彩时，固定 `seed`、提示词和 `steps`，只把 `denoise` 从 `0.3` 调到 `0.7`，就能观察**允许重画的幅度**怎样影响原图的姿势、衣服和背景。若你想比较**提示词引导有多强**，则固定 `denoise`，只改 `cfg`。[ComfyUI](https://docs.comfy.org/zh/built-in-nodes/sampling/ksampler?utm_source=chatgpt.com)

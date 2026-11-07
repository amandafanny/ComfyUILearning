**Depth Anything 的功能是从一张普通图片估计场景中各处的远近，输出一张灰度深度图。** 以常见的 `comfyui_controlnet_aux` 插件为例，名为 **`Depth Anything`** 的节点有 **1 个图片输入、1 个图片输出、2 个可调参数**。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything.py?utm_source=chatgpt.com)

### 一、输入接口

| 接口    | 类型    | 接什么                                                                                                                                                                                                                                                  |
| ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image` | `IMAGE` | 接 `Load Image` 输出的原图。比如想让新图保留「前景伸出的手、中间的人物、远处的街道」这三层空间关系，就输入这张街景照片。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything.py?utm_source=chatgpt.com) |

### 二、输出接口

| 接口    | 输出内容                                                                                                                                           | 接到哪里                                                                                                                                                                                                                                                             |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IMAGE` | **相对深度图**。在同一张图里，通常越靠近镜头越亮，越远越暗。上例中，伸向镜头的手应比脸亮，脸又比远处的街道亮。灰度表示相对远近，无法直接换算成米。 | 接 `Preview Image` 检查远近关系。要用它控制生成，就接 `Apply ControlNet.image`，并为该节点加载与基础模型匹配的 **Depth ControlNet**。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything.py?utm_source=chatgpt.com) |

### 三、节点自身的 2 个可调参数

| 参数         | 默认值                      | 代表什么                                                                                                                                                                                                                                                                                                                                      |
| ------------ | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ckpt_name`  | `depth_anything_vitl14.pth` | 选择**哪个模型来估计深度**。文件名里的 `vits`、`vitb`、`vitl` 分别代表 Small、Base、Large。Small 较快，Base 居中，Large 是默认的大模型。这里选择的是读参考图的模型，生成最终画面的基础模型仍由工作流上游加载。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything.py?utm_source=chatgpt.com) |
| `resolution` | `512`                       | 设置**预处理图像的短边尺寸**，保留宽高比。例如，输入 `1200 × 800` 的横图，设为 `512`，输出深度图为 `768 × 512`。最终生成图片的宽高由 KSampler 接收的 latent 决定。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything.py?utm_source=chatgpt.com)                                             |

如果你的节点标题是 **`Depth Anything V2 - Relative`**，接口和参数分类相同，默认模型换成了 `depth_anything_v2_vitl.pth`。这个版本还有一个值得注意的细节：插件先用固定的 `input_size=518` 做深度估计，再按 `resolution` 缩放输出。因此，把 `512` 调到 `1024` 会得到更大的深度图，但不会让模型重新以更高精度判断手和脸的距离。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/depth_anything_v2.py?utm_source=chatgpt.com)

结合前面的 DWPose，**想让角色“举起右手”，看 DWPose 的关节位置；想让这只手“伸向镜头，比脸更近”，看 Depth Anything 的明暗层次。** 如果深度图里的前后关系已经正确，但生成结果跟随得太弱，再调下游 `Apply ControlNet` 的 `strength`。[github.com](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/src/custom_controlnet_aux/dwpose/__init__.py?utm_source=chatgpt.com)

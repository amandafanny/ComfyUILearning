**`SAM3 Segmentation (RMBG)` 用文字指定要选中的物体，一次输出抠好的图、遮罩和物体位置。** 例如，给它一张人物照片，输入 `red jacket`，就可以用 `IMAGE` 得到透明背景的外套，用 `MASK` 指定局部重绘区域，用 `MASK_IMAGE` 检查袖口有没有漏选。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

以下按 **1038lab 的 `ComfyUI-RMBG` 插件**说明，节点内部名称是 `SAM3Segment`。当前版本有 **1 个接线输入、4 个输出、13 个节点内设置**。模型直接在节点里选择，无需另接 SAM3 模型加载器。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

### 一、输入接口

| 输入    | 类型    | 接什么                                                                                                                                                                                                                                                                             |
| ------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image` | `IMAGE` | 接 `Load Image` 的图片，或 `Load Video FFmpeg (Upload)` 输出的一批视频帧。输入视频帧时，节点会**逐帧独立分割**。画面里有多个人时，同一排序编号在下一帧可能指向另一个人。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com) |

### 二、输出接口

| 输出         | 类型    | 内容与接法                                                                                                                                               |
| ------------ | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IMAGE`      | `IMAGE` | **抠图成品**，保留被选物体原本的颜色和纹理。`background=Alpha` 输出带透明通道的 RGBA 图；选 `Color` 则把物体合成到指定纯色背景上。                       |
| `MASK`       | `MASK`  | **供下游使用的遮罩**。默认白色是选中区域，黑色是其余区域。做局部重绘接 `InpaintModelConditioning.mask`；做 Wan 换人可接 `WanVideo Animate Embeds.mask`。 |
| `MASK_IMAGE` | `IMAGE` | **供人查看的遮罩图**。把同一张遮罩转成三通道灰度图，可直接接 `Preview Image` 检查轮廓、漏选和边缘。                                                      |
| `BBOXES`     | `BBOX`  | 每个保留物体的矩形外框，格式为 `[左, 上, 右, 下]` 的像素坐标。可接支持 `BBOX` 的裁剪或精细抠图节点。选择 `Merged` 后，遮罩会合并，框仍按物体分别返回。   |

实际接线时，**要透明素材取 `IMAGE`，要告诉另一个节点“改哪里”取 `MASK`，要肉眼检查选区取 `MASK_IMAGE`**。如果你的版本只有前三个输出，`BBOXES` 是插件后来增加的接口。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

### 三、节点自身的 13 个可调参数

| 分类     | 参数                   | 默认值                  | 代表什么、怎么调                                                                                                                        |
| -------- | ---------------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| 识别模型 | `model_name`           | `sam3.1_multiplex_fp16` | 选择分割模型。当前版本另有 `sam3` 可选。                                                                                                |
| 识别目标 | `prompt`               | 空                      | 写**原图里要找的东西**。选整个人用 `person`，只选外套用 `red jacket`。留空时源码会使用 `object`。                                       |
| 检测筛选 | `confidence_threshold` | `0.50`                  | 检测置信度门槛，范围 `0.05–0.95`。目标没被找到，可以降低；无关物体也被选中，可以提高。                                                  |
| 实例输出 | `output_mode`          | `Merged`                | `Merged` 把一张图中保留的物体合为一张遮罩；`Separate` 为每个物体各输出一张抠图和遮罩。                                                  |
| 数量上限 | `max_segments`         | `0`                     | 每张输入图最多保留几个候选，范围 `0–128`。`0` 表示不限，`2` 表示最多保留两个。                                                          |
| 指定实例 | `segment_pick`         | `0`                     | 从保留的候选中选第几个，范围 `0–128`。`0` 表示全部，`1` 选排在第一的候选。排序依据是模型的遮罩响应，和画面从左到右的顺序无关。          |
| 边缘柔化 | `mask_blur`            | `0`                     | 遮罩高斯模糊半径，范围 `0–64`。提高后边缘会由黑到白渐变，适合缓和合成时的硬边，也可能模糊细发丝。                                       |
| 边缘扩缩 | `mask_offset`          | `0`                     | 范围 `-64～64`。**正值扩大白区，负值收缩白区**。袖口漏了一圈可从 `+1` 试起；边缘带进一圈背景可从 `-1` 试起。                            |
| 选区反转 | `invert_output`        | `false`                 | 用于交换前景和背景选区。当前版本在 `Merged` 模式下有重复反转的问题，见下文。                                                            |
| 背景类型 | `background`           | `Alpha`                 | `Alpha` 保留透明背景；`Color` 用纯色填充背景。它控制抠图成品的外观。                                                                    |
| 背景颜色 | `background_color`     | `#222222`               | `background=Color` 时使用的颜色。例如 `#FFFFFF` 为白色，`#00FF00` 为绿色。                                                              |
| 运行设备 | `device`               | `Auto`                  | 可选 `Auto`、`CPU`、`GPU`。`Auto` 由插件选择设备；当前实现中，手动选 `GPU` 要求 CUDA 可用。                                             |
| 显存管理 | `unload_model`         | `false`                 | 关闭时可复用已加载的模型；开启后，节点执行完会移除模型缓存。后面马上要运行较大的 Wan 模型时可用它腾出显存，代价是下次分割需要重新加载。 |

当前源码在 `Merged` 模式下会先处理每个物体的遮罩，合并后再处理一次。因此，`invert_output=true` 在只有一个物体时可能反转两次，最后又选回原物体；模糊和扩缩也可能叠加。**想得到“所有人物以外的背景”时，可靠的接法是 `Merged`、`invert_output=false`，再把输出的 `MASK` 接到 `Invert Mask`，只反转一次。** [GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

### 四、用三个人的画面理解 `Merged`、`Separate` 和 `segment_pick`

假设 `prompt=person`，一张图里有三个人通过了置信度门槛。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

| 设置                                           | 结果                                                          |
| ---------------------------------------------- | ------------------------------------------------------------- |
| `Merged`，`max_segments=0`，`segment_pick=0`   | **1 张遮罩**，三个人都是白色。适合统一换背景。                |
| `Separate`，`max_segments=0`，`segment_pick=0` | **3 张遮罩**，每张只选中一个人。适合分别抠图。                |
| `Separate`，`max_segments=2`，`segment_pick=1` | 先保留排序靠前的两人，再取其中第一人，最终得到 **1 张遮罩**。 |

对你前面的 **Wan 视频换人流程**，区别很实际：若输入 81 帧，`Merged` 会为每个输入帧输出一张遮罩，便于和 81 帧背景对齐；`Separate` 的输出数量取决于每帧检测到几个人。若画面里有多人、你只想换其中一位，可以先用 `person in a red jacket` 这样的描述缩小目标，再预览整段 `MASK_IMAGE`，检查遮罩有没有中途跳到别人身上。[GitHub](https://github.com/1038lab/ComfyUI-RMBG/blob/main/py/AILab_SAM3Segment.py?utm_source=chatgpt.com)

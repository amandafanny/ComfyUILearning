**`WanVideo Animate Embeds` 把“谁来演、怎么动、用什么表情、保留哪个背景”整理成一组视觉条件，交给 Wan 视频采样器。** 比如，你有一张漫画人物立绘和一段真人挥手视频：立绘提供角色外观，真人视频提供挥手动作和面部表情。这个节点输出 `image_embeds`，实际生成视频由下游采样器完成。[GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/example_workflows/wanvideo_WanAnimate_example_01.json?utm_source=chatgpt.com)

以下按 **Kijai 的 `ComfyUI-WanVideoWrapper`** 说明。`WanVideo Animate Embeds` 有 **1 个必接输入、7 个可选接线输入、1 个输出、9 个可调参数**。[GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/nodes.py?utm_source=chatgpt.com)

### 一、输入接口

| 输入              | 类型                     | 是否必接               | 代表什么，通常接哪里                                                                                                                                    |
| ----------------- | ------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vae`             | `WANVAE`                 | **必接**               | 接 `WanVideo VAE Loader`。它把角色参考图、姿势图等编码成 Wan 模型能使用的潜在空间表示。                                                                 |
| `ref_images`      | `IMAGE`                  | 可选，角色动画通常要接 | **谁来演。** 接目标角色的参考图，例如漫画人物立绘。想让漫画人物模仿真人挥手，这里放漫画人物。                                                           |
| `clip_embeds`     | `WANVIDIMAGE_CLIPEMBEDS` | 可选                   | 接 `WanVideo ClipVision Encode` 的输出。通常让同一张角色参考图再走一条 CLIP Vision 分支，为模型补充外观特征。                                           |
| `pose_images`     | `IMAGE`                  | 可选                   | **身体怎么动。** 接驱动视频逐帧提取的姿势图，例如 DWPose 的黑底彩色骨架图。此口接 `IMAGE`，DWPose 的关键点坐标输出另有用途。                            |
| `face_images`     | `IMAGE`                  | 可选                   | **脸上怎么动。** 接从驱动视频逐帧裁出的真人脸部图像，用来传递表情、嘴形等变化。插件会将输入居中裁切并缩放到 `512 × 512`，所以应先裁出人脸，再送入此口。 |
| `bg_images`       | `IMAGE`                  | 可选                   | **保留哪个场景。** 换人时接原视频的背景帧。作者的示例先用人物遮罩盖住原演员，再把处理后的背景帧接进来。                                                 |
| `mask`            | `MASK`                   | 可选                   | **画面里要换谁。** 换人时，白色标出待替换的人物，黑色标出希望保留的背景。遮罩要与 `bg_images` 逐帧对齐。                                                |
| `start_ref_image` | `IMAGE`                  | 可选                   | **从哪一帧接着演。** 例如把上一段生成视频的末帧接进来，让新片段从已有画面延续。接入它会触发插件的分窗口生成流程。                                       |

最容易接反的是 **`ref_images` 和 `face_images`**。前者放**成片里想看到的角色**，后者放**提供表情的演员**。而且 `pose_images` 和 `face_images` 都应是一串按时间排列的帧。[GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/nodes.py?utm_source=chatgpt.com)

### 二、输出接口

| 输出           | 类型                 | 通常接哪里                                                                                                 |
| -------------- | -------------------- | ---------------------------------------------------------------------------------------------------------- |
| `image_embeds` | `WANVIDIMAGE_EMBEDS` | 接 **`WanVideo Sampler.image_embeds`**。其中打包了角色参考、姿势、表情，以及可选的背景、遮罩和帧窗口设置。 |

结合前一个节点看：**`WanVideo TextEncode Cached.text_embeds` 接采样器的 `text_embeds`；本节点的 `image_embeds` 接采样器的 `image_embeds`。** 一条线传“文字描述什么”，另一条线传“角色长什么样、动作怎么演”。[github.com](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/nodes.py?utm_source=chatgpt.com)

### 三、节点自身的 9 个可调参数

| 分类       | 参数                |     默认值 | 具体作用                                                                                                                                                   |
| ---------- | ------------------- | ---------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 画面尺寸   | `width`             |      `832` | 目标画面宽度。界面步进为 8，但这版代码实际会**向下对齐到 16 的倍数**，例如填 `840`，实际用 `832`。                                                         |
| 画面尺寸   | `height`            |      `480` | 目标画面高度，同样向下对齐到 16 的倍数。宽高一起决定横屏还是竖屏。                                                                                         |
| 时间长度   | `num_frames`        |       `81` | **希望生成多少帧**。节点按 `4n+1` 向下对齐，比如填 `80` 实际按 `77`，填 `81` 则保留 `81`。播放时长还取决于导出 FPS。                                       |
| 分段生成   | `frame_window_size` |       `77` | **每个窗口处理多少帧**。总帧数大于窗口时，插件分段生成，让后一个窗口承接前一个窗口的末帧。较小的窗口通常能降低单次运算的显存压力。                         |
| 身体动作   | `pose_strength`     |      `1.0` | 姿势条件的倍率，范围 `0–10`。骨架预览正确，但生成的人物抬手幅度不足时，可以单独提高它做对比。                                                              |
| 面部表情   | `face_strength`     |      `1.0` | 面部动作条件的倍率，范围 `0–10`。身体动作已经合适，但眨眼、张嘴的表现偏弱时，先检查 `face_images` 的裁脸质量，再调这一项。                                 |
| 色彩一致性 | `colormatch`        | `disabled` | 在插件的分窗口流程中，把生成帧的色彩向**角色参考图**匹配，用于减轻片段间的色调漂移。可选 `mkl`、`hm`、`reinhard`、`mvgd`、`hm-mvgd-hm`、`hm-mkl-hm`。      |
| 显存管理   | `force_offload`     |     `true` | 本节点准备好条件后，将 **VAE** 移到卸载设备，通常是 CPU，为后续采样腾出显存。之后再次使用 VAE 时可能需要搬回 GPU。                                         |
| 显存管理   | `tiled_vae`         |    `false` | 本节点执行 VAE 编码时按空间分块，以降低峰值显存。遇到**编码参考图或姿势帧时**显存不足，可以打开；它和按时间分段的 `frame_window_size` 管的是两个不同阶段。 |

**默认的 `81` 帧和 `77` 帧窗口会触发分段。** 作者的示例说明，插件会把不足一个完整窗口的尾段补齐，所以成片可能多于请求帧数。若你需要恰好 81 帧，且显存足够，可以设 `num_frames=81`、`frame_window_size=81`，并让 `start_ref_image` 留空。若改用采样器的 `context_options` 分段，也应让这两个数相等，避免同时启用两套分段机制。[GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/nodes.py?utm_source=chatgpt.com)

### 四、两种常见接法

| 你的目标                           | 怎样接                                                                                                                                                           |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **让立绘模仿真人动作**             | 立绘接 `ref_images`；同一张立绘经 CLIP Vision 编码后接 `clip_embeds`；真人视频的骨架帧接 `pose_images`，裁脸帧接 `face_images`。`bg_images` 和 `mask` 可以留空。 |
| **保留真人视频的场景，只替换演员** | 在上面的接法上，再把盖住原演员的背景帧接 `bg_images`，把逐帧人物遮罩接 `mask`。重点检查**骨架、裁脸、背景、遮罩是否来自同一段视频、同一起始时间和帧率**。        |

例如，第 20 帧骨架已经举手，而第 20 帧遮罩还停在举手前的位置，此时优先修正帧对齐；调高 `pose_strength` 无法把错位的遮罩移回手臂上。[GitHub](https://github.com/kijai/ComfyUI-WanVideoWrapper/blob/main/example_workflows/wanvideo_WanAnimate_example_01.json?utm_source=chatgpt.com)

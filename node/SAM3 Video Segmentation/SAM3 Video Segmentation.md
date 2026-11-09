**`SAM3 Video Segmentation` 用来在视频里选中目标，并跟踪它在后续帧中的位置。** 比如要在 Wan 工作流里替换一位演员，你可以在人物清晰的一帧点中他，再得到随他移动的逐帧遮罩。和前面 `SAM3 Segmentation (RMBG)` 逐帧独立识别相比，视频跟踪会利用帧与帧之间的信息来维持目标身份。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

ComfyUI 有两个插件使用**完全相同的节点名称**。看右侧输出，就能确定该读哪一部分。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

| 你的节点界面                                             | 对应版本                                                          | 工作方式                                           |
| -------------------------------------------------------- | ----------------------------------------------------------------- | -------------------------------------------------- |
| 右侧只有 **`video_state`**，左侧有 `prompt_mode`         | **A．ComfyUI-SAM3**，与前面讲的 `SAM3 Text Segmentation` 同一插件 | 本节点登记视频和提示，下游节点执行跟踪并输出遮罩。 |
| 右侧有 **`masks`、`session_id`、`objects`、`obj_masks`** | **B．ComfyUI-Easy-Sam3**                                          | 本节点直接运行跟踪并输出遮罩。                     |

## A．右侧只有 `video_state` 的版本

### 1. 输入接口

| 输入              | 类型                 | 代表什么，怎么接                                                                                                                                |
| ----------------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `video_frames`    | `IMAGE`              | 按时间排列的视频帧。用前面讲的 **`Load Video FFmpeg (Upload)`** 时，将它的 `IMAGE` 输出接到这里。                                               |
| `video`           | `VIDEO`              | 也可以直接接 ComfyUI 原生 `Load Video` 的 `VIDEO` 输出。与 `video_frames` **二选一**；两个都接时优先使用 `video`。                              |
| `positive_points` | `SAM3_POINTS_PROMPT` | `point` 模式使用。点击**要跟踪的人物内部**，例如外套中央，告诉模型“我要这个人”。                                                                |
| `negative_points` | `SAM3_POINTS_PROMPT` | `point` 模式使用。点击容易误选的区域，例如旁边另一个人的外套，告诉模型“这里排除”。同一组正、负点共同描述一个目标。                              |
| `positive_boxes`  | `SAM3_BOXES_PROMPT`  | `box` 模式使用。在提示帧上框住目标。当前实现取这个接口中的第一个框。                                                                            |
| `negative_boxes`  | `SAM3_BOXES_PROMPT`  | 界面定义为负向排除框，也只取第一个。当前插件在把框交给跟踪器时，对正负框使用了相同的角点标签；要明确排除干扰人物，用 `point` 模式的负点更可靠。 |

这里**没有模型输入口**。模型加载器的输出要接到下一步 `SAM3 Propagate.sam3_model_config`。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

### 2. 节点自身的 4 个可调参数

| 参数              | 默认值 | 控制什么                                                                                                         |
| ----------------- | ------ | ---------------------------------------------------------------------------------------------------------------- |
| `prompt_mode`     | `text` | **怎样指定目标**。`text` 用文字找，`point` 用点选，`box` 用框选。一次执行只读取所选模式对应的提示。              |
| `text_prompt`     | 空     | `text` 模式下填写目标描述。例如 `person in a red jacket`。英文逗号可分隔多个概念，如 `person, dog`。             |
| `frame_idx`       | `0`    | **在哪一帧给模型看提示**，从 `0` 开始。若演员开头被遮挡，第 21 帧才清晰，可以设 `20`，并在第 21 帧上点选或画框。 |
| `score_threshold` | `0.30` | **检测置信度门槛**，范围 `0–1`，步进 `0.05`。文字描述正确却漏检时可降低；相似物体也被选中时可提高。              |

如果画面里有三个人，输入 `person` 可能同时找到三人。**要替换其中某一人，优先在清晰帧上点中他**，比单纯提高置信度更能表达“我要哪一个”。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

### 3. 输出接口与后续接线

唯一输出 **`video_state`**，类型为 `SAM3_VIDEO_STATE`，保存了视频信息、提示帧和目标提示。要得到能接入 Wan 的标准 `MASK`，还需两步。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

| 步骤       | 怎么接                                                                                               | 在这里调什么                                                                                                                     |
| ---------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| ① 执行跟踪 | `SAM3 Video Segmentation.video_state` → `SAM3 Propagate.video_state`，并给 `SAM3 Propagate` 接上模型 | `start_frame` 是传播起点，`end_frame=-1` 表示到末尾；`direction` 可选向后续帧 `forward`、向之前帧 `backward` 或双向 `both`。     |
| ② 提取遮罩 | 把 `SAM3 Propagate` 的 `masks`、`video_state` 接到 `SAM3 Video Output` 对应输入，`scores` 可选       | `obj_id=-1` 合并所有目标；选具体 ID 可单独取出一个目标。它的 **`masks` 输出才是标准 `MASK`**，`visualization` 用于检查跟踪效果。 |

`frame_idx` 和 `start_frame` 容易混淆：**前者决定在哪一帧告诉模型“找谁”，后者决定从哪一帧开始传播**。如果你在第 21 帧选人，并希望覆盖前后两段，可让传播从索引 `20` 开始，方向选 `both`，然后预览整段遮罩。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

## B．右侧直接输出 `masks` 的 Easy-Sam3 版本

### 1. 输入和输出接口

| 分类         | 接口                                         | 作用                                                                                                |
| ------------ | -------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 必接输入     | `sam3_model`、`video_frames`                 | 分别接按 **`video` 模式**加载的 SAM3 模型，以及视频的 `IMAGE` 帧序列。                              |
| 可选提示输入 | `positive_coords`、`negative_coords`、`bbox` | 在提示帧上提供前景点、排除点或目标框。点坐标使用 JSON，例如 `[{"x":50,"y":120}]`。                  |
| 可选配置输入 | `session_id`、`extra_config`                 | 指定跟踪会话标识，或传入 `SAM3 Video Model Extra Config` 的高级配置。                               |
| 输出         | `masks`                                      | **每帧一张合并遮罩**。同一帧中识别到多个物体时取并集，可直接接标准 `MASK` 输入。                    |
| 输出         | `session_id`                                 | 这次跟踪的会话标识。是否保留会话由 `close_after_propagation` 控制。                                 |
| 输出         | `objects`                                    | 对象 ID 和各对象的逐帧遮罩。需要只取一个演员时，可接 `SAM3 Get Object Mask`。                       |
| 输出         | `obj_masks`                                  | 按 **帧 × 对象 × 高 × 宽**组织的遮罩，保留对象维度。普通节点需要每帧一张遮罩时，用 `masks` 更直接。 |

这些接口属于 Easy-Sam3，此版本已经在节点内部执行传播。[GitHub](https://github.com/yolain/ComfyUI-Easy-Sam3/blob/main/nodes.py?utm_source=chatgpt.com)

### 2. 节点自身的可调参数

| 分类     | 参数                        | 默认值或起点 | 代表什么                                                                           |
| -------- | --------------------------- | ------------ | ---------------------------------------------------------------------------------- |
| 目标     | `prompt`                    | 空           | 用文字描述要跟踪的物体，例如 `person in a red jacket`。                            |
| 目标     | `frame_index`               | 从 `0` 起    | 在哪一帧施加文字、点或框提示。                                                     |
| 目标     | `object_id`                 | `1`          | 添加提示时使用的对象 ID，用于区分跟踪目标。                                        |
| 筛选     | `score_threshold_detection` | `0.5`        | 接受检测结果的门槛。调高会筛掉更多低置信度检测。                                   |
| 筛选     | `new_det_thresh`            | `0.7`        | 允许一次新检测**加入为新跟踪对象**的门槛。若后续帧不断冒出无关对象，重点看这一项。 |
| 跟踪范围 | `propagation_direction`     | `both`       | `forward` 向后续帧跟踪，`backward` 向之前帧跟踪，`both` 双向跟踪。                 |
| 跟踪范围 | `start_frame_index`         | `0`          | 传播从哪一帧开始，可与施加提示的 `frame_index` 分别设置。                          |
| 跟踪范围 | `max_frames_to_track`       | `-1`         | 最多跟踪多少帧。`-1` 表示不人为限制。                                              |
| 资源管理 | `close_after_propagation`   | `true`       | 跟踪完成后关闭本次会话。                                                           |
| 资源管理 | `keep_model_loaded`         | `false`      | 是否让模型继续驻留在运行设备上。连续处理多段素材时，保留模型可以减少反复搬运。     |

**用在你的 Wan 换人流程里**：A 版接 **`SAM3 Video Output.masks → WanVideo Animate Embeds.mask`**；B 版若只跟踪了一个人，直接接 **`SAM3 Video Segmentation.masks → WanVideo Animate Embeds.mask`**。如果 B 版同时跟踪多人，先从 `objects` 取出目标演员，再接 Wan，否则同一帧里的其他人也会进入替换区域。[GitHub](https://github.com/PozzettiAndrea/ComfyUI-SAM3/blob/main/nodes/sam3_video_nodes.py?utm_source=chatgpt.com)

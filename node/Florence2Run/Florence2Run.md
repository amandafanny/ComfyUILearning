**`Florence2Run` 是 ComfyUI 里的看图分析节点。** 同一张照片，选 `caption` 可以让它描述画面，选 `ocr` 可以读取文字，选 `referring_expression_segmentation` 并输入 `red jacket`，可以让它找出红色外套并输出遮罩。**先选 `task`，再决定要不要填 `text_input`，以及使用哪个输出。** [GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

以下按 **Kijai 的 `ComfyUI-Florence2` 插件**说明。`Florence2Run` 有 **2 个接线输入、4 个输出、9 个节点内设置**。[GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

### 一、输入接口

| 接口              | 类型       | 接什么                                                                                                                                                              |
| ----------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image`           | `IMAGE`    | **待分析的图片**。接 `Load Image` 的 `IMAGE` 输出，也可以接上游节点生成的图。                                                                                       |
| `florence2_model` | `FL2MODEL` | **负责看图的 Florence-2 模型**。接 `DownloadAndLoadFlorence2Model` 或 `Florence2ModelLoader` 的输出。做文档问答或生成特定格式的提示词时，需要选择支持该任务的模型。 |

例如，普通图片描述可以用 `Florence-2-large-ft`；文档问答可用 `Florence-2-DocVQA`；`prompt_gen_*` 系列任务要配合支持相应指令的 PromptGen 模型。[GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

### 二、输出接口

| 接口      | 类型     | 输出什么                                                                                                             | 常见用途                                           |
| --------- | -------- | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `image`   | `IMAGE`  | 在检测、分割或带位置的 OCR 任务中，输出画了框、轮廓或文字标注的预览图。                                              | 接 `Preview Image`，核对模型找到了什么。           |
| `mask`    | `MASK`   | 检测任务可以输出**矩形框遮罩**；分割任务输出沿物体轮廓绘制的**多边形遮罩**。同一张图里选中的多个区域会合到一张遮罩。 | 接 `InpaintModelConditioning.mask`，用于局部重绘。 |
| `caption` | `STRING` | 模型生成的文字。随任务不同，可以是画面描述、识别出的文字、文档问答答案，或生成的提示词。                             | 看图反推提示词、读招牌、读票据时，主要看这个口。   |
| `data`    | `JSON`   | 任务对应的结构化结果。检测任务可返回框坐标，短语定位可返回框和标签，带位置的 OCR 可返回文字及对应区域。              | 给后续节点或脚本做定位、裁剪和数据处理。           |

**四个输出并非每次都有内容。** 例如，选择 `more_detailed_caption` 后，结果在 `caption`。这个版本的 `image` 和 `mask` 此时会返回 `64 × 64` 的全黑占位数据。选择描述式分割时，则主要看 `mask` 和标注预览，`data` 为空。[GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

### 三、节点自身的 9 个可调参数

| 分类     | 参数                 | 默认值           | 具体作用                                                                                                                                                           |
| -------- | -------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 任务选择 | `task`               | `region_caption` | 决定这次是描述图片、找物体、提取遮罩、读文字，还是回答问题。默认任务会检测物体并画框。                                                                             |
| 任务输入 | `text_input`         | 空               | 输入**要找的对象、要定位的短语或要问的问题**。只有 `caption_to_phrase_grounding`、`referring_expression_segmentation`、`docvqa` 接受非空文本；其他模式填了会报错。 |
| 遮罩绘制 | `fill_mask`          | `true`           | 做**框检测**时，开启后把选中矩形填成白色遮罩。做**描述式分割**或 `ocr_with_region` 时，开关控制预览显示色块还是轮廓，输出的 `mask` 仍是填满的。                    |
| 遮罩筛选 | `output_mask_select` | 空               | 在框检测或短语定位任务中，指定哪些结果进入遮罩。留空选全部；`0,2` 选第 1 和第 3 个框，也可填预览中的精确标签。编号从 `0` 开始，逗号后不要加空格。                  |
| 输出长度 | `max_new_tokens`     | `1024`           | 最多生成多少个 token，范围 `1–4096`。长 OCR 或密集检测结果被截断时，可以调高这个上限。                                                                             |
| 候选搜索 | `num_beams`          | `3`              | 生成结果时同时比较多少条候选序列，范围 `1–64`。调高通常更慢，也更占显存。                                                                                          |
| 随机采样 | `do_sample`          | `true`           | 开启后允许随机选择后续 token，描述措辞可能变化。做 OCR 或比较两次结果时，可以关闭它以减少随机性。                                                                  |
| 随机种子 | `seed`               | `1`              | 控制 Florence-2 推理中的随机采样。想对比参数效果时固定它；它和 KSampler 的出图种子各自独立。                                                                       |
| 显存管理 | `keep_model_loaded`  | `false`          | 关闭时，节点在运行后请求卸载模型以腾出显存。连续分析多张图时，开启可减少反复加载。                                                                                 |

这里最容易弄反的是 **`text_input` 的用途**：想让模型详细描述一张图，直接选 `more_detailed_caption`，文本框留空；想让模型找出图里的红外套，才填 `red jacket`。[GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

### 四、`task` 下拉菜单怎么选

| 你想做什么                                       | 选择的 `task`                                                                                                                                            | 主要看哪个输出                            |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| 用一句话描述整张图，或生成更详细的描述           | `caption`、`detailed_caption`、`more_detailed_caption`，依次更详细                                                                                       | `caption`                                 |
| 找物体并画框、为各区域写详细说明、只提出候选区域 | `region_caption`、`dense_region_caption`、`region_proposal`，分别对应这三种目的                                                                          | `image`、`data`                           |
| 找出一句描述中的对象位于哪里                     | `caption_to_phrase_grounding`，在 `text_input` 写描述，例如 `a person holding a red umbrella`                                                            | `image`、`data`，需要矩形遮罩时用 `mask`  |
| 沿指定物体的轮廓生成遮罩                         | `referring_expression_segmentation`，在 `text_input` 写目标，例如 `the red jacket`                                                                       | **`mask`**，用 `image` 检查轮廓           |
| 读取文字，或同时找出文字的位置                   | `ocr` 只读文字；`ocr_with_region` 同时标出文字区域                                                                                                       | `caption`；需要位置时再看 `image`、`data` |
| 根据文档图片回答问题                             | `docvqa`，例如输入 `What is the invoice total?`                                                                                                          | `caption`                                 |
| 生成绘图标签、混合提示词或构图分析               | `prompt_gen_tags` 输出标签；`prompt_gen_mixed_caption` 混合描述与标签；`prompt_gen_analyze` 分析构图；`prompt_gen_mixed_caption_plus` 结合混合描述和分析 | `caption`                                 |

沿用前面**给外套换色**的例子：`task` 选 **`referring_expression_segmentation`**，`text_input` 填 **`the red jacket`**。先用 `image` 预览确认遮罩贴着衣服轮廓，再把 `mask` 接到 `InpaintModelConditioning.mask`，把**原始照片**接到 `pixels`。如果选 `region_caption`，得到的白色矩形还可能罩住外套旁边的手和背景，重绘范围就会偏大。[GitHub](https://github.com/kijai/ComfyUI-Florence2/blob/main/nodes.py?utm_source=chatgpt.com)

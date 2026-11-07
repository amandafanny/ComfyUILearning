以 `comfyui_controlnet_aux` 插件的 **DWPose Estimator** 为例，节点分成三部分：**1 个图片输入、2 个输出、7 个可调参数**。它从参考图里找出人物，估计关节和五官的位置，再把同一个姿势输出成**骨架图**和**坐标数据**。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/utils.py?utm_source=chatgpt.com)

### 一、输入接口：给它什么

| 接口    | 类型    | 接什么                                                                                                                                                                                                     |
| ------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image` | `IMAGE` | 接**提供姿势的人物参考图**。例如想让新角色模仿某人举手，就把那张举手照片从 `Load Image` 接到这里。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/utils.py?utm_source=chatgpt.com) |

### 二、输出接口：从它拿到什么

| 接口            | 输出内容                                                                     | 常接到哪里                                                                                                                                                                                                                         |
| --------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `IMAGE`         | **画出来的姿势**。黑底彩色骨架图，可以直接看到手臂方向、身体姿势和手指位置。 | 接 `Preview Image` 检查骨架，或接 `Apply ControlNet` 的 `image` 输入，用姿势引导生成。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/src/custom_controlnet_aux/dwpose/__init__.py?utm_source=chatgpt.com) |
| `POSE_KEYPOINT` | **姿势的坐标数据**。按人物记录肩、肘、腕、手指、五官等关键点的位置。         | 接 `Save Pose Keypoints` 保存成 OpenPose 格式的 JSON。只想用骨架图控制出图时，这个接口可以留空。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/dwpose.py?utm_source=chatgpt.com)            |

### 三、节点自身的 7 个可调参数

| 分类       | 参数                       | 界面默认值                           | 具体作用                                                                                                 |
| ---------- | -------------------------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| 输出部位   | `detect_body`              | `enable`                             | 保留身体骨架，如肩、肘、腕、髋、膝、踝。想控制站姿、坐姿、抬手等大动作，就开启它。                       |
| 输出部位   | `detect_hand`              | `enable`                             | 保留手部和手指关键点。比如“举手比 V”，手臂举多高主要看 `body`，哪两根手指伸出来主要看 `hand`。           |
| 输出部位   | `detect_face`              | `enable`                             | 保留面部密集关键点，如眼睛、眉毛、嘴和脸部轮廓。适合参考嘴巴张合等面部姿态。                             |
| 找人       | `bbox_detector`            | `yolox_l.onnx`                       | 选择**人体检测模型**。先在画面中框出每个人，确定接下来去哪里找关节。                                     |
| 找关节     | `pose_estimator`           | `dw-ll_ucoco_384_bs5.torchscript.pt` | 选择**姿势估计模型**。它在检测到的人体范围内推算肩、膝、指尖等关键点的位置。                             |
| 骨架图尺寸 | `resolution`               | `512`                                | 设置**输出骨架图的短边**，保持宽高比。例如 `1200 × 800` 的照片会输出 `768 × 512` 的骨架图。              |
| 骨架线粗细 | `scale_stick_for_xinsr_cn` | `disable`                            | 按画面尺寸调整**身体骨架线的粗细**，用来适配 Xinsir 的 OpenPose ControlNet。开启后，关节坐标仍在原位置。 |

以上默认值来自插件的节点界面定义。三个 `detect_*` 可以独立切换。当前实现先估计全身姿势，再按开关筛选输出，最后才按 `resolution` 缩放骨架图。因此，**如果预览里漏了一根手指，把 `resolution` 从 512 改到 1024 只会放大骨架图**。如果预览里的手势已经正确，但生成的人物没有跟随，应该检查下游 `Apply ControlNet` 的 `strength`。[GitHub](https://github.com/Fannovel16/comfyui_controlnet_aux/blob/main/node_wrappers/dwpose.py?utm_source=chatgpt.com)

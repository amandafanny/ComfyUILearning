**`Video Combine` 是 ComfyUI 工作流的导出节点。** 它把一批按顺序排列的图像帧，以指定帧率编码成视频或动图，也可以把口播、音乐或原声合入成片。比如，Wan 生成了 **81 帧**，按 **16 FPS** 导出就是 **5.06 秒**；把导出帧率改成 **24 FPS**，同一批帧只播放 **3.38 秒**，动作会加快。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/videohelpersuite/nodes.py?utm_source=chatgpt.com)

以下按 **ComfyUI-VideoHelperSuite（VHS）** 的 `Video Combine` 说明。它有 **1 个必接输入、3 个可选输入、1 个输出、6 个通用设置**，另外还会根据导出格式显示专属参数。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/videohelpersuite/nodes.py?utm_source=chatgpt.com)

### 一、输入接口

| 输入         | 类型                                  | 是否必接 | 代表什么，通常接哪里                                                                                                                                |
| ------------ | ------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `images`     | `IMAGE`，接兼容 VAE 时也可接 `LATENT` | **必接** | 要合成视频的一批帧，输入顺序就是播放顺序。在前面的 Wan 工作流里，最直接的接法是 **`WanVideo Decode` 的图像输出 → `Video Combine.images`**。         |
| `audio`      | `AUDIO`                               | 可选     | 要合入成片的音轨。保留原声时，接 `Load Video FFmpeg (Upload).audio`；做口播时，接载入 WAV 后的 `AUDIO`。要导出带声音的成片，可选 `video/h264-mp4`。 |
| `meta_batch` | `VHS_BatchManager`                    | 可选     | 长视频分批处理时，接与上游视频读取节点相同的 `Meta Batch Manager`，让各批帧连续写入同一成片。普通短片可以留空。                                     |
| `vae`        | `VAE`                                 | 可选     | 当第一个口接兼容的 latent 时，用这个 VAE 分批解码。若上游已经输出 `IMAGE`，这里留空即可。                                                           |

`image/gif`、`image/webp` 是无声动图格式。这两个导出分支也不支持 `meta_batch`。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/videohelpersuite/nodes.py?utm_source=chatgpt.com)

### 二、输出接口

| 输出        | 类型            | 实际用途                                                                                                                                                                          |
| ----------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Filenames` | `VHS_FILENAMES` | 包含本次导出的位置标记和文件路径列表。列表可能包含首帧 PNG、无声中间文件和最终带声视频。要把**最终成片路径**传给下游，可以接 VHS 的 `Select Filename`，设 `index=-1` 取最后一项。 |

`Video Combine` 自身就会写文件并显示预览，**右侧不接其他节点也能导出**。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/videohelpersuite/nodes.py?utm_source=chatgpt.com)

### 三、节点自身的 6 个通用设置

| 分类     | 参数              | 默认值        | 控制什么                                                                                                                        |
| -------- | ----------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 播放速度 | `frame_rate`      | `8` FPS       | 每秒播放多少张输入帧。**固定帧数下，FPS 越高，视频越短。** 改这个值不会生成中间动作帧。                                         |
| 重复播放 | `loop_count`      | `0`           | **含义取决于格式。** MP4 等普通视频设 `0` 表示只编码一遍，设 `1` 表示额外重复一遍。GIF、动画 WebP 设 `0` 则表示播放器无限循环。 |
| 文件命名 | `filename_prefix` | `AnimateDiff` | 输出文件的前缀。例如填 `wan_dance`，插件会自动加编号和扩展名，如 `wan_dance_00001.mp4`。                                        |
| 导出格式 | `format`          | `image/gif`   | 决定导出动图还是视频，也决定下面出现哪些参数。需要带口播的 MP4 时选 `video/h264-mp4`。                                          |
| 往返播放 | `pingpong`        | `false`       | 开启后先正放，再沿原帧倒放。例如 `A B C D` 会变成 `A B C D C B`。它调整画面顺序，单独接入的音轨不会随之倒放。                   |
| 保存位置 | `save_output`     | `true`        | `true` 写入 ComfyUI 的 `output` 目录；`false` 仍会生成预览文件，但写入 `temp` 临时目录。正式交付成片时选 `true`。               |

处理有原声的视频时，最容易出错的是 `frame_rate`。如果上一节点把 30 FPS 的原片按 **16 FPS** 读入，想维持原来的动作速度，就把这里也设为 **16**。还可以把 `frame_rate` 转成接线输入，接 `Video Info (Loaded)` 的 `fps`，让它自动跟随实际读取帧率。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/videohelpersuite/nodes.py?utm_source=chatgpt.com)

### 四、选 `video/h264-mp4` 后出现的参数

| 参数            | 默认值    | 具体作用                                                                                                           |
| --------------- | --------- | ------------------------------------------------------------------------------------------------------------------ |
| `pix_fmt`       | `yuv420p` | 输出像素格式。`yuv420p` 是兼容性较好的 8 位格式；`yuv420p10le` 是 10 位格式，需要播放器和后续软件支持。            |
| `crf`           | `19`      | 压缩质量。对这个 H.264 预设，**数值越低，通常画质越好、文件越大**。它影响压缩损失，不改变帧数或播放速度。          |
| `save_metadata` | `true`    | 尝试把工作流、提示词等元数据写入输出视频。它控制文件附带的信息，不影响可见画面。                                   |
| `trim_to_audio` | `false`   | 接了音频才有意义。关闭时，通常保留画面长度，短音频结束后补静音。开启时，**音频和画面谁先结束，成片就在哪里结束**。 |

例如，你有一段 **10 秒画面、7 秒口播**：想保留结尾 3 秒无声画面，就关闭 `trim_to_audio`；想让口播一结束视频也结束，就开启它。反过来，只有 **7 秒画面、10 秒口播**时，这个开关不会把画面延长到 10 秒。切换到 `image/webp` 后，界面则会显示 `lossless`（默认开启）来控制无损压缩。[GitHub](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite/blob/main/video_formats/h264-mp4.json?utm_source=chatgpt.com)

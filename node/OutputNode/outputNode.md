| 常见输出节点                          | 接收的数据                          | 作用                                     |
| ------------------------------------- | ----------------------------------- | ---------------------------------------- |
| **Save Image／保存图像**              | `IMAGE`                             | 把图片保存到输出目录，并显示结果         |
| **Preview Image／预览图像**           | `IMAGE`                             | 显示图片，使用临时文件供预览             |
| **Save Latent／保存潜在表示**         | `LATENT`                            | 保存潜在数据，方便以后加载、继续处理     |
| **Save Video／保存视频**              | `VIDEO`                             | 将视频保存为文件                         |
| **Video Combine（VideoHelperSuite）** | 通常是多帧 `IMAGE`，可附加音频      | 将图像序列合成为视频或动图，供预览或保存 |
| **Preview as Text / Preview Any**     | 在节点里查看、复制提示词            | ComfyUI 内置                             |
| **Save Text**                         | 把提示词保存为 `.txt`、`.md` 等文件 | 较新版 ComfyUI 内置                      |
| **Show Text 🐍**                      | 显示提示词，也能继续传给后续节点    | ComfyUI-Custom-Scripts 插件              |

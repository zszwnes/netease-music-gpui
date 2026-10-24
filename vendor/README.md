# GPUI 本地补丁

以下为 GPUI 补丁的准备及维护说明。

这里跟踪 `patches/` 下的补丁，以及 `gpui-pre-windows/` 这一份已打好补丁的源码（原因见下）。
补丁从原 stash 的 GPUI 0.3.7 实验移植到当前使用的 **0.3.8**。

- `gpui-image-transform-0.3.8.patch`：图片 GPU 旋转。
- `gpui-backdrop-blur-0.3.8.patch`：圆角矩形背景模糊（毛玻璃）。
- `gpui-svg-app-fonts-0.3.8.patch`：SVG 文本加载 `assets/font/` 中的应用字体，用于推荐页日历等图案。

首次构建或补丁更新后，在项目根目录运行：

```bash
python3 scripts/prepare_gpui.py
cargo run --locked
```

准备脚本从 crates.io 下载固定版本的 `gpui-pre`、`gpui-pre-apple`、`gpui-pre-wgpu`，
校验 SHA-256，并按文件名顺序使用 `git apply` 应用 `patches/` 下的全部补丁。
这三个生成的源码目录已加入 `.gitignore`；
根目录 Cargo 的 `[patch.crates-io]` 使用这些本地包，使直接和间接依赖共用相同版本。
原始发布包中的 Apache-2.0 许可证随源码保留。

`gpui-pre-windows/` 是唯一随仓库提交的依赖源码。CI 的缓存键只看 `patches/` 的哈希，
缓存命中时会跳过准备脚本，届时若该目录不存在，cargo 会因为 `[patch.crates-io]` 指向空目录
而直接报 `patch location ... does not contain packages`。把这份已打好补丁的源码纳入版本控制后，
无论缓存是否命中，Windows 构建都能拿到 `PrimitiveBatch::BackdropBlurs` 分支。
内容来自 gpui-pre-windows 0.3.8 发布包外加一个补丁（见文件内注释），换行统一为 LF；
升级 GPUI 时需要重新生成这一份，而不是再给它加 `.patch`。

脚本可重复运行；版本与补丁未变时直接跳过。需要修改补丁时，应编辑 patch，
不要只修改生成的源码，因为下一次准备会替换这些目录。

GPU 旋转接口：

```rust
img(url).with_transformation(Transformation::rotate(radians(angle)))
```

图片仍复用 GPUI 原有解码和纹理缓存，每帧只改变矩阵。布局和 hitbox 不变，
圆角在图片局部坐标中计算，祖先裁剪仍使用窗口坐标。
图片的线性采样限制在自身图集区域的像素中心范围内，避免旋转时混入相邻图片的颜色。
补丁同步调整了 Scene 包围盒、Metal/WGSL 绘制路径和 WebGL 的记录步长。
Metal 构建脚本从同级 `gpui-pre` 生成共享结构，避免 Rust 与 shader 布局不一致。

升级补丁后先运行 `cargo check --locked --all-targets`，再手动启动应用，检查旋转方向、圆形裁剪和背景衔接。GPU 绘制效果需要在对应平台实际查看。

背景模糊接口：

```rust
window.paint_backdrop_blur(bounds, px(12.), px(20.).into());
```

它把当前帧已经画好的内容拷到一张纹理，做 17×17 高斯采样后连同圆角遮罩画回原位置，
因此只影响之后绘制的元素（按钮、卡片自身的底色照常叠在上面）。
Metal 后端每帧把 drawable 拷进一张带 mip 链的纹理：

- 采样点间距随半径增长（`radius / 4`）。一直读 level 0 时每个采样点只覆盖一个纹素，
  间距超过一个纹素后就会把背景的高频内容混叠成可见的万格栅格。
- 因此按 `log2(间距)` 选 mip 级别，让每个采样点平均一個和间距同样宽的格子。
- 采样器必须写 `mip_filter::linear`：`mip_filter::none` 会让采样器停留在“非 mipmap”
  状态，片元里的 `level()` 会被忽略、永远返回 level 0，模糊就退回栅格。
- 需要 `layer.set_framebuffer_only(false)` 才能读取 drawable，这是该接口开启的。

WGPU 后端目前忽略该批次（Windows/Linux 上不绘制模糊）；需要在其他平台使用时照样补一个实现。
Windows 的 DirectX 后端同样跳过该批次（`vendor/gpui-pre-windows/src/directx_renderer.rs`）。

升级 GPUI 时，更新准备脚本中的版本与发布包 SHA-256，并重新生成、验证 patch。
如果上游提供同等接口，可删除补丁、准备脚本和 Cargo 中的路径覆盖。

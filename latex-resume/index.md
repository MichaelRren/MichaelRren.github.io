# 基于 Latex 生成中文简历

# 使用 Latex 编写模版的优势
1. 对于 `.doc` 文件而言，在不同的机器或不同的版本的 `Office` 打开都会可能使**文档内容偏移**。
2. 即使同一版本的 `Office`，在不同的机器上，打印的字体都可能不同，例如有些纯英文机器上缺少支持的中文字体等。
3. 携带不方便，由于每次都要基于最新的版本进行修改，因此需要时刻携带最新版本简历。
4. 快速修改简历。针对不同岗位，不同的要求，可以在云端快速进行修改，保存多个版本。

因此，通过 `Overleaf` 平台，使用基于 Latex 的简历模版，可以做到在线保存最新版本的简历，且生成的 `.pdf` 文件是一致的，不会因为**使用环境原因导致输出的文件与之前的不一致**，无需调整。

# 模版样式
<center>{{< figure src="images/resume.png" width="50%">}}</center>

# 项目地址
[仓库链接](https://github.com/MichaelRren/resume-zh)

# 使用方法
1. github 上以 zip 格式下载此项目
2. 将 zip 导入 Overleaf 中
3. 在 Overleaf 页面点击 Menu bar，修改 Compiler，从 pdfLaTex 变为 XeLaTex（支持中文）
4. 修改 resume.tex 文件
5. 完成后进行 recompile

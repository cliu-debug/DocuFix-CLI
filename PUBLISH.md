# DocuFix 发布指南

为了让您的项目正式面世，您需要完成 **GitHub 代码托管** 和 **PyPI 包发布** 两个步骤。

## 1. 发布到 GitHub (代码开源)

1. **初始化库** (如果在本地还没做的话):
   ```bash
   git init
   git add .
   git commit -m "feat: initial open source release v0.1.0"
   ```

2. **在 GitHub 上创建仓库**:
   - 访问 [github.com/new](https://github.com/new)
   - 仓库名设为 `DocuFix-CLI`
   - 不要勾选 "Initialize this repository with a README"

3. **推送到 GitHub**:
   ```bash
   git remote add origin https://github.com/您的用户名/DocuFix-CLI.git
   git branch -M main
   git push -u origin main
   ```

---

## 2. 发布到 PyPI (支持 pip install)

1. **安装打包工具**:
   ```bash
   pip install --upgrade build twine
   ```

2. **构建分发包**:
   在项目根目录下运行：
   ```bash
   python -m build
   ```
   这将生成 `dist/` 目录，包含 `.whl` 和 `.tar.gz` 文件。

3. **上传到 PyPI**:
   - 首先在 [pypi.org](https://pypi.org/account/register/) 注册账号。
   - 然后运行：
     ```bash
     python -m twine upload dist/*
     ```
   - 输入您的 PyPI 令牌 (Token) 即可完成发布。

---

## 3. 发布后的仪式感

发布成功后，您可以：
- 运行 `python -m src.cli scan <URL>` 获取您的 **GEO Score Badge**。
- 将生成的 Badge 代码贴在 GitHub 的 `README.md` 顶部。
- 在 Twitter/X 或开发者社区宣布 **DocuFix** 的诞生！

🚀 **祝 DocuFix 的开源之旅大获成功！**

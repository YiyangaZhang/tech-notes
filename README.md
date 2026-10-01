# tech-notes

公开技术学习笔记 —— 一个 **learn in public** 的知识库：把学习过程中的理解、实验与整理公开记录下来，欢迎交流、勘误与探讨。笔记会持续修订，不保证每篇都完美；发现问题欢迎通过 Discussions / Issues 反馈。

## 关于

一名软件工程师的公开笔记，日常关注 AI / LLM 应用、云基础设施与数据工程。这里只放可公开的通用知识与学习材料；工作相关的私有内容不在这里。

## 笔记导航

### AI / LLM

| 笔记 | 说明 |
|---|---|
| [FDE 角色指南：前沿部署工程师做什么](ai/fde-role-guide.md) | Forward Deployed Engineer 的角色定义、与 SWE/SA/PM/QE 的边界、AI Agent 工程边界、能力要求与转型/面试甄别清单 |
| [AutoGluon Demo（Jupyter Notebook）](ai/ml-autogluon-demo.ipynb) | AutoGluon 自动化机器学习的实验笔记本（持续补充中） |

### AWS / 云基础设施

| 笔记 | 说明 |
|---|---|
| [AWS 网络基础](aws/networking.md) | Route Table、ACL / Security Group、IGW / VPG 等 VPC 网络基础概念（含示意图） |

### 半导体 / 行业资料

| 资料 | 说明 |
|---|---|
| [STDF V4 规范（2007）](semiconductor/stdf_V4-2007_spec.pdf) | 半导体测试数据格式 STDF（Standard Test Data Format）V4 官方规范 PDF，行业公开标准资料 |

## 如何交流

- **提问、求解释、想法与选题探讨** → [Discussions](https://github.com/YiyangaZhang/tech-notes/discussions)
- **事实错误、typo、链接失效、代码问题** → [开一个 Issue](https://github.com/YiyangaZhang/tech-notes/issues/new/choose)，注明笔记路径、问题位置与建议修改
- 详细说明见 [CONTRIBUTING.md](CONTRIBUTING.md)

## 组织约定

- 目录按主题划分（`ai/`、`aws/`、`semiconductor/`），文件名使用小写 kebab-case。
- 仓库内跳转均为相对链接，在 VSCode 中 `Ctrl+Click`（或预览中点击）会以新编辑器页签打开。

## License

本仓库采用**双许可**：

- **文字内容（Markdown 笔记、文档）**：[Creative Commons Attribution-ShareAlike 4.0 International](LICENSE)（CC BY-SA 4.0）——可自由转载与改编，需署名并以相同许可共享衍生作品。
- **代码（Notebook、脚本及可独立运行的代码片段）**：[MIT License](LICENSE-MIT)。

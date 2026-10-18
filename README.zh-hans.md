> 其他语言：
>
> - [English](/README.md)
> - [Korean (한국어)](/README.ko.md)

# E-Viewer

[![CI](https://github.com/OpportunityLiu/E-Viewer/actions/workflows/ci.yml/badge.svg)](https://github.com/OpportunityLiu/E-Viewer/actions/workflows/ci.yml)

适用于 Windows 10（1809 及以上）和 Windows 11 的 E-Hentai / ExHentai UWP 客户端。

[![](https://raw.github.com/wiki/OpportunityLiu/E-Viewer/Images/Screenshots/1.png)](https://github.com/OpportunityLiu/E-Viewer/wiki)
[更多截图](https://github.com/OpportunityLiu/E-Viewer/wiki/Home)

## 发布

在 [GitHub Releases](https://github.com/OpportunityLiu/E-Viewer/releases) 下载发布版本。

[![Github All Releases](https://img.shields.io/github/downloads/OpportunityLiu/E-Viewer/total.svg)](https://github.com/OpportunityLiu/E-Viewer/releases) [![Github Latest Release](https://img.shields.io/github/downloads/OpportunityLiu/E-Viewer/latest/total.svg)](https://github.com/OpportunityLiu/E-Viewer/releases/latest)

发布包为 `.msix` 格式。每个架构还提供一个 zip，内含安装包、依赖、证书和 `Install.ps1`。如果直接安装 `.msix`，请从对应架构的 zip 中安装依赖，并先信任 `.cer` 证书。

`.appxsym` 是用于调试的符号文件。

网页登录依赖 Microsoft Edge WebView2 运行时。Windows 11 已内置；Windows 10 如果缺失，请安装 Evergreen WebView2 运行时。Cookie 登录不需要 WebView2。

## 常见问题

1. [如何安装](https://github.com/OpportunityLiu/E-Viewer/wiki/How-to-Install)

2. [如何解决 UWP 无法使用代理的问题](https://github.com/OpportunityLiu/E-Viewer/wiki/Resolve-Connection-Issues)

3. [如何使用键盘或 Xbox 控制器操作](https://github.com/OpportunityLiu/E-Viewer/wiki/Tips)

4. 如果网页登录提示 WebView2 运行时不可用，请安装 Evergreen WebView2 运行时，或改用 Cookie 登录。

## 编译

以下内容面向希望自行编译本项目的开发者。

### 前置条件

- 安装了“通用 Windows 平台开发”工作负载的 Visual Studio 2022
- Windows SDK 10.0.22621.0（最低支持 Windows 10 build 17763）

### 步骤

- 将仓库克隆到本地
- 打开 `ExViewer.sln`
- 还原 NuGet 包
- 使用 [Opportunity.ResourceGenerator](https://github.com/OpportunityLiu/ResourceGenerator) 生成资源文件
  （CI 中执行 `Opportunity.ResourceGenerator.Generator.exe ./ExViewer.sln`）
- 编译并运行

CI 使用 `windows-2022`，因为 `windows-latest` 已不再提供 UWP 工具链。

## 翻译

如果希望帮助翻译本项目，可以 fork 仓库，修改 `**/Strings/` 下的资源文件并提交 pull request。

也可以像[简体中文](/README.zh-hans.md)版本一样，提供 README 的翻译。

# CardFightRogue-CI

[CardFightRogue](https://github.com/CardFightRogue/CardFightRogue)（私有仓库）的自动测试。

这个仓库是公开的，所以 GitHub Actions 的运行分钟数不限。它本身不含游戏代码：每次运行时拉取私有仓库的代码，安装 Godot 4.7.2，无界面运行三个测试（内容校验、规则测试、冒烟测试）。

> **运行日志是公开的。** 任何人都能在 Actions 页面看到测试输出，其中会出现私有仓库里的文件路径、测试名称和报错信息。不要在测试输出里打印不想公开的内容。

## 访问私有仓库的方式：deploy key

工作流用一把专用的 SSH 密钥拉取私有仓库的代码，已经配置好，平时不用管：

- 公钥登记在私有仓库的 Settings → Deploy keys，名称“CardFightRogue-CI（只读）”。它**只能读这一个仓库**，不能写，也访问不了别的仓库。
- 私钥存在本仓库的密钥 `PRIVATE_REPO_DEPLOY_KEY` 里（Settings → Secrets and variables → Actions）。存入后任何人都读不出来，只有工作流运行时能用。
- 它不绑定任何个人账号，也不会过期。

### 需要更换密钥时

怀疑密钥泄露，或者想定期轮换时：

```bash
ssh-keygen -t ed25519 -N "" -C "CardFightRogue-CI deploy key" -f ci_key
```

```bash
gh repo deploy-key add ci_key.pub --repo CardFightRogue/CardFightRogue --title "CardFightRogue-CI（只读）"
```

```bash
gh secret set PRIVATE_REPO_DEPLOY_KEY --repo CardFightRogue/CardFightRogue-CI < ci_key
```

然后删除本地的 `ci_key` 和 `ci_key.pub`，并在私有仓库的 Deploy keys 页面删掉旧的那把。

组织需要允许仓库使用 deploy key（组织 Settings 里的 Deploy keys 选项），否则添加公钥时会报“Deploy keys are disabled for this repository”。

## 运行

- **手动**：Actions → 测试 → Run workflow，可以填要测试的分支或提交。
- **定时**：每天北京时间 6:00 自动测试 `main`。
- **命令行**：

  ```bash
  gh workflow run 测试 --repo CardFightRogue/CardFightRogue-CI -f ref=main
  ```

## 导出试玩版

手动触发“导出试玩版”，会导出 Windows 和 Linux 两个版本，并实际启动一次 Linux 版确认没有报错。

- **网页**：Actions → 导出试玩版 → Run workflow，跑完后在这次运行页面底部的 Artifacts 里下载。
- **命令行**：

  ```bash
  gh workflow run build.yml --repo CardFightRogue/CardFightRogue-CI -f ref=main
  ```

  下载最近一次导出的 Windows 版到当前目录：

  ```bash
  gh run download --repo CardFightRogue/CardFightRogue-CI --pattern "CardFightRogue-windows-*"
  ```

Windows 版解压后双击 `CardFightRogue.exe` 即可；同目录下的 `CardFightRogue.console.exe` 是带控制台窗口的版本，排查问题时用。产物保留 14 天。

> **产物是公开的。** 任何登录了 GitHub 的人都能下载，而导出的游戏包里包含全部脚本和数据，可以被解包。

## 推送后自动运行（可选）

想让私有仓库每次推送后自动测试，需要在私有仓库里加一个很小的工作流，它只负责通知这个仓库（每次占用私有仓库几秒钟的运行时间）。

1. 打开 <https://github.com/settings/personal-access-tokens/new> 创建一个 fine-grained 令牌：Resource owner 选 `CardFightRogue` 组织，Repository access 只选 `CardFightRogue-CI`，Permissions 选 **Contents：Read and write**（触发 `repository_dispatch` 需要这个权限）。deploy key 只能用来拉代码，不能调用这个接口，所以这一步仍然要用令牌。
2. 存为**私有仓库**的密钥 `CI_DISPATCH_TOKEN`：

   ```bash
   gh secret set CI_DISPATCH_TOKEN --repo CardFightRogue/CardFightRogue
   ```

3. 在私有仓库新建 `.github/workflows/notify-ci.yml`：

   ```yaml
   name: 通知 CI
   on:
     push:
       branches: [main]
   jobs:
     notify:
       runs-on: ubuntu-latest
       steps:
         - run: |
             gh api repos/CardFightRogue/CardFightRogue-CI/dispatches \
               -f event_type=private-push \
               -f "client_payload[ref]=${{ github.sha }}"
           env:
             GH_TOKEN: ${{ secrets.CI_DISPATCH_TOKEN }}
   ```

## 更换 Godot 版本

改 `.github/workflows/tests.yml` 顶部的 `GODOT_VERSION`。

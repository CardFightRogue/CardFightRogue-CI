# CardFightRogue-CI

[CardFightRogue](https://github.com/CardFightRogue/CardFightRogue)（私有仓库）的自动测试。

这个仓库是公开的，所以 GitHub Actions 的运行分钟数不限。它本身不含游戏代码：每次运行时拉取私有仓库的代码，安装 Godot 4.7.2，无界面运行三个测试（内容校验、规则测试、冒烟测试）。

> **运行日志是公开的。** 任何人都能在 Actions 页面看到测试输出，其中会出现私有仓库里的文件路径、测试名称和报错信息。不要在测试输出里打印不想公开的内容。

## 第一次使用：配置访问令牌

工作流需要一个能读取私有仓库的令牌。只需配置一次。

1. 打开 <https://github.com/settings/personal-access-tokens/new>，创建一个 fine-grained 令牌：
   - **Resource owner**：选 `CardFightRogue` 组织
   - **Repository access**：Only select repositories → 只选 `CardFightRogue`
   - **Permissions** → Repository permissions → **Contents：Read-only**（其他都不用）
   - 有效期按需要设置，到期后要重新生成并更新
2. 把令牌存为本仓库的密钥 `PRIVATE_REPO_TOKEN`。在命令行运行下面的命令，然后粘贴令牌：

   ```bash
   gh secret set PRIVATE_REPO_TOKEN --repo CardFightRogue/CardFightRogue-CI
   ```

   或者在网页上：本仓库 Settings → Secrets and variables → Actions → New repository secret。

如果组织开启了“fine-grained 令牌需要管理员批准”，还要在组织的 Settings → Personal access tokens 里批准这个令牌。

## 运行

- **手动**：Actions → 测试 → Run workflow，可以填要测试的分支或提交。
- **定时**：每天北京时间 6:00 自动测试 `main`。
- **命令行**：

  ```bash
  gh workflow run 测试 --repo CardFightRogue/CardFightRogue-CI -f ref=main
  ```

## 推送后自动运行（可选）

想让私有仓库每次推送后自动测试，需要在私有仓库里加一个很小的工作流，它只负责通知这个仓库（每次占用私有仓库几秒钟的运行时间）。

1. 再创建一个 fine-grained 令牌：Repository access 只选 `CardFightRogue-CI`，Permissions 选 **Contents：Read and write**（触发 `repository_dispatch` 需要这个权限）。
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

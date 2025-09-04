# Git 入门指南 (小白轻松学) 🚀



Git 是一款强大的分布式版本控制系统，简单来说，它可以帮你追踪文件的每一次修改，方便你回到过去任何一个版本，也方便团队协作。GitHub 则是一个基于 Git 的代码托管平台，你可以把本地的 Git 仓库同步到 GitHub 上。

---

## 一、Git 安装与初次配置

1.  **下载与安装 Git:**
    * 访问 [Git 官网](https://git-scm.com/downloads) 下载对应你操作系统的安装包，一路下一步默认安装即可。
    * 安装完成后，在命令行 (Windows 用户推荐 Git Bash，随 Git 一起安装) 或终端 (macOS/Linux) 输入 `git --version`，如果能看到版本号，说明安装成功。

2.  **初次配置用户信息 (非常重要！):**
    打开命令行/终端，输入以下命令，替换成你自己的名字和邮箱。这些信息会出现在你的每一次提交记录中。
    ```bash
    git config --global user.name "你的名字"
    git config --global user.mail "你的邮箱@example.com"
    ```
    你可以用以下命令检查配置是否成功：
    ```bash
    git config --list
    ```

---

## 二、Git 核心概念与本地仓库操作

### 核心概念

* **工作区 (Working Directory):** 你电脑上能看到的项目文件夹。
* **暂存区 (Staging Area / Index):** 一个临时存放你改动过文件的区域，只有进入暂存区的文件才能被提交到仓库。
* **本地仓库 (Local Repository):** 保存了项目所有版本历史记录的地方，在你的电脑上 (项目文件夹下的 `.git` 隐藏文件夹)。
* **远程仓库 (Remote Repository):** 托管在网络服务器上的仓库，如 GitHub。

### 常用指令与作用

1.  **`git init`**: **初始化仓库**
    * **作用：** 在当前项目文件夹下创建一个新的 Git 本地仓库。执行后会生成一个隐藏的 `.git` 文件夹。
    * **用法：**
        ```bash
        cd /path/to/your/project  # 进入你的项目文件夹
        git init
        ```

2.  **`git status`**: **查看仓库状态**
    * **作用：** 显示工作区和暂存区中文件的状态（哪些文件被修改了、哪些文件在暂存区等）。这是最常用的命令之一！
    * **用法：**
        ```bash
        git status
        ```

3.  **`git add <文件名>` 或 `git add .`**: **添加到暂存区**
    * **作用：** 将工作区的修改添加到暂存区。
    * **用法：**
        * 添加指定文件: `git add readme.txt`
        * 添加所有已修改或新增的文件 (不包括已删除的文件): `git add .`
        * 添加所有变化 (包括已删除的文件): `git add -A`

4.  **`git commit -m "提交说明"`**: **提交到本地仓库**
    * **作用：** 将暂存区的所有内容提交到本地仓库，形成一个新的版本。`-m` 后面跟的是本次提交的说明，务必清晰明了。
    * **用法：**
        ```bash
        git commit -m "新增了用户注册功能"
        ```

5.  **`git log`**: **查看提交历史**
    * **作用：** 显示从近到远的提交历史记录，包括提交ID (commit hash)、作者、日期和提交说明。
    * **用法：**
        ```bash
        git log
        git log --oneline # 以更简洁的单行方式显示
        git log --oneline -5  #显示最近五条
        ```

6.  **`git diff <文件名>`**: **查看文件修改内容**
    * **作用：** 显示工作区文件与暂存区文件之间的差异，或者工作区文件与上一次提交之间的差异（如果文件未添加到暂存区）。
    * **用法：**
        * 查看某个文件具体修改了什么: `git diff readme.txt`
        * 查看所有未暂存的修改: `git diff`

7.  **`git checkout -- <文件名>` 或 `git restore <文件名>`**: **撤销工作区修改**
    * **作用：** 撤销工作区中对某个文件的修改，使其恢复到上一次提交或暂存的状态 (慎用！修改会丢失)。
    * **用法：**
        ```bash
        git checkout -- readme.txt  # 旧版 Git
        git restore readme.txt      # 新版 Git (推荐)
        ```

8.  **`git reset HEAD <文件名>` 或 `git restore --staged <文件名>`**: **取消暂存**
    * **作用：** 将暂存区的文件移回工作区，即撤销 `git add` 操作。
    * **用法：**
        ```bash
        git reset HEAD readme.txt  # 旧版 Git
        git restore --staged readme.txt # 新版 Git (推荐)
        git reset --hard HEAD^  #回退到上一个版本
        git reset --hard 提交哈希值 #回退到指定版本
        ```

9.  **`git branch`**: **分支管理**

    * **作用：** 列出、创建或删除分支。分支可以让你在不影响主线（通常是 `main` 或 `master` 分支）的情况下开发新功能。
    * **用法：**
        * 列出所有本地分支: `git branch`
        * 创建一个新分支: `git branch new-feature`
        * 切换到指定分支: `git checkout new-feature` (旧版) 或 `git switch new-feature` (新版，推荐)
        * 创建并立即切换到新分支: `git checkout -b new-feature` (旧版) 或 `git switch -c new-feature` (新版，推荐)
        * 删除分支 (通常在合并后): `git branch -d branch-name` (如果未合并会提示，用 `-D` 强制删除)

10.  **`git merge <分支名>`**: **合并分支**
     * **作用：** 将指定分支的修改合并到当前所在的分支。
     * **用法：** (假设当前在 `main` 分支，想合并 `new-feature` 分支)

         ```bash
         git switch main
         git merge new-feature
         
         ```

11.  **恢复误操作**

     ```bash
     git reflog
     git reset --hard <desired-commit-hash>
     ```

     

     

---

## 三、关联并操作 GitHub 远程仓库

1.  **在 GitHub 上创建远程仓库：**
    * 登录 GitHub ([https://github.com](https://github.com))。
    * 点击页面右上角的 "+" 号，选择 "New repository"。
    * 填写仓库名 (Repository name)，建议和本地项目文件夹名一致。
    * 选择公开 (Public) 或私有 (Private)。
    * **重要：不要**勾选 "Initialize this repository with a README"、".gitignore" 或 "license"，因为我们本地已经有了 (或者稍后创建)。
    * 点击 "Create repository"。

2.  **将本地仓库与远程仓库关联：**
    * 创建成功后，GitHub 会显示一个仓库地址，通常是 HTTPS 或 SSH 格式。复制 HTTPS 地址 (对新手更友好)。
    * 回到你的本地项目命令行/终端，执行：
        ```bash
        git remote add origin <你复制的GitHub仓库HTTPS地址>
        ```
        这里的 `origin` 是远程仓库的默认别名，你可以自定义，但通常用 `origin`。
    * 验证关联是否成功：
        ```bash
        git remote -v
        ```
        应该能看到 `origin` 指向你 GitHub 仓库的 fetch 和 push 地址。

3.  **`git push -u origin <本地分支名>`**: **推送本地提交到远程仓库**
    * **作用：** 将本地仓库的提交推送到 GitHub 远程仓库。`-u` 参数会将本地分支与远程分支关联起来，以后推送可以直接用 `git push`。
    * **用法：** (假设你想推送本地的 `main` 分支)
        先创建main分支
        ```bash
         git branch -M main 
        ```
        ```bash
        git push -u origin main
        ```
        首次推送可能需要你输入 GitHub 的用户名和密码 (或 Personal Access Token，如果启用了双因素认证)。

4.  **`git pull origin <远程分支名>`**: **从远程仓库拉取更新**
    * **作用：** 获取远程仓库的最新版本并与本地分支合并。这在你与他人协作或者在多台电脑上工作时非常重要。
    * **用法：** (假设你想拉取远程 `main` 分支的更新到本地当前分支)
        ```bash
        git pull origin main
        ```
        如果本地分支与远程分支已关联 (通过 `git push -u` 或 `git branch --set-upstream-to`)，可以直接使用 `git pull`。

5.  **`git clone <远程仓库地址>`**: **克隆远程仓库到本地**
    * **作用：** 如果你想获取一个已存在于 GitHub 上的项目，可以用这个命令将其完整复制到你的本地电脑，自动建立本地仓库和远程仓库的关联。
    * **用法：**
        ```bash
        git clone <GitHub仓库的HTTPS或SSH地址>
        ```

---

## 四、多项目整理

1. **删除外层仓库的.git（保留代码）**

```bash
rm -rf .git
```



2. **重新初始化并添加子模块**

```bash
	git init
	git submodule add <仓库A的GitURL>
	git submodule add <仓库B的GitURL>
```



3. **提交变更**

```bash
git commit -m "添加子模块"
```

## 五、总结与后续学习

这只是 Git 和 GitHub 的入门基础。Git 非常强大，还有很多高级功能如变基 (`rebase`)、标签 (`tag`)、储藏 (`stash`) 等待你去探索。

**建议：**

* **多练习！** 亲手操作是最好的学习方式。
* 遇到问题多使用 `git status` 和 `git log` 查看状态。
* 查阅官方文档或优质教程。

祝你 Git 使用愉快！🎉

## 当前电脑配置

- 邮箱

  - > 15188636576@163.com

- 用户名

  - > huazn

- 查看方法

  ```cmd
  git config --global user.email  # 查看邮箱
  git config --global user.name   # 查看用户名
  ```

  

## 克隆相关

- 设置代理

> git config --global http.proxy 127.0.0.1:10809
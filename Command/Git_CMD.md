# Git 常用指令

### 配置用户信息

- 设置用户名，将 “Your Name” 替换：

```bash
git config --global user.name "Your Name"
```

- 设置用户邮箱，将 “your_email@example.com” 替换：

```bash
git config --global user.email "your_email@example.com"
```

- 生成 SSH 密钥对，一路回车使用默认设置即可, 这里的邮箱地址建议与前面配置的用户邮箱一致：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

密钥生成后，默认保存在`C:\Users\你的用户名\.ssh`目录下，其中`id_rsa`是私钥，`id_rsa.pub`是公钥。  若要将生成的 SSH 密钥添加到 GitHub 或其他代码托管平台，需将公钥内容复制并粘贴到平台的 SSH 密钥设置中。使用以下命令快速复制公钥内容：

```bash
clip < ~/.ssh/id_rsa.pub
```

然后登录到代码托管平台，找到 SSH 密钥设置页面，粘贴公钥内容并保存。



### 1. 在 GitHub 创建仓库

1. 登录到 GitHub 网站（[https://github.com/](https://github.com/) ）。
2. 点击页面右上角的 “+” 号，选择 “New repository”。
3. 在 “Create a new repository” 页面：
   - 填写仓库名称（Repository name）。
   - 可添加仓库描述（Description），这一步可选。
   - 选择仓库的可见性，通常有 “Public”（公开，任何人都能看到）和 “Private”（私有，只有你和协作者能看到，私有仓库可能需要付费套餐）。
   - 如果你希望初始化仓库时带有一个 `README` 文件、`.gitignore` 文件或 `LICENSE` 文件，可以勾选相应的选项。一般建议先创建一个简单的 `README` 文件，方便他人了解项目。
4. 完成设置后，点击 “Create repository” 创建仓库。

### 2. 初始化本地项目为 Git 仓库

1. 打开 Git Bash 或 Windows 命令提示符，导航到你本地项目所在的目录。例如，如果项目在 `C:\myproject` 目录下，在命令行中输入：

```bash
cd C:\myproject
```

2. 初始化一个新的 Git 仓库，在项目目录下运行以下命令：

```bash
git init
```

这会在项目目录下创建一个隐藏的 `.git` 文件夹，用于跟踪项目的版本历史。

### 3. 将本地项目文件添加到暂存区并提交

1. 使用以下命令将项目中的所有文件添加到暂存区：

```bash
git add.
```

这里的 “.” 表示当前目录下的所有文件和子目录。如果你只想添加特定文件，可以将 “.” 替换为具体的文件名或目录名。 

2. 执行提交操作，将暂存区的文件提交到本地仓库，并添加一条提交信息描述本次提交的内容。例如：

```bash
git commit -m "Initial commit"
```

将 `"Initial commit"` 替换为你自己的有意义的提交信息，简洁描述本次提交做了哪些更改。

### 4. 添加远程仓库地址

1. 在 GitHub 上创建好的仓库页面，找到 “HTTPS” 或 “SSH” 克隆链接。如果选择 “HTTPS” 方式（较为常用，不需要额外配置 SSH 密钥，但每次推送代码时可能需要输入用户名和密码），复制该链接。
2. 在本地项目的 Git Bash 或命令提示符中，运行以下命令添加远程仓库地址，将 `<repository-url>` 替换为你复制的 GitHub 仓库链接：

bash

```bash
git remote add origin <repository-url>
```

这里 “origin” 是远程仓库的默认名称，你也可以使用其他名称，但 “origin” 是一个广泛使用的约定。

### 5. 将本地仓库内容推送到 GitHub 远程仓库

1. 如果是首次推送，并且你使用的是 HTTPS 链接，现在的 Git 默认分支名称可能是 `main`，根据实际情况调整，推送到远程仓库的 `main` 分支：

```bash
git push -u origin main
```

如果使用 SSH 链接且已正确配置 SSH 密钥，同样运行上述命令。这一步会要求你输入 GitHub 的用户名和密码（如果使用 SSH 则无需输入密码）。`-u` 参数表示将本地分支与远程分支建立关联，以后推送和拉取时就可以简化命令。  

2. 后续推送代码时，如果没有更改远程仓库地址和分支名称，只需运行：

```bash
git push
```

这样就可以将本地的更改推送到 GitHub 远程仓库了。

如果在推送过程中遇到冲突（例如其他人先推送了更改），你需要先拉取最新的更改（`git pull origin master`），解决冲突后再重新提交并推送。

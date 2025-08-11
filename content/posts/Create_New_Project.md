---
date: '2025-08-11T20:08:00+08:00'
draft: false
title: 'Create New Project' # <--- 修改这一行
summary: "带虚拟环境的 Python 项目"
---


**创建项目文件夹**

打开命令行工具（比如 CMD、PowerShell 或者 Windows Terminal），然后进入存放所有代码的目录（例如 D:\projects）。

```
# 1. 进入项目的父目录(请根据实际情况修改 "D:\projects")
cd D:\projects

2. 创建新文件夹作为根目录
mkdir my_new_project

3. 进入这个新创建的文件夹
cd my_new_project
```


**创建虚拟环境**

```
# 确保在项目根目录下 (D:\projects\my_new_project)
# -m venv 的意思是 "以模块(module)方式运行 venv"
# 最后的 "venv" 是给虚拟环境文件夹取的名字
python -m venv venv

# 执行完毕后，会发现项目文件夹里多出了一个名为 venv 的子文件夹。这里面包含了独立的 Python 解释器和未来将要安装的库。
```

**激活虚拟环境**
```
# 在 Windows PowerShell 中，运行以下命令
.\venv\Scripts\activate
```
```
激活成功后，会看到命令行提示符的前面多了一个 (venv) 的标记，像这样：
(venv) PS D:\projects\my_new_project>
```
这个 (venv) 标志非常重要，它告诉你当前终端已经处于激活的虚拟环境中。所有后续的 pip 安装和 python 命令都将只在这个环境内生效。
```
# 在 VS Code 中打开项目,确保还在项目根目录下 (有 (venv) 标志),这个命令会用 VS Code 打开当前文件夹
code .
``` 

**配置 VS Code 解释器**

1.当用 VS Code 打开带有 venv 文件夹的项目时，VS Code 通常会在右下角弹出一个提示：```We noticed a new environment has been created. Do you want to select it for the workspace folder? ```(我们发现了一个新环境，您要为工作区选择它吗？)。

请务必点击``` Yes```。

2.如果 VS Code 没有自动提示，可以手动选择：

按下 ```Ctrl+Shift+P``` 打开命令面板。

输入并选择``` Python: Select Interpreter (Python: 选择解释器)```。

在列表中，选择那个带有 ('venv': venv) 标志或者路径中包含 .\venv\Scripts\python.exe 的选项。

**安装项目所需的库**

现在，可以在 VS Code 的集成终端中（确保终端前面有 (venv) 标志）安装任何需要的第三方库了。

**完整的、专业的开发流程**
1. 在项目开始时就创建```.gitignore``` 文件，并加入 ```venv/```
2. 在虚拟环境中安装项目所需的库（例如```pip install pandas```）
3. 每当安装或更新了重要的库之后，就更新 ```requirements.txt```文件：
```pip freeze > requirements.txt```
1. 将源代码、```.gitignore``` 和更新后的 ```requirements.txt``` 一起提交到GitHub。

     ```
    git add .
    git commit -m "Add new feature and update dependencies"
    git push
    ```
**拿到项目后,只需要执行以下几步**

 1. 克隆仓库
    ```
    git clone <your_repo_url>
    cd <your_repo_name>
    ```
1. 创建并激活他们自己的```venv```
   ```
   python -m venv venv
   ```
   ```
   在Windows上:
   .\venv\Scripts\activate
   在macOS/Linux上:
   source venv/bin/activate
   ```
2. 根据依赖列表，一键安装所有库
   ```
   pip install -r requirements.txt
   ```
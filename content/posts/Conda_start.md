---
date: '2025-09-08T16:31:36+08:00'
draft: false
title: 'Conda Start' # <--- 修改这一行
summary: "Conda使用"
---

https://docs.conda.io/projects/conda/en/latest/user-guide/getting-started.html
#### 创建环境
创建一个新环境
```
conda create -n <env-name>
```
创建环境并且下载指定包
```
conda create -n myenvironment python numpy pandas
conda create -n myenv scipy=0.17.3  //特定库版本
```
创建环境指定特定pyhton版本
conda create -n myenv python=3.9
conda create -n myenv python=3.9
特定pyhton版本和多个库
conda create -n myenv python=3.9 scipy=0.17.3 astroid babel
#### 查看所有环境
```

conda info --envs
例如：
conda environments:

   base           /home/username/Anaconda3
   myenvironment   * /home/username/Anaconda3/envs/myenvironment
当前环境带*
```

改变当前环境为默认的一个
```
conda activate
```
   
#### 下载库
```
# via environment activation
conda activate myenvironment
conda install matplotlib

# via command line option
conda install --name myenvironment matplotlib
```

If a package you want is located in another channel, such as conda-forge, you can manually specify the channel when installing the package:
```
conda install conda-forge::numpy
```
#### 更新Conda
查看Conda版本
```
conda --version
```
更新到最新版本
```
conda update conda
```
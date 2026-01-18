---
title: Hexo的搭建与部署
date: 2026-01-11 19:20:00
tags: [Hexo, 博客, 教程]
categories: 工具类
---
### 一、创建英文管理员账户 hector

打开 Windows 设置 → 账户 → 家庭和其他用户 → 添加其他用户到这台电脑

### 二、登录用户，配置开发环境

登录 Windows 用户：hector

配置 Git 身份信息

git config --global user.name "hector"
git config --global user.email "QQ@qq.com"


验证：

git config --list

输出应包含用户名和邮箱

### 三、生成 SSH Key

打开 Git Bash

生成 SSH Key：ssh-keygen -t ed25519 -C "QQ@qq.com"


文件路径默认 /c/Users/hector/.ssh/id_ed25519 → 回车

id_ed25519 → 私钥

id_ed25519.pub → 公钥

查看公钥内容：cat ~/.ssh/id_ed25519.pub

复制输出内容添加到 GitHub：

打开 GitHub → Settings → SSH and GPG keys → New SSH key

Title：Hexo-blog SSH

Key：粘贴上一步复制的内容 → 点击 Add SSH key

### 四、测试 SSH 连接
ssh -T git@github.com



### 五、克隆或切换 Hexo 仓库到 SSH

切换远程 URL 到 SSH：

cd /f/blog/blog_hexo

git remote set-url origin git@github.com:hectorycl/hexo-blog.git

git remote -v

输出显示 SSH URL

### 六、Hexo 开发环境搭建

hexo init blog_hexo
cd blog_hexo
npm install

测试本地运行：

hexo clean
hexo g
hexo s


### 七、第一次通过 SSH push

git add .

git commit -m "Init Hexo blog"


push 到 GitHub：git push -u origin master


成功后仓库显示 Hexo 文件

后续 push 只需：git push

### 八、Vercel 部署（hector 用户）

登录 https://vercel.com
 → 使用 GitHub 账户授权

New Project → Import Git Repository → hexo-blog

Build Command：npm run build

Output Directory：public

点击 Deploy,成功后 Hexo 博客上线



### 九. 相关命令

1.  进入到博客目录 

    cd blog_hexo


2.  部署上线
    
    hexo clean 
    
    hexo g -d

3.  备份源码

    git add .

    git commit -m "update blog"

    git push 

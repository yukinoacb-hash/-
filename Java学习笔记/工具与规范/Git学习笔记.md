# Git 学习笔记

> 📅 日期：2026-09-27
> 🗄️ 你的仓库：D:\学习笔记
> 🌐 远程：GitHub（origin）

---

## 📖 目录

- [x] 第 1 课：Git 是什么
- [ ] 第 2 课：
- [ ] 第 3 课：
- [ ] 第 4 课：

---

## 第 1 课：Git 是什么

### Git = 时光机 + 同步器

```
没 Git 的时候：
  项目文件夹里一堆"最终版"、"最终版2"、"真的最终版"……

有 Git 的时候：
  每次修改打个"快照"（commit）
  可以随时回到任何一个历史版本
  可以同步到 GitHub 上备份
```

### 三个区（最核心的概念）

```
工作区（Working Directory）   ← 你正在写代码的地方
    ↓ git add
暂存区（Staging Area）        ← 临时存放，准备提交
    ↓ git commit
本地仓库（Local Repository）   ← 已经保存的历史记录
    ↓ git push
远程仓库（Remote Repository）  ← GitHub 上的备份
```

### 你刚才的操作就是这个流程

```bash
# 1. 你把笔记文件放进了工作区（已经在了）
# 2. 告诉 Git 哪些文件要提交
git add Java学习笔记/复习笔记.md

# 3. 把这些文件保存成一次"快照"
git commit -m "添加复习笔记"

# 4. 同步到 GitHub
git push origin master
```

### 常用操作流程图

```
修改文件
    ↓
git add 文件名      →  把文件放进"暂存区"
    ↓
git commit -m "说明" →  把暂存区的东西保存成一次"快照"
    ↓
git push            →  把快照同步到 GitHub
    ↓
git pull            →  从 GitHub 拉取最新的代码
```

---

## 第 2 课：基本命令

### ① git status — 查看当前状态（最常用）

```bash
git status
```

会告诉你：
- 哪些文件改了但还没 add（红色）
- 哪些文件 add 了但还没 commit（绿色）

### ② git add — 把文件放进暂存区

```bash
# 添加单个文件
git add 文件名

# 添加所有改过的文件
git add .

# 添加一个目录下所有文件
git add Java学习笔记/
```

### ③ git commit — 保存一次快照

```bash
git commit -m "这里写这次改了什么"
```

好的 commit 信息写法：
```bash
git commit -m "📝 添加 Spring 基础笔记"
git commit -m "🐛 修复查询接口空指针异常"
git commit -m "✨ 新增学生删除功能"
```

### ④ git log — 查看历史记录

```bash
git log

# 简洁版（一行一条）
git log --oneline

# 图形版（看分支）
git log --graph --oneline
```

### ⑤ git push — 推送到 GitHub

```bash
# 第一次推送
git push -u origin master

# 之后推送
git push
```

### ⑥ git pull — 从 GitHub 拉取最新代码

```bash
git pull
```

---

## 第 3 课：在 IDEA 里用 Git（不用敲命令）

IDEA 右侧有个 **Git** 面板（或者在菜单栏点 Git），常用的操作都能点按钮完成：

### 查看修改
```
右键文件 → Git → Show Diff
```
会显示你改了哪些行，绿色是新增，红色是删除。

### 提交
```
右键项目 → Git → Commit Directory
```
弹窗里勾选要提交的文件，写 commit 信息，点 Commit。

### 推送
```
Git → Push
```

### 拉取
```
Git → Pull
```

---

## 第 4 课：.gitignore — 哪些文件不用提交

你项目里有些文件不需要提交到 Git，比如：

```
# IDE 配置
.idea/
*.iml

# 编译后的文件
target/
*.class

# 系统文件
.DS_Store
Thumbs.db
```

这些已经写在你仓库的 `.gitignore` 里了，Git 会自动忽略它们。

---

## 常用命令速查

```bash
git status          # 查看当前状态
git add 文件名      # 添加文件到暂存区
git add .           # 添加所有改过的文件
git commit -m "说明" # 提交
git push            # 推送到 GitHub
git pull            # 从 GitHub 拉取
git log --oneline   # 查看历史
git diff            # 查看改了哪些内容
```

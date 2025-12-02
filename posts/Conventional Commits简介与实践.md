---
title: Conventional Commit 简介与实践
description: 帮助你快速掌握 Conventional Commit 规范，并一步步完成 commitlint + husky 的工程化配置。
pubDate: 2023-04-17 20:32:09
---
## 1. Conventional Commit是什么？

Conventional Commit是一套关于Git提交信息（commit message）的规范，该规范旨在使Git提交信息更易于人类阅读、可被自动化工具处理，更便于项目协作和版本管理。

### 1.1 优点

* 统一格式：项目成员按相同格式书写commit message。
* 提升可读性：帮助reviewer理解commit的主要内容。
* 增强自动化能力：易于被自动化工具处理，例如基于Commit Message生成Changelog。
* 支持CI/CD工作流：CI/CD工具可以根据commit信息执行相关脚本。
* 兼容SemVer：CI/CD工具可以根据commit message里的BREAKING-CHANGE、feat、fix升级major、minor、patch版本号。

## 2. Commit Message格式

```BASH
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```
说明：
* <>为必填占位符，[]为可选占位符。
* 字段一般不区分大小。 BREAKING CHANGE 必须大写。

## 3. 占位符详解

### 3.1 type（必填）
用于描述本次提交的变更类型。必须是一个名词。若没有scope，其后需紧跟半角冒号。

常用类型：
  * fix：修复bug（patch）。
  * feat：引入新的特性（minor）。
  * test：测试相关改动。
  * perf：性能优化。
  * docs：文档更新。
  * ci：CI配置变更。
  * build：构建系统或依赖变更。
  * refactor：不影响行为的代码重构。

### 3.2 scope（可选）
用于说明影响的模块或作用范围。

建议使用名词作为 scope，其含义通常是项目中的某个模块或功能域，以保持语义清晰一致。

在格式上
* scope被半角括号包围，
* 紧邻着type。
* 后紧跟半角冒号。

示例：
```
feat(localize): support English translation
```

### 3.3 description（必填）
对本次改动内容的简要描述。

字段前必须有一个空格，从而与半角冒号隔开。

### 3.4 body（可选）
用于描述本次变更的详细内容。必须与标题之间空一行。

### 3.5 footer（可选）
有关本次改动的额外标注。

可以有多条页脚，但是每一个页脚都需要遵循

格式为：
```
`<token>: <description>`
```
注意
* description 前有一个空格。
* token 中若存在空格，需要用`-`替换，但是 BREAKING CHANGE 是个例外。

示例 token：
* `BREAKING CHANGE`：表示变更不向后兼容，应导致 major 版本号 +1。
* `Change-Id`：Gerrit 系统生成的变更 ID。

可通过标题中的 ! 标记破坏性变更：
```
feat!: remove deprecated API
```

## 4. 工具

### 4.1 Commitlint（校验规范）
* `@commilint/cli`：commitlint的cli工具，帮助检查commit message是否符合规范。
* `@commitlint/config-conventional`：官方 Conventional Commit 配置

### 4.2 辅助工具
* `husky`：执行git hook钩子，便于使用版本管理工具管理和共享钩子。

## 5. 项目实践
以下基于Node v16环境。

### 5.1 安装依赖

#### 5.1.1 由于 Node 16 不支持 commitlint最新版，因此需要安装兼容版本

进入官网查找所有release的版本：conventional-changelog/commitlint: 📓 Lint commit messages (github.com)

查看某个release的包支持的node版本： `npm show @commitlint/cli@v17.0 engines`，可以以看到支持版本为 { node: '>=v14' }用它！

#### 5.1.2 安装如下依赖
对于单独的项目直接安装如下依赖即可，对于monorepo项目，将如下依赖安装至根目录中，从而对于子项目可以共享工具。配置文件可以在子项目下进行创建。
<!-- 安装依赖（适用于普通项目或 monorepo，建议放在根目录）。 -->
```bash
yarn add -D @commitlint/cli@17.0.0
yarn add -D @commitlint/config-conventional@17.0.0
yarn add -D husky
```

### 5.2 添加 commitlint 配置文件
创建commitlint.config.js（见附录）

### 5.3 配置自动执行commitlint。
借助 Husky，实现自动检查 commit message。

#### 5.3.1 初始化 Husky
生成 .husky目录。
```
npx husky install
```

#### 5.3.2 创建 `commit-msg` hook
在项目根目录下执行
```
npx husky add .husky/commit-msg "npx commitlint --edit $1"
```

该命令会创建 commit-msg 钩子，它会在提交 commit message 前触发，并将提交内容传递给 commitlint 工具进行校验。
更多 Git 钩子相关机制可参考官方文档： https://www.kernel.org/pub/software/scm/git/docs/githooks.html

### 5.4 Gerrit 集成（如使用）
Gerrit 会在 `.git/hooks/commit-msg` 自动生成 `Change-Id`。为兼容 Husky，需要：
1. 将 Gerrit 的 commit-msg hook 内容复制到 .husky/commit-msg 的顶部
2. 并确保在 npx commitlint --edit $1 之前 执行 

否则会导致 Change-Id 不生效

### 5.5 将 Husky 脚本纳入版本管理（推荐）
为了方便在项目成员间的协作，需要：
1. 将 .husky 目录下的钩子纳入版本管理，确保成员间使用统一的钩子。
2. 确保 hook 可执行：`git update-index --chmod=+x .husky/pre-commit`
3. 在 `package.json` 添加 `"postinstall": husky install .husky` 保证其他成员安装依赖后自动初始化 Husky

### 5.6 不纳入版本管理的替代方案
可在 package.json 中使用自动生成脚本：
```json
"postinstall": "yarn prepare:husky && yarn prepare:gerrit-commit-msg && yarn prepare:commitlint" // 该脚本将在安装依赖时调用，并顺序执行其中的脚本。
"prepare:husky": "husky install .husky" // 安装 .husky目录
"prepare:gerrit-commit-msg": "curl -L <your_gerrit_commit_msg_url> -o .husky/commit-msg && chmod +x .husky/commit-msg" // 将gerrit的commit-msg钩子添加到husky的commit-msg钩子文件中。
"prepare:commitlint": "npx husky add ./.husky/commit-msg \"npx --no -- commitlint --edit \"$1\"\"" // 将gerrit的commit-msg钩子添加到husky的commit-msg钩子文件中。
```
此方式会自动生成 Gerrit + commitlint 集成脚本。

## 6. 附录：commitlint.config.js 示例
```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'body-max-line-length': [2, 'always', 1000],
    'header-max-length': [0, 'always', 50],
    'type-enum': [
      2,
      'always',
      ['chore', 'build', 'ci', 'docs', 'feat', 'fix', 'perf', 'refactor', 'test'],
    ],
    'scope-enum': [2, 'always', ['<your-scope>']],
  },
  ignores: [(message) => message.toUpperCase().startsWith('WIP')],
};
```
---
name: java-release-deploy
description: Java多模块项目发布部署工具。用于检查开发分支、合并master、修改SNAPSHOT依赖为正式版本、分析依赖关系并按顺序打包部署。当用户提到"发布"、"deploy"、"修改SNAPSHOT"、"打包部署"、"版本升级"时使用此技能。
---

# Java Release Deploy

多模块 Java 项目的发布部署流程自动化工具。

## 使用场景

- 开发分支准备合并到 master 前的版本发布
- 将 SNAPSHOT 依赖修改为正式版本
- 多项目依赖的顺序打包和部署

## 核心概念

### 模块分类

在发布流程中，需要区分以下三类模块：

| 类型 | 说明 | Deploy策略 |
|-----|------|-----------|
| **对外 RPC 接口模块** | 包含 `@MoaProvider` 实现的接口定义，供其他服务远程调用 | ✅ 需要 deploy |
| **公共依赖包** | 基础工具类、公共配置、内部 SDK、数据模型等（非 RPC） | ❌ 仅需 install |
| **内部实现模块** | service/impl 层业务逻辑 | ❌ 无需版本变更 |

### 公共依赖包

公共依赖包是指内部维护的、**非对外 RPC 接口**的依赖包：
- **基础工具类**：字符串处理、日期工具、加密工具等
- **公共配置**：统一配置中心、常量定义等
- **内部 SDK**：内部服务调用封装、中间件封装等
- **数据模型**：DTO、VO、Entity 等数据结构定义

**识别方式**（二选一）：

1. **自动识别 + 用户确认**：
   - 根据模块名特征自动识别：common, util, core, base, support, config, model, entity
   - 检查不包含 RPC 框架注解（`@MoaProvider`, `@DubboService` 等）
   - **识别后列出并询问用户确认**

2. **用户直接指定**：
   - 用户可直接输入内部公共依赖包的 artifactId 或模块路径
   - 例如：`ultron-dependency`, `xxx-common`, `xxx-util`

**确认提示模板**：

> 检测到以下模块可能是公共依赖包（非对外 RPC 接口）：
> - xxx-common
> - xxx-util
> - xxx-model
>
> 请确认以上识别是否正确，或输入需要补充的公共依赖包名称：

### 对外 RPC 接口模块

对外 RPC 接口模块是指包含 RPC 接口定义、供其他服务远程调用的模块：

**识别特征**（内部 MOA 框架）：
- 包含 `@MoaProvider` 注解的实现类
- `io/interfaces` 目录下的接口定义
- yml 配置中的 `moa.consumer.configs` 服务定义

**识别特征**（其他 RPC 框架）：
- `@DubboService`, `@FeignClient`, `@GrpcService` 等注解

## 工作流程

### 第一步：发现目标项目

**重要**：必须先对所有项目执行 `git fetch` 拉取远程分支信息，再进行扫描。否则可能遗漏仅存在于远程的开发分支。

```bash
# 1. 先 fetch 所有项目的远程分支
for dir in */; do
  if [ -d "$dir/.git" ] && [ -f "$dir/pom.xml" ]; then
    cd "$dir"
    REMOTE=$(git remote | head -1)
    git fetch $REMOTE 2>/dev/null
    cd ..
  fi
done

# 2. 再扫描包含指定开发分支的项目
for dir in */; do
  if [ -d "$dir/.git" ] && [ -f "$dir/pom.xml" ]; then
    cd "$dir"
    if git branch -a 2>/dev/null | grep -q "分支名"; then
      echo "✓ $dir"
    fi
    cd ..
  fi
done
```

**注意**：不先 fetch 会导致遗漏仅在远程新建的分支（如其他同事新推送的分支）。

### 第二步：更新分支并合并

先获取远程仓库名称（通常是 origin，但可能不同）：

```bash
# 获取默认远程仓库名称
REMOTE=$(git remote | head -1)
echo "远程仓库: $REMOTE"
```

对每个项目执行：

```bash
git fetch $REMOTE
git checkout master && git pull $REMOTE master
git checkout 开发分支 && git pull $REMOTE 开发分支
git merge master -m "Merge master into 开发分支"
```

**冲突处理**：
- pom.xml 版本号冲突：通常采用 master 版本作为基础，然后升级
- 代码冲突：需要人工确认

### 第三步：查找 SNAPSHOT 依赖

```bash
grep -rn "SNAPSHOT" "项目目录/pom.xml"
```

### 第四步：确认公共依赖包

**识别并确认哪些模块是公共依赖包（无需 deploy）**：

```bash
# 自动识别可能的公共依赖包
find . -type f -name "pom.xml" -exec grep -l "packaging>jar" {} \; | \
  xargs -I{} dirname {} | \
  grep -E "(common|util|core|base|support|config|model|entity)" | sort -u
```

**向用户确认**：

> 检测到以下模块可能是公共依赖包（仅需 install，无需 deploy）：
> - xxx-common
> - xxx-util
>
> 请确认以上识别是否正确？
> 或直接输入您内部的公共依赖包名称（逗号分隔）：

**用户可选操作**：
- 确认自动识别结果
- 补充/修正公共依赖包列表
- 直接输入公共依赖包的 artifactId

### 第五步：确定版本升级策略

| 场景 | 版本策略 |
|-----|---------|
| 有 API/对外接口层代码变更 | master 版本 +1，需要 deploy |
| 仅 service/impl 层变更 | 不需要 deploy |
| 仅依赖版本更新 | 不需要升级自身版本 |

检查对外 API（RPC 接口）层变更：

**对外 API 模块定义**：指包含 RPC 接口定义、供其他服务远程调用的模块。

```bash
# 步骤1：识别对外 RPC 接口模块
# 检查变更文件中是否包含 RPC 框架注解

# 内部 MOA 框架识别（优先）
# - @MoaProvider：定义 RPC 接口实现类
# - @MoaConsumer：引用 RPC 接口
git diff master --name-only -- "*.java" | while read file; do
  if grep -qE "@MoaProvider|@MoaConsumer" "$file" 2>/dev/null; then
    dirname "$file" | cut -d'/' -f1-2
  fi
done | sort -u

# 检查 yml 配置中的 MOA 服务定义
git diff master --name-only -- "*.yml" "*.yaml" | while read file; do
  if grep -qE "moa:|serviceUri:|interfaceName:" "$file" 2>/dev/null; then
    echo "配置变更: $file"
  fi
done

# 其他 RPC 框架（Dubbo、Feign、gRPC）
git diff master --name-only -- "*.java" | while read file; do
  if grep -qE "@DubboService|@FeignClient|@GrpcService" "$file" 2>/dev/null; then
    dirname "$file" | cut -d'/' -f1-2
  fi
done | sort -u

# 步骤2：检查接口定义文件变更
# 对外 API 通常是 interface 且在特定模块下（io/interfaces/api 目录）
git diff master --name-only -- "*.java" | grep -E "(io/interfaces|/api/|/client/)" | head -10

# 步骤3：确认接口模块
# 检查变更的 interface 文件是否被 @MoaProvider 实现
```

**判断标准**：
- ✅ 需要 deploy：模块包含 `@MoaProvider` 实现的接口定义，且接口有变更
- ✅ 需要 deploy：`io/interfaces` 目录下的接口定义有变更
- ❌ 无需 deploy：仅 `@MoaConsumer` 引用变更（消费方无需发布）
- ❌ 无需 deploy：仅内部实现变更，或仅公共依赖包变更

### 第六步：分析依赖关系

按依赖层级排序打包顺序（从底层到上层）：

1. 基础依赖包（如 ultron-dependency）
2. 公共包装层（如 ultron-wrapper）
3. 业务模块（依赖上述包的项目）
4. 聚合服务

### 第七步：本地安装（保底）

```bash
# 安装到本地（跳过测试，启用并行构建）
mvn clean install -DskipTests -T 1C
```

**注意**：
- 此步骤为保底操作，确保所有依赖可在本地正常编译
- 跳过测试加速打包（`-DskipTests`）
- 按依赖顺序依次执行，从底层依赖到上层业务模块

### 第八步：询问用户是否部署到远程仓库

**在所有项目本地安装完成后，向用户确认**：

> 本地安装已完成，以下项目检测到对外 RPC 接口变更，需要部署到远程仓库：
> - 项目A-api (版本 x.x.x)
> - 项目B-interface (版本 y.y.y)
>
> 是否继续执行 deploy 到远程仓库？(Y/n)

**决策依据**：
- 有对外 RPC 接口变更的模块 → 建议 deploy
- 仅公共依赖包变更（工具类/配置/数据模型） → 无需 deploy
- 仅内部 service/impl 变更 → 无需 deploy
- 用户可选择跳过部署，仅保留本地安装

### 第九步：部署到远程仓库（可选）

仅在用户确认后执行：

```bash
# 部署到远程仓库（仅对外 RPC 接口模块，启用并行构建）
mvn deploy -DskipTests -T 1C
```

**注意**：
- 仅部署包含对外 RPC 接口定义的模块
- 公共依赖包（工具类/配置/数据模型）无需 deploy，本地 install 即可
- 400 错误表示版本已存在，无需重复部署
- 401 错误检查 Maven settings.xml 中的认证配置
- 部署顺序同样遵循依赖关系，从底层到上层

### 第十步：提交变更并推送到远程（可选）

**询问用户是否提交变更并推送到远程**：

> 是否提交变更并推送到远程分支？
> 1. 提交并推送到远程 (git commit && git push)
> 2. 仅提交到本地 (git commit)
> 3. 暂不提交，稍后手动处理
>
> 请选择 (1/2/3)：

**根据用户选择执行**：

```bash
# 选项1: 提交并推送到远程
git add -A
git commit -m "chore: 修改SNAPSHOT依赖为正式版本 (分支名)"
REMOTE=$(git remote | head -1)
git push $REMOTE 开发分支

# 选项2: 仅提交到本地
git add -A
git commit -m "chore: 修改SNAPSHOT依赖为正式版本 (分支名)"

# 选项3: 跳过提交步骤
```

### 第十一步：创建/合并 GitLab MR（可选）

**仅在用户已推送到远程后询问**。如果用户在第十步选择了选项2或3，则跳过此步骤。

**询问用户是否需要创建/合并 GitLab MR**：

> 是否需要创建 GitLab MR 将开发分支合并到 master？(Y/n)

**GitLab MR 工具（glab）**：

使用 `glab` 命令行工具管理 MR：

1. **检查 glab 是否已安装**：
```bash
which glab || echo "glab 未安装"
```

2. **如未安装，使用 brew 安装**：
```bash
brew install glab
```

3. **配置 glab 认证**：

优先检查环境变量中是否已有 Token：
```bash
# 检查是否已配置 GITLAB_TOKEN 环境变量
if [ -n "$GITLAB_TOKEN" ]; then
  echo "已从环境变量获取 GITLAB_TOKEN"
  glab auth login --hostname <your-gitlab-host> --token "$GITLAB_TOKEN"
else
  echo "未找到 GITLAB_TOKEN 环境变量，需要获取 Token"
fi
```

**未找到 Token 时的处理流程**：

如果环境变量中没有 GITLAB_TOKEN，必须引导用户获取：

1. 使用 `open` 命令自动打开浏览器到 Token 创建页面：
```bash
# <your-gitlab-host> 替换为实际的 GitLab 域名
open "https://<your-gitlab-host>/-/profile/personal_access_tokens"
```
2. 提示用户在页面上完成以下操作：
   - **Token name**: 填写一个有意义的名称（如 `glab-cli`）
   - **Expiration date**: 建议设置为 1 年
   - **Scopes**: 勾选 `api`、`read_repository`、`write_repository`
   - 点击 **Create personal access token** 按钮
   - **复制生成的 Token**（页面仅显示一次）
3. 等待用户提供获取到的 Token
4. 使用 Token 完成 glab 认证并持久化保存到 shell 配置文件

**Token 持久化**：获取到 Token 后，检测用户当前 shell 类型并保存到对应配置文件：
```bash
# 检测当前 shell 类型并确定配置文件
CURRENT_SHELL=$(basename "$SHELL")
case "$CURRENT_SHELL" in
  zsh)  SHELL_RC="$HOME/.zshrc" ;;
  bash) SHELL_RC="$HOME/.bashrc" ;;
  fish) SHELL_RC="$HOME/.config/fish/config.fish" ;;
  *)    SHELL_RC="$HOME/.profile" ;;  # 通用兜底
esac

# 检查是否已存在 GITLAB_TOKEN 配置（避免重复追加）
if ! grep -q "GITLAB_TOKEN" "$SHELL_RC" 2>/dev/null; then
  if [ "$CURRENT_SHELL" = "fish" ]; then
    echo 'set -gx GITLAB_TOKEN "<token值>"' >> "$SHELL_RC"
  else
    echo 'export GITLAB_TOKEN="<token值>"' >> "$SHELL_RC"
  fi
  echo "已将 GITLAB_TOKEN 写入 $SHELL_RC"
fi

# 使配置生效
source "$SHELL_RC" 2>/dev/null || true
```
这样下次执行时可以自动从环境变量读取，无需再次输入。

4. **检查是否已有 MR**：
```bash
cd 项目目录
glab mr list --source-branch 开发分支 --target-branch master
```
   - 如果已有 MR，展示 MR 信息给用户，询问是否需要合并
   - 如果没有 MR，自动创建

5. **创建 MR**（仅在不存在时）：

首先收集与 master 的所有差异 commit 描述作为 MR 描述：
```bash
# 获取与 master 的差异 commit 列表
COMMITS=$(git log master..开发分支 --oneline --no-merges)
# 格式化为 MR 描述
DESCRIPTION="## Commits\n${COMMITS}"
```

然后创建 MR：
```bash
glab mr create --source-branch 开发分支 --target-branch master \
  --title "Merge 开发分支 into master" \
  --description "$DESCRIPTION"
```

6. **合并 MR**（如用户要求）：
```bash
glab mr merge <MR编号> --yes
```

7. **获取合并后 master 最新 commitId**（合并 MR 后执行）：
```bash
git checkout master && git pull $REMOTE master
COMMIT_ID=$(git log -1 --format='%H')
echo "master 最新 commitId: $COMMIT_ID"
```

将所有项目合并后的 master commitId 收集起来，在报告中以如下格式输出，用于提交上线单：

```markdown
## 上线单 CommitId

| 项目 | master commitId |
|------|----------------|
| 项目A | commitIdA |
| 项目B | commitIdB |

**CommitId 汇总（复制用）**：
`commitIdA,commitIdB,...`
```

**注意**：
- 首次使用需要用户到 GitLab 的 Personal Access Tokens 页面创建 Token
- Token 建议勾选 `api`、`read_repository`、`write_repository` 权限范围
- 配置后 Token 会被缓存到环境变量，后续无需重复认证
- **必须先检查已有 MR**，避免重复创建

### 第十二步：生成报告

生成 deploy-report-{分支名}.md 文档，包含：
- 涉及项目列表
- 版本变更详情
- 执行的操作
- Git 提交记录
- MR 链接（如已创建）
- 上线单 CommitId 汇总表（如已合并 MR）

## 公共依赖包更新流程

当公共依赖包有代码修改时：

1. **升级公共依赖包版本号**（在 master 基础上 +1）
2. **检查哪些项目引用了该依赖包的 SNAPSHOT 版本**
3. **更新引用项目中的版本号为正式版本**
4. **本地 install 即可，无需 deploy**（因为不是对外 RPC 接口）

```bash
# 查找可能的公共依赖包模块
find . -type f -name "pom.xml" -exec grep -l "packaging>jar" {} \; | \
  xargs -I{} dirname {} | \
  grep -E "(common|util|core|base|support|config|model|entity)" | sort -u

# 查看当前分支 pom.xml 的依赖版本变更
git diff master -- "**/pom.xml" | grep -E "(<version>|<artifactId>)" | head -20

# 确认变更的依赖不包含 RPC 接口
# 如果依赖模块不包含 @MoaProvider/@DubboService 等注解，则为公共依赖包
```

## 常见问题

### 编译错误处理

1. **拼写错误**：仔细检查变量名、方法名
2. **API 不兼容**：确保依赖版本已正确安装到本地
3. **废弃 API**：替换为新 API（如 `sun.misc.BASE64Encoder` → `java.util.Base64`）

### Deploy 失败

- **400 Bad Request**：版本已存在，无需重复部署
- **401 Unauthorized**：检查 Maven settings.xml 中的认证配置
- **网络超时**：重试或检查网络连接

## 输出示例

```markdown
# dev-xxx 分支依赖版本修改及部署报告

## 涉及项目
| 项目 | 类型 | 本地安装 | 需Deploy | 已Deploy | 新版本 |
|-----|------|---------|---------|---------|-------|
| xxx-api | 对外RPC接口 | ✅ | ✅ | ✅ | 1.0.1 |
| xxx-common | 公共依赖包 | ✅ | ❌ | - | 2.0.1 |
| xxx-service | 内部实现 | ✅ | ❌ | - | - |

## 执行的操作
1. 分支更新与合并
2. SNAPSHOT 依赖修改
3. 本地安装（保底）
4. 变更提交
5. 推送分支并创建/合并 MR
6. 用户确认后部署到远程仓库

## 部署状态
- 用户选择: [是/否] 部署到远程仓库
- 部署结果: [成功/跳过/失败原因]

## 上线单 CommitId
| 项目 | master commitId |
|------|----------------|
| projectA | abc123def456 |
| projectB | 789xyz012abc |

**CommitId 汇总（复制用）**：
`abc123def456,789xyz012abc`

## 模块分类说明
- 对外RPC接口：包含 @MoaProvider（或 @DubboService/@FeignClient）注解，需要 deploy
- 公共依赖包：工具类/配置/数据模型，仅需 install
- 内部实现：service/impl 层，无需版本变更
```

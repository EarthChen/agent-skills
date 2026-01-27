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

## 工作流程

### 第一步：发现目标项目

检查工作区中哪些项目包含指定的开发分支：

```bash
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

### 第四步：确定版本升级策略

| 场景 | 版本策略 |
|-----|---------|
| 有 API/IO 层代码变更 | master 版本 +1，需要 deploy |
| 仅 service 层变更 | 不需要 deploy |
| 仅依赖版本更新 | 不需要升级自身版本 |

检查 API 变更：

```bash
git diff master --name-only | grep -E "(api|io|interfaces)"
```

### 第五步：分析依赖关系

按依赖层级排序打包顺序（从底层到上层）：

1. 基础依赖包（如 ultron-dependency）
2. 公共包装层（如 ultron-wrapper）
3. 业务模块（依赖上述包的项目）
4. 聚合服务

### 第六步：打包部署

```bash
# 安装到本地（跳过测试）
mvn clean install -DskipTests

# 部署到远程仓库（仅有 API 变更的项目）
mvn deploy -DskipTests
```

**注意**：
- 先 install 再 deploy，确保本地依赖可用
- 跳过测试加速打包（`-DskipTests`）
- 400 错误表示版本已存在，无需重复部署

### 第七步：提交变更

```bash
git add -A
git commit -m "chore: 修改SNAPSHOT依赖为正式版本 (分支名)"
```

### 第八步：生成报告

生成 deploy-report-{分支名}.md 文档，包含：
- 涉及项目列表
- 版本变更详情
- 执行的操作
- Git 提交记录

## 二方包版本更新规则

当 ultron-dependency 或 ultron-wrapper 等二方包有代码修改时：

1. **升级二方包版本号**（在 master 基础上 +1）
2. **检查哪些项目主动引用了该二方包的 SNAPSHOT 版本**
3. **仅更新主动修改过引用版本的项目**（不是所有引用项目都要更新）

检查方法：

```bash
git diff master -- pom.xml | grep -E "ultron.dependency|ultron-wrapper"
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
| 项目 | API变更 | 需要Deploy | 新版本 |
|-----|--------|-----------|-------|
| xxx | ✅ | ✅ | 1.0.1 |

## 执行的操作
1. 分支更新与合并
2. SNAPSHOT 依赖修改
3. 打包部署
4. 变更提交
```

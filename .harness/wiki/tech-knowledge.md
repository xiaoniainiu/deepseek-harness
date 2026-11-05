# 技术知识库：deepseek-harness

> Agent 在编码和评审阶段按需查阅。初始化时自动生成，随项目迭代持续补充。

---

## 一、项目架构

- **技术栈**: TypeScript/Node.js
- **架构模式**: Monolith
- **项目规模**: 56 sql files / 3 modules
- **部署模型**: GitLab CI + Docker

## 二、外部系统依赖



## 三、关键配置文件

| 文件 | 用途 |
|------|------|


## 四、项目启动命令

```bash
# 编译
npx tsc --noEmit

# 测试
npx vitest run

# Lint


# 启动开发服务器
# TODO: 补充 dev server 命令
```

## 五、已知坑点（Pitfalls）

暂无已知坑点。随项目迭代在此补充。

---

## 六、补充指南

遇到以下情况时，补充本文件：
1. 发现外部系统的非直觉行为
2. 遇到环境/配置相关的 debug 经验
3. 发现代码库中的隐含约定（非注释可见）
4. 跑通新的集成测试路径

格式：一行描述 + 具体细节。避免 copy-paste 大段代码。

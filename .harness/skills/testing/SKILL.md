---
expiration_signal:
  status: not_yet_evaluated
  last_evaluated: "2026-05-25"
  evaluation_baseline:
    intercepted_real_bugs: 0
    triggered_total: 0
    last_triggered_days_ago: null
  retire_path: dormant
---

# 测试技能

## 触发条件

Phase 5（单测编写）+ Phase 6（单测评审）+ Phase 9（线上验证）

## 核心原则

改了哪个接口就测哪个，用真实数据结构，不用 mock 掩盖问题。

---

## ⚠️ Phase 5 强制首步：依赖骨架生成（不可跳过）

<STOP>编写任何 @Test 方法之前，必须先完成下面 3 步。直接写测试→编译失败 3 次是最常见的反模式。</STOP>

### Step 1: 读取被测类 import 声明

```
读取目标文件顶部 import 区域（前 50 行左右）
→ 直接复用到测试文件（不猜路径，不凭记忆）
```

### Step 2: 统计依赖注入字段

```
grep -c "@Autowired\|@Inject\|@Resource\|@Value" <被测类文件>
→ 统计 DI 字段数量

IF 字段数 > 8:
  → 使用 @SpringBootTest(classes={TargetClass.class}) + @MockBean
  → 或使用 ReflectionTestUtils 手动注入（轻量级）
  → 不要用 @InjectMocks（会因依赖链过长失败）

IF 字段数 ≤ 8:
  → 使用 @ExtendWith(MockitoExtension.class) + @InjectMocks + @Mock
  → 逐一列出每个需要 Mock 的字段
```

### Step 3: 生成测试骨架并编译通过

先生成以下骨架，确认编译通过后再填充测试逻辑：

```java
package <与被测类同包>;

import <直接从被测类复制的 import>;
import org.junit.jupiter.api.*;
import org.mockito.*;

@ExtendWith(MockitoExtension.class)
class <被测类>Test {
    @InjectMocks  // 或 @SpringBootTest
    private <被测类> target;
    
    @Mock private <依赖1> dep1;
    @Mock private <依赖2> dep2;
    // ... 逐一列出
    
    @BeforeEach
    void setUp() {
        // Value 注入等初始化
    }
    
    @Test
    void placeholder_compileCheck() {
        assertNotNull(target);
    }
}
```

**验证**：`mvn test-compile -pl <module>` 编译通过后才开始写真正的测试方法。

---

## 内部技能依赖

- TDD 执行循环 → `skills/coding/tdd-loop.md`
- 反幻觉验证 → `skills/coding/verification.md`
- **Playwright E2E 测试** → `skills/testing/playwright.md`（前端 E2E + CAS 认证 + HiUI 组件 + CI 集成）

---

## 测试过程基线（MIT 信息部）

> 来源：小米信息部测试过程基线，所有业务线强制执行。

### 测试方案规范（Phase 5 前置）

| 要求 | 对应 Harness |
|------|-------------|
| P0 项目必须有测试方案 | `requirement.md` 含测试策略章节 |
| 方案关联 PRD，明确业务方/用户 | plan.json `context` 字段 |
| 明确功能测试重点/难点 | plan.json `testDescription` |
| 方案经产研评审通过 | Phase 2 评审 + [H2] |

### 测试用例设计规范（Phase 5）

| 要求 | Harness 门禁 |
|------|-------------|
| 步骤描述完整，他人可读可执行 | testDescription 具体到输入/输出/边界 |
| 覆盖正向+逆向+异常+隐性需求 | acceptanceCriteria 含正常+异常路径 |
| 上下游交互用例标明数据流向和检查点 | context 字段含集成点描述 |
| 核心业务场景维护为主 | 优先测核心路径，不追求覆盖率虚高 |

### 测试用例评审规范（Phase 6）

| 要求 | Harness 门禁 |
|------|-------------|
| 产研测三方参与评审 | Phase 6 门禁 + [H3] |
| 表述逻辑清晰，不直接读 case | review 输出结构化（P0/P1/P2） |
| 评审结论无异议后发送纪要 | review-v{N}.md 记录 |

### 缺陷提报与跟踪规范（Phase 10）

| 要求 | Harness 映射 |
|------|-------------|
| 关键信息完整，复现步骤准确 | issues.json `description` 含 file:line + 复现步骤 |
| 明确缺陷等级，不得一切定 P0/P1 | issues.json `severity` 严格按 high/medium/low |
| 及时跟踪验证关闭，偶现缺陷确认根因 | issues.json `status` open→resolved 生命周期 |

### 测试报告规范（Phase 8）

| 要求 | Harness 映射 |
|------|-------------|
| 报告含 case 通过率 + 缺陷遗留 + 测试结论 | CI 产出 + run-state.json metrics |
| 遗留问题三方确认，措施在报告中体现 | retrospective.md 遗留问题章节 |

### 线上验证规范（Phase 9）

| 要求 | Harness 映射 |
|------|-------------|
| 所有需求上线后必须验证 | Phase 9 门禁：health check + 冒烟 |
| 明确验证范围（数据/权限） | plan.json verificationCommands |
| 测试数据做标识，验证后及时清理 | Phase 9 checklist：数据清理确认 |
| 对业务有侵扰的测试提前沟通 | [H4] 人工确认点含影响说明 |

---

### 测试方法设计（BCDE 维度）

编译通过后（已完成上方骨架三步），按 BCDE 四维度设计测试方法：

| 维度 | 含义 | 关注点 |
|------|------|--------|
| **B**order | 边界值 | 恰好满足/不满足条件的阈值 |
| **C**orrect | 正确路径 | 正常业务流程的期望行为 |
| **D**esign | 设计意图 | 开关/配置/前置条件 |
| **E**rror | 异常路径 | 空值/超限/无权限/网络超时 |

---

## 补充规则

1. **测试范围**：与变更代码 1:1 对应，不写无关测试
2. **数据准备**：用贴近生产的数据结构，不用 `test123` 占位
3. **断言明确**：断言具体字段值，不只断言"不为 null"
4. **独立性**：每个测试独立运行，不依赖执行顺序
5. **自测证明**：自测方式三选一（单测覆盖率 / Teego 用例通过 / 截图证明），推荐单测

## 门禁验证

```
Phase 5: status=SUCCESS + tests>0 + passed=total
Phase 6: 评审通过（产研确认覆盖充分）
Phase 9: health check + smoke test + 测试数据已清理
```

如果 `tests == 0`，Phase 5 门禁不通过。

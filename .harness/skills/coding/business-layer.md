# Business 层 SOP

## 适用范围

Service / UseCase / 业务编排

## 实现步骤

1. **确认业务规则**
   - 查 wiki/business-flows.md 确认状态流转
   - 明确前置条件和后置动作

2. **方法设计**
   - 一个 public 方法 = 一个业务操作
   - private 方法抽取 > 30 行的逻辑块
   - 方法命名体现业务含义（不是技术动作）

3. **事务管理**
   - 写操作加 @Transactional
   - 读操作不加
   - 事务边界 = 一个 public 方法

4. **编排逻辑**
   ```
   public method:
     1. 参数校验（业务级）
     2. 查询数据
     3. 执行业务规则
     4. 持久化结果
     5. 后置动作（通知/事件）← post-commit
   ```

5. **外部调用**
   - 通过注入的 External Client 调用
   - 通知类操作放 post-commit（避免数据未落库就发了通知）
   - 外部调用失败不应导致主流程回滚（除非业务要求）

6. **并发控制**
   - 涉及竞态的写操作 → 分布式锁 或 乐观锁
   - 锁粒度：越细越好（按业务 ID 锁，不全局锁）

## 检查清单

- [ ] 写操作有 @Transactional？
- [ ] 通知/消息在 post-commit 发送？
- [ ] 外部调用有超时和降级？
- [ ] 并发写入有锁保护？
- [ ] 不超过一个 public 方法处理同一业务操作？

## 反模式

```
// WHY: Agent 在事务内调外部系统发消息，外部系统成功但后续代码异常回滚了，用户收到通知但操作实际没成功
// 正确: 通知放 TransactionSynchronization.afterCommit()
```

# 1、ERC20 transfer方法，主要是修改balance的状态么？

对，**“修改 balances 状态”就是 transfer 的全部核心工作**。  
在 EVM 层面可以简化为一句话：

> 把 `balances[from]` 减 `amount`，再把 `balances[to]` 加 `amount`，最后记一条 `Transfer` 事件——其他都是围绕这两笔 SSTORE 的安全与合规包装。

所以：  
- **主要任务** → 改余额（两次 SSTORE）。  
- **附带任务** → 参数校验、事件日志、返回 true。  

只要这两处状态写成功，链上就永久承认“钱已经过去”；事件日志只是让链外能“看见”这次变化，并不影响资产本身。




# 飞书卡片赞踩功能实现总结

## 实现概述

本次实现为 HappyClaw 项目添加了完整的飞书卡片反馈功能，用户可以对 AI 回复进行点赞、点踩或召唤人工协助。

## 实现的功能

### 1. 数据库层 (src/db.ts)

#### 表结构
- **表名**: `message_feedback`
- **字段**:
  - `id`: 自增主键
  - `message_id`: 飞书消息 ID
  - `user_id`: 用户 ID
  - `action`: 反馈类型 (`thumb_up` 或 `thumb_down`)
  - `created_at`: 创建时间
- **约束**:
  - `UNIQUE(message_id, user_id)`: 同一用户对同一消息只能有一条反馈
  - `CHECK(action IN ('thumb_up', 'thumb_down'))`: 限制 action 值
- **索引**:
  - `idx_message_feedback_message_id`: 消息 ID 索引
  - `idx_message_feedback_user_id`: 用户 ID 索引

#### 数据库函数
- `storeMessageFeedback(messageId, userId, action)`: 存储用户反馈，使用 UPSERT 允许用户修改反馈
- `getMessageFeedbackStats(messageId)`: 查询某条消息的点赞和点踩统计

#### Schema 版本
- 从 v38 升级到 v39

### 2. 飞书卡片按钮 (src/feishu-cards/builder.ts)

在 `buildAgentReplyCard()` 函数中，当 `status === 'done'` 时添加三个按钮：
- **👍 有用** (thumb_up): 普通按钮
- **👎 没用** (thumb_down): 普通按钮
- **🙋 召唤人工** (call_human): 主要按钮（蓝色）

### 3. 飞书事件处理 (src/feishu.ts)

#### 类型定义更新
在 `ConnectOptions` 接口中添加：
- `onCardFeedback`: 反馈回调函数
- `onCallHuman`: 召唤人工回调函数

#### 事件处理逻辑
在 `card.action.trigger` 事件处理器中：
1. **点赞/点踩处理**:
   - 提取 `action`, `messageId`, `userId`
   - 添加 emoji reaction（点赞用 `MeMeMe`，点踩用 `EMBARRASSED`）
   - 对话题群（thread）在 root message 上添加 reaction
   - 调用 `onCardFeedback` 回调存储到数据库

2. **召唤人工处理**:
   - 调用 `onCallHuman` 回调
   - 向群内注入系统消息

### 4. 主进程回调 (src/index.ts)

#### handleCardFeedback()
- 记录日志：`User feedback: 👍 有用` 或 `User feedback: 👎 没用`
- 调用 `storeMessageFeedback()` 存储到数据库
- 捕获并记录错误

#### handleCallHuman()
- 记录日志：`User requested human assistance`
- 获取用户信息并生成系统消息
- 存储消息到数据库
- 通过 WebSocket 广播到 Web 端

#### 回调注册
在三个位置注册回调：
1. 初始连接 (connectUserIMChannels)
2. 热重载 - admin (reloadFeishuConnection)
3. 热重载 - user (reloadUserIMConnection)

### 5. IM 管理器类型 (src/im-manager.ts)

在 `ConnectFeishuOptions` 接口中添加：
- `onCardFeedback`: 反馈回调
- `onCallHuman`: 召唤人工回调

## 技术特点

1. **UPSERT 机制**: 使用 `INSERT ... ON CONFLICT ... DO UPDATE` 允许用户修改反馈
2. **数据库约束**: UNIQUE 和 CHECK 约束确保数据完整性
3. **Emoji Reaction**: 点赞显示举手（MeMeMe），点踩显示尬笑（EMBARRASSED）
4. **话题群支持**: 自动识别话题群并在 root message 上添加 reaction
5. **错误处理**: 完善的错误捕获和日志记录
6. **类型安全**: 完整的 TypeScript 类型定义

## 调用链路

```
用户点击按钮
    ↓
飞书服务器推送 card.action.trigger 事件
    ↓
src/feishu.ts:1678 - 接收事件
    ↓
src/feishu.ts:1714 - 识别 thumb_up/thumb_down/call_human
    ↓
src/feishu.ts:1723 - 添加 emoji reaction
    ↓
src/feishu.ts:1734 - 调用 connectOptions.onCardFeedback()
    ↓
src/index.ts:8306 - handleCardFeedback() 执行
    ↓
src/index.ts:8318 - 调用 storeMessageFeedback()
    ↓
src/db.ts:5900 - 执行 SQL INSERT ... ON CONFLICT
    ↓
完成
```

## 日志关键字

### 正常流程
```
[INFO] Card action: feedback
    chatJid: "feishu:oc_xxx#agent:xxx"
    messageId: "om_xxx"
    action: "thumb_up"
    userId: "ou_xxx"

[INFO] User feedback: 👍 有用
    chatJid: "feishu:oc_xxx#agent:xxx"
    action: "thumb_up"
    messageId: "om_xxx"
    userId: "ou_xxx"

[DEBUG] Feedback stored to database
    messageId: "om_xxx"
    userId: "ou_xxx"
    action: "thumb_up"
```

### 异常日志
```
[DEBUG] Card action: no mapping for messageId
    messageId: "om_xxx"

[ERROR] Failed to store feedback
    err: { ... }
    messageId: "om_xxx"
    userId: "ou_xxx"
    action: "thumb_up"
```

## 数据查询示例

### 查询反馈记录数
```sql
SELECT COUNT(*) as count FROM message_feedback;
```

### 查询最近的反馈
```sql
SELECT * FROM message_feedback ORDER BY created_at DESC LIMIT 10;
```

### 统计某条消息的反馈
```sql
SELECT action, COUNT(*) as count
FROM message_feedback
WHERE message_id = 'om_xxx'
GROUP BY action;
```

### 统计用户的反馈习惯
```sql
SELECT 
  user_id,
  SUM(CASE WHEN action = 'thumb_up' THEN 1 ELSE 0 END) as thumb_up_count,
  SUM(CASE WHEN action = 'thumb_down' THEN 1 ELSE 0 END) as thumb_down_count
FROM message_feedback
GROUP BY user_id;
```

## 测试验证

所有功能已通过以下测试：
- ✅ 数据库表创建
- ✅ 插入点赞反馈
- ✅ 同一用户修改反馈（点赞改点踩）
- ✅ 多个用户反馈
- ✅ UNIQUE 约束验证
- ✅ CHECK 约束验证
- ✅ TypeScript 类型检查通过

## 相关文件清单

| 文件 | 修改内容 |
|------|---------|
| `src/db.ts` | 添加 message_feedback 表、storeMessageFeedback()、getMessageFeedbackStats() |
| `src/feishu-cards/builder.ts` | 在 done 状态卡片添加反馈按钮 |
| `src/feishu.ts` | 更新 ConnectOptions 类型、处理 card.action.trigger 事件 |
| `src/index.ts` | 添加 handleCardFeedback()、handleCallHuman()、注册回调 |
| `src/im-manager.ts` | 更新 ConnectFeishuOptions 类型 |

## 版本信息

- **实现日期**: 2026-06-01
- **Schema 版本**: v38 → v39
- **分支**: feat/feedback

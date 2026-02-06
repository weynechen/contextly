# Contextly MVP 设计补充与完善

> 结论：原始设计文档方向正确，但存在若干关键缺口（如：操作次数未定义、SM-2 公式缺失、字段约束与索引未说明、错误处理/权限/限流/脱网策略缺失、客户端与插件的 API 协议/字段约定不完整）。因此需要补充以下内容以形成可落地的 MVP 设计规范。

---

## 1. 需求澄清与补充

### 1.1 操作次数（原子化）
**目标：从选词到完成记录的用户操作次数 ≤ 2 次。**
- **操作 1**：选中单词/短语。
- **操作 2**：右键菜单点击“Add to Contextly”。
> 插件后台自动提交，不再需要确认。

### 1.2 数据模型一致性
- `word` 允许短语（如 “prompt engineering”），但必须完整包含在 `context` 中。
- `context` 长度建议限制在 **50~400 字符**，过长则进行句子截断或段落降采样。

### 1.3 去重与规范化
- `word` 保存时统一 **trim + 单空格合并**（多空格折叠为单空格）。
- 以 `word + context + source_url` 的哈希作为去重依据，重复则直接返回成功提示（避免多次记录）。 
- `source_url` 进行去查询参数的规范化（默认去掉 `utm_*` 等追踪参数）。

---

## 2. PocketBase 设计补全

### 2.1 Collection: `words`
| 字段名 | 类型 | 必填 | 说明 | 约束/默认值 |
| --- | --- | --- | --- | --- |
| `word` | Text | 是 | 单词/短语原型 | 1~64 字符，全部小写存储（保留原始大小写另存） |
| `context` | Text | 是 | 含该词的原始句子/段落 | 50~400 字符 |
| `translation` | Text | 否 | 中文释义 | 允许空 |
| `source_url` | URL | 否 | 来源网页链接 | 允许空 |
| `interval` | Number | 是 | 间隔天数 | 默认 `0` |
| `ease_factor` | Number | 是 | 易记因子 | 默认 `2.5`，最小 `1.3` |
| `next_review` | DateTime | 是 | 下次复习时间 | 默认创建时间 |
| `created` | DateTime | 自动 | 系统字段 | PB 自动生成 |
| `updated` | DateTime | 自动 | 系统字段 | PB 自动生成 |

### 2.2 索引与访问控制
- **索引建议**：
  - `next_review` 单字段索引（训练页查询）。
- **权限**：
  - MVP 简化：PocketBase 管理员 token 配置在客户端（不推荐正式环境）。
  - 生产方案：使用 PB 用户鉴权 + 仅访问本人数据。

### 2.3 校验与限制
- `word` 必须在 `context` 中出现（忽略大小写匹配）。
- 超过 400 字符的 `context` 使用 **句子级截断**（保留包含 `word` 的句子）。
- 防滥用：建议在 PocketBase 层做 **简单限流**（如每分钟 60 次写入）。

---

## 3. 复习算法（SM-2 简化版补充）

### 3.1 评分规则
按钮与评分映射：
- **Again** = 0
- **Good** = 3
- **Easy** = 4

### 3.2 计算逻辑
1. 计算新的 `ease_factor`：
   ```
   EF' = max(1.3, EF + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02)))
   ```
   其中 `q` 为评分（0/3/4）。
2. 更新 `interval`：
   - 如果 `q < 3`：`interval = 0`（立即复习）
   - 如果 `interval == 0`：`interval = 1`
   - 如果 `interval == 1`：`interval = 3`
   - 否则：`interval = round(interval * EF')`
3. 更新 `next_review`：
   ```
   next_review = now + interval (days)
   ```

### 3.3 边界条件
- 若 `context` 中未找到 `word`，则拒绝提交（返回错误信息给客户端）。
- 若 `interval` 计算结果为 0，`next_review` 仍设置为 `now`（允许立刻复习）。

---

## 4. Firefox 插件设计补全

### 4.1 事件流
1. 用户选择文本 -> 触发 `contextmenu`。
2. 右键菜单点击 -> `background.js` 收到指令。
3. `content.js` 读取选区 -> 生成 context -> 发送回 `background.js`。
4. `background.js` 调用 PB API -> 成功后通知。

### 4.2 语境提取策略（更明确）
- 获取选中的 `selection.toString()`。
- 向前寻找最近句号（`.?!。！？`）或换行符 `\n`。
- 向后寻找最近句号或换行符。
- 截取 `[start, end]` 之间内容作为 `context`。
- 若无法找到分隔符，回退到整段文本（closest block element innerText）。

### 4.3 API 约定
**POST** `/api/collections/words/records`
```json
{
  "word": "selected_word",
  "context": "full sentence containing selected_word",
  "source_url": "https://...",
  "interval": 0,
  "ease_factor": 2.5,
  "next_review": "2024-01-01T00:00:00Z"
}
```

### 4.4 错误处理
- API 失败时显示 “Save failed” 通知，用户不需额外操作。
- 当 `context` 过短或无法提取时，阻止提交并提示 “Context too short”。

### 4.5 配置与鉴权
- 插件配置一个 `PB_BASE_URL`（可通过 options 页或环境变量注入）。
- 使用 PB 管理员 token 或用户 token 作为 `Authorization` header（MVP 允许管理员 token）。

---

## 5. React Native App 设计补全

### 5.1 数据拉取
**GET** `/api/collections/words/records?filter=next_review<=now`
- 按 `next_review` 升序排序。
- 单次拉取限制 50 条。

### 5.2 UI/状态机补全
- **状态 1 Cloze**：默认显示 `[____]`，仅 context + source 域名。
- **状态 2 Reveal**：点击显示 `word` + `translation`。
- **状态 3 Feedback**：显示三按钮并提交评分。

### 5.3 更新接口
**PATCH** `/api/collections/words/records/{id}`
```json
{
  "interval": 3,
  "ease_factor": 2.36,
  "next_review": "2024-01-04T00:00:00Z"
}
```

### 5.4 域名展示
- `source_url` 为空时显示 `unknown`，否则显示 `URL.hostname`。

---

## 6. MVP 范围外（明确不做）
- 多人协作/词库共享。
- 社交打卡/排行榜。
- 自动翻译与语音朗读。
- 离线缓存与本地数据库。

---

## 7. 最小可行验收标准
- **插件端**：选词 + 菜单点击 ≤ 2 步，成功后 1 秒内显示保存提示。
- **后端**：可在 PB 管理界面看到新增 `words` 记录。
- **移动端**：至少 3 条记录可进入 Cloze 训练流程，并能更新 `next_review`。

---

## 8. 版本号建议
- `v0.1.0`：仅实现捕捉/存储/训练闭环。
- `v0.2.0`：加入离线缓存与增量同步。

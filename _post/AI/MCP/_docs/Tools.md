___
## 概述

Tools使得server能暴露可执行的函数给clients, 通过tools LLM可以和外部系统交互, 执行计算和在现实世界执行操作
Tools是`model-controlled`, 模型可以自主选择并执行.

## Tools Define

```ts
{
  name: string;          // Unique identifier for the tool
  description?: string;  // Human-readable description
  inputSchema: {         // JSON Schema for the tool's parameters
    type: "object",
    properties: { ... }  // Tool-specific parameters
  },
  annotations?: {        // Optional hints about tool behavior
    title?: string;      // Human-readable title for the tool
    readOnlyHint?: boolean;    // If true, the tool does not modify its environment
    destructiveHint?: boolean; // If true, the tool may perform destructive updates
    idempotentHint?: boolean;  // If true, repeated calls with same args have no additional effect
    openWorldHint?: boolean;   // If true, tool interacts with external entities
  }
}
```
## Tools Discovery and Update

-  Clients can list available tools at any time
-  Servers can notify clients when tools change using `notifications/tools/list_changed`
-  Tools can be added or removed during runtime
- Tool definitions can be updated (though this should be done carefully)

## Tool Annotations

Tool Annotation提供了额外关于工具行为的元数据, 辅助clients理解如何呈现与管理tool, 有如下的用途
1. Provide UX-specific information without affecting model context
2. Help clients categorize and present tools appropriately
3. Convey information about a tool’s potential side effects
4. Assist in developing intuitive interfaces for tool approval

MCP定义的Tool Annotation

| Annotation        | Type    | Default | Description                                                                                                                          |
| ----------------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `title`           | string  | -       | A human-readable title for the tool, useful for UI display                                                                           |
| `readOnlyHint`    | boolean | false   | If true, indicates the tool does not modify its environment                                                                          |
| `destructiveHint` | boolean | true    | If true, the tool may perform destructive updates (only meaningful when `readOnlyHint` is false)                                     |
| `idempotentHint`  | boolean | false   | If true, calling the tool repeatedly with the same arguments has no additional effect (only meaningful when `readOnlyHint` is false) |
| `openWorldHint`   | boolean | true    | If true, the tool may interact with an “open world” of external entities                                                             |

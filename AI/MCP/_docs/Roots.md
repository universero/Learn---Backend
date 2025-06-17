___
## 概述

Roots定义了servers能执行操作的边界, 提供了client通知server相关资源及其位置
Roots是一个客户端建议服务器应该关注的URI, 当客户端连接到服务器时, 客户端会声明servers应该工作的Roots, 例子如下
```
file:///home/user/projects/myapp
https://api.example.com/v1
```

## 作用

感觉可以简单理解为允许组合的工作目录
1. **Guidance**: They inform servers about relevant resources and locations
2. **Clarity**: Roots make it clear which resources are part of your workspace
3. **Organization**: Multiple roots let you work with different resources simultaneously

常用于定义:
- Project directories
- Repository locations
- API endpoints
- Configuration locations
- Resource boundaries
## 工作方式

When a client supports roots, it:
1. Declares the `roots` capability during connection
2. Provides a list of suggested roots to the server
3. Notifies the server when roots change (if supported)

While roots are informational and not strictly enforcing, servers should:
4. Respect the provided roots
5. Use root URIs to locate and access resources
6. Prioritize operations within root boundaries

___
## 消息格式

MCP 使用 JSON-RPC 2.0 作为消息格式, 传输层负责在发送时将消息转换为JSON-RPC格式, 在接受时将JSON-RPC格式转换为消息. 目前有三种JSON-RPC消息:

- Requests
```ts
{
  jsonrpc: "2.0",
  id: number | string,
  method: string,
  params?: object
}
```

- Responses
```ts
{
  jsonrpc: "2.0",
  id: number | string,
  result?: object,
  error?: {
    code: number,
    message: string,
    data?: unknown
  }
}
```

- Notification
```ts
{
  jsonrpc: "2.0",
  method: string,
  params?: object
}
```

## 内嵌传输类型

### Standard Input/Output (stdio)

通过标准输入输出流进行通信, 对于本地进程交互于命令行工具尤为有用

Use stdio when:
- Building command-line tools
- Implementing local integrations
- Needing simple process communication
- Working with shell scripts

### Streamable HTTP

通过HTTP Post实现客户端到服务端的通信, 以及SSE可选项实现服务端到客户端通信

Use Streamable HTTP when:
- Building web-based integrations
- Needing client-server communication over HTTP
- Requiring stateful sessions
- Supporting multiple concurrent clients
- Implementing resumable connections

#### Session Management

Streamable HTTP支持有状态的session来维持上下文
1. **Session Initialization**: Servers may assign a session ID during initialization by including it in an `Mcp-Session-Id` header
2. **Session Persistence**: Clients must include the session ID in all subsequent requests using the `Mcp-Session-Id` header
3. **Session Termination**: Sessions can be explicitly terminated by sending an HTTP DELETE request with the session ID

#### 可恢复性和重新交付

为了实现断开后重连, Streamable HTTP提供了
1. **Event IDs**: Servers can attach unique IDs to SSE events for tracking
2. **Resume from Last Event**: Clients can resume by sending the `Last-Event-ID` header
3. **Message Replay**: Servers can replay missed messages from the disconnection point

## 自定义传输

只需实现Transport接口即可自定义传输协议, 常用于如下场景:
- Custom network protocols
- Specialized communication channels
- Integration with existing systems
- Performance optimization

```ts
interface Transport {
  // Start processing messages
  start(): Promise<void>;

  // Send a JSON-RPC message
  send(message: JSONRPCMessage): Promise<void>;

  // Close the connection
  close(): Promise<void>;

  // Callbacks
  onclose?: () => void;
  onerror?: (error: Error) => void;
  onmessage?: (message: JSONRPCMessage) => void;
}
```

## 错误处理

Transport需要处理如下的错误情况:
1. Connection errors
2. Message parsing errors
3. Protocol errors
4. Network timeouts
5. Resource cleanup
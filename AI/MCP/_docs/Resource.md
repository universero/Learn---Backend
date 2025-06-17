___
## 概述

Resources是MCP的基本组成之一, 允许server开放数据和内容供clients使用和作为LLM交互的上下文.
Resource被设计成`application-controlled`, 具体的使用方式由应用程序决定, 可能是由用户选择, 可能是启发式选择, 也可能是模型自己选择. server的作者应该对以上各种使用方式做好准备, 为了自动地把资源公开给模型, 应该使用`model-controlled`的primitive如tools

Resources 代表希望公开给client的任意类型的数据, 可以包括
- File contents
- Database records
- API responses
- Live system data
- Screenshots and images
- Log files
- And more
所有的资源通过独一的URI标识
## Resource URI

```
[protocol]://[host]/[path]
```
For example:
- `file:///home/user/documents/report.pdf`
- `postgres://database/customers/schema`
- `screen://localhost/display1`
protocol和path structure 取决于MCP server的实现, server可以自定义URI scheme

## Resource Types

资源可以包含两种类型的内容: text和binary
### Text Resource

Text resources contain UTF-8 encoded text data. These are suitable for:
- Source code
- Configuration files
- Log files
- JSON/XML data
- Plain text
### Binary Resource

Binary resources contain raw binary data encoded in base64. These are suitable for:
- Images
- PDFs
- Audio files
- Video files
- Other non-text formats
## Resource Discovery

clients通过如下的方式发现可用的资源

### Direct Resource

服务端通过`resource/list`开放一系列具体的资源, 每个资源包含如下内容
```ts
{
  uri: string;           // Unique identifier for the resource
  name: string;          // Human-readable name
  description?: string;  // Optional description
  mimeType?: string;     // Optional MIME type
  size?: number;         // Optional size in bytes
}
```
### Resource Template

对于动态资源, 服务器可以暴露URI Template, client可以通过模板来组装有效的资源URI
```ts
{
  uriTemplate: string;   // URI template following RFC 6570
  name: string;          // Human-readable name for this type
  description?: string;  // Optional description
  mimeType?: string;     // Optional MIME type for all matching resources
}
```

## Reading Resources

clients通过`resources/read`请求来获取资源, server响应如下的内容
```
{
  contents: [
    {
      uri: string;        // The URI of the resource
      mimeType?: string;  // Optional MIME type

      // One of:
      text?: string;      // For text resources
	      blob?: string;      // For binary resources (base64 encoded)
    }
  ]
}
```

## Resource Updates

MCP通过如下两种机制支持实时的资源更新
### List Changes

Server 可以通过`notifications/resource/list_changed`通知clients可用资源列表改变
### Content Changes

Clients可以订阅特定资源的更新
- clients 发送`resources/subscribe`和特定的URI
- 当资源更新时, server 发送`notifications/resources/updated`
- clients使用`resources/read`获取最新的数据
- clients使用`resources/unsubscribe`取消订阅
___
本文用Cherry-Studio作为MCP-Client并编写了一个简单的MCP-Server实现大模型调用计算器

## Client准备

Cherry-Studio是一个开源的MCP-Client, 集成了不少的功能, 可以用来作为我们MCP-Server的使用测试
如果嫌麻烦, 可以使用MCP官方提供的一个可视化的[测试工具]([modelcontextprotocol/inspector: Visual testing tool for MCP servers](https://github.com/modelcontextprotocol/inspector)), 这个更为轻量级, 也不需要实际调用AI, 不过还是前者更为直观

## Server编写

这里使用了开源的MCP-go框架实现了一个简单的计算器, 能让大模型调用来实现加减乘除
```go
package main

import (
	"context"
	"fmt"

	"github.com/mark3labs/mcp-go/mcp"
	"github.com/mark3labs/mcp-go/server"
)

func main() {
	// Create a new MCP server
	s := server.NewMCPServer(
		"Calculator Demo",
		"1.0.0",
		server.WithToolCapabilities(false),
		server.WithRecovery(),
	)

	// Add a calculator tool
	calculatorTool := mcp.NewTool("calculate",
		mcp.WithDescription("Perform basic arithmetic operations"),
		mcp.WithString("operation",
			mcp.Required(),
			mcp.Description("The operation to perform (add, subtract, multiply, divide)"),
			mcp.Enum("add", "subtract", "multiply", "divide"),
		),
		mcp.WithNumber("x",
			mcp.Required(),
			mcp.Description("First number"),
		),
		mcp.WithNumber("y",
			mcp.Required(),
			mcp.Description("Second number"),
		),
	)

	// Add the calculator handler
	s.AddTool(calculatorTool, func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
		// Using helper functions for type-safe argument access
		op, err := request.RequireString("operation")
		if err != nil {
			return mcp.NewToolResultError(err.Error()), nil
		}

		x, err := request.RequireFloat("x")
		if err != nil {
			return mcp.NewToolResultError(err.Error()), nil
		}

		y, err := request.RequireFloat("y")
		if err != nil {
			return mcp.NewToolResultError(err.Error()), nil
		}

		var result float64
		switch op {
		case "add":
			result = x + y
		case "subtract":
			result = x - y
		case "multiply":
			result = x * y
		case "divide":
			if y == 0 {
				return mcp.NewToolResultError("cannot divide by zero"), nil
			}
			result = x / y
		}

		return mcp.NewToolResultText(fmt.Sprintf("%.2f", result)), nil
	})

	// Start the server
	if err := server.ServeStdio(s); err != nil {
		fmt.Printf("Server error: %v\n", err)
	}
}

```
上述的Server使用的stdio的通信方式, 最好是编译成一个二进制文件, 然后在Client中直接启动exe即可

## 添加Server

在Cherry-Studio的设置中选择MCP/添加MCP Server, 将编译出来的exe的绝对路径填写在命令那一栏, 即可成功添加一个MCP Server
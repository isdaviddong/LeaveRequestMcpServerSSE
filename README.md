# McpServer - 請假系統 MCP 伺服器

這是一個基於 Model Context Protocol (MCP) 的 ASP.NET Core 伺服器，提供請假管理相關的工具函數。

## 專案概述

本專案實作了一個 MCP 伺服器，透過 Server-Sent Events (SSE) 協定提供以下功能：
- 查詢員工請假天數
- 提交請假申請
- 取得當前日期時間

## 技術架構

### 開發環境
- **.NET 10.0** - 最新的 .NET 框架
- **ASP.NET Core** - Web 應用程式框架
- **ModelContextProtocol (v0.5.0-preview.1)** - MCP 核心函式庫
- **ModelContextProtocol.AspNetCore (v0.5.0-preview.1)** - ASP.NET Core 整合套件

### 通訊協定
- **Server-Sent Events (SSE)** - 用於 MCP 伺服器與客戶端之間的即時通訊

## 專案結構

```
McpServer/
├── Program.cs                      # 主程式進入點與工具定義
├── McpServer.csproj               # 專案檔案
├── McpServer.sln                  # 解決方案檔案
├── appsettings.json               # 應用程式設定
├── appsettings.Development.json   # 開發環境設定
├── Properties/
│   └── launchSettings.json        # 啟動設定
└── .vscode/
    └── mcp.json                   # MCP 客戶端設定
```

## 程式碼說明

### Program.cs

#### 1. 應用程式初始化

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddConsole(consoleLogOptions =>
{
    consoleLogOptions.LogToStandardErrorThreshold = LogLevel.Trace;
});
```
- 建立 ASP.NET Core 應用程式建構器
- 設定主控台日誌記錄，將所有 Trace 等級以上的日誌輸出到標準錯誤串流

#### 2. MCP 伺服器設定

```csharp
builder.Services
    .AddMcpServer()           // 註冊 MCP 伺服器服務
    .WithHttpTransport()      // 啟用 HTTP/SSE 傳輸層
    .WithToolsFromAssembly(); // 自動掃描並註冊程式集中的工具
```
- `AddMcpServer()`: 將 MCP 伺服器核心服務加入 DI 容器
- `WithHttpTransport()`: 啟用基於 HTTP 的 SSE 傳輸協定
- `WithToolsFromAssembly()`: 自動掃描當前程式集中標記 `[McpServerTool]` 的方法

#### 3. 中介軟體設定

```csharp
var app = builder.Build();
app.MapMcp();  // 註冊 MCP 端點（通常為 /sse 和對應的 POST endpoint）
app.Run();
```
- 建構應用程式
- `MapMcp()`: 自動對應 MCP 所需的端點路由
- 啟動應用程式

### LeaveRequestTool 類別

此靜態類別包含所有請假相關的工具函數，使用 `[McpServerToolType]` 標記以供 MCP 自動掃描。

#### 工具 1: GetLeaveRecordAmount

```csharp
[McpServerTool, Description("取得請假天數")]
public static int GetLeaveRecordAmount(
    [Description("要查詢請假天數的員工名稱")] string employeeName)
{
    if (employeeName.ToLower() == "david")
        return 5;
    else if (employeeName.ToLower() == "eric")
        return 6;
    else
        return 3;
}
```

**功能說明：**
- 查詢指定員工的已請假天數
- 參數：`employeeName` - 員工姓名（不區分大小寫）
- 回傳值：已請假天數（整數）
- 目前使用硬編碼資料模擬：
  - david: 5天
  - eric: 6天
  - 其他員工: 3天
- **實務應用時應改為查詢資料庫**

#### 工具 2: LeaveRequest

```csharp
[McpServerTool, Description("進行請假，回傳結果")]
public static string LeaveRequest(
    [Description("請假起始日期")] string 請假起始日期,
    [Description("請假天數")] int 天數,
    [Description("請假事由")] string 請假事由,
    [Description("代理人")] string 代理人,
    [Description("請假者姓名")] string 請假者姓名)
```

**功能說明：**
- 提交請假申請
- 參數（全部必填）：
  - `請假起始日期`: 請假開始的日期
  - `天數`: 請假天數
  - `請假事由`: 請假原因
  - `代理人`: 職務代理人姓名
  - `請假者姓名`: 申請人姓名
- 回傳值：
  - 成功：包含完整請假資訊的字串
  - 失敗：錯誤訊息提示必填欄位未填寫
- **實務應用時應將資料寫入資料庫並觸發審核流程**

#### 工具 3: GetCurrentDate

```csharp
[McpServerTool, Description("取得今天日期")]
public static string GetCurrentDate()
{
    return DateTime.UtcNow.AddHours(8).ToString("yyyy-MM-dd HH:mm:ss");
}
```

**功能說明：**
- 取得當前日期時間（台灣時區）
- 回傳值：格式化的日期時間字串 (yyyy-MM-dd HH:mm:ss)
- 時區處理：UTC+8 (台灣標準時間)

## 如何使用

### 1. 安裝相依套件

```bash
dotnet restore
```

### 2. 執行專案

```bash
dotnet run
```

預設會在 `http://localhost:5177` 啟動伺服器。

### 3. 設定 MCP 客戶端

在 `.vscode/mcp.json` 中設定連線資訊：

```json
{
  "servers": {
    "leave-request": {
      "type": "sse",
      "url": "https://your-server-url/sse"
    }
  }
}
```

### 4. 使用範例

透過支援 MCP 的客戶端（如 GitHub Copilot），可以自然語言查詢：

- "eric請了幾天假？" → 呼叫 `GetLeaveRecordAmount("eric")` → 回傳 6
- "幫我請3天假，從明天開始，家中有事，代理人是John" → 呼叫 `LeaveRequest(...)`
- "今天日期是什麼？" → 呼叫 `GetCurrentDate()`

## MCP 註解說明

- `[McpServerToolType]`: 標記類別為 MCP 工具容器
- `[McpServerTool]`: 標記方法為 MCP 工具函數
- `[Description(...)]`: 提供工具或參數的說明，幫助 AI 理解用途

## 開發注意事項

### 日誌記錄
專案設定將所有日誌輸出到標準錯誤串流 (stderr)，便於除錯和監控。

### 擴展建議
目前的實作為示範性質，建議進行以下改進：

1. **資料持久化**
   - 整合 Entity Framework Core
   - 連接實際資料庫（SQL Server、PostgreSQL 等）

2. **身份驗證與授權**
   - 加入使用者驗證機制
   - 實作角色權限控管

3. **業務邏輯增強**
   - 請假審核流程
   - 剩餘假期計算
   - 假別分類（事假、病假、特休等）
   - 請假衝突檢查

4. **錯誤處理**
   - 加入完整的例外處理
   - 回傳結構化的錯誤訊息

5. **單元測試**
   - 為每個工具函數撰寫測試案例

## 相關資源

- [Model Context Protocol 官方文件](https://modelcontextprotocol.io/)
- [ASP.NET Core 文件](https://docs.microsoft.com/aspnet/core)
- [Server-Sent Events 規範](https://html.spec.whatwg.org/multipage/server-sent-events.html)

## 授權

本專案僅供學習與示範用途。

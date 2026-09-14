# 智能 3D 建物模型檢核與自動修復系統 — 開發規格書 (Development Specification)

## 1. 專案概述 (Project Overview)
本專案旨在建置一套基於 **MCP (Model Context Protocol)** 與 **RAG (Retrieval-Augmented Generation)** 架構之智能 3D 建物模型檢核與自動修復系統。系統透過整合 ArcGIS 圖台、GitHub Copilot API/SDK 以及語意檢索技術，提供使用者以對話式 AI 介面進行 3D 建物點位與幾何異常檢測、規範比對與自動化空間數據修復。

### 1.1 系統目標
- **微服務化與漸進式轉型**：在不影響舊版 WebForm (C# .NET Framework 4.8) 圖台運作前提下，以微服務架構開發現代化 Vue 3 前端與 ASP.NET Core 後端。
- **智能對話互動 (AI Agent)**：透過 GitHub Copilot SDK / API 與微軟 Semantic Kernel 實現語意理解、意圖路由與自動工具呼叫。
- **標準化工具驅動 (MCP Protocol)**：透過 Model Context Protocol 將 3D 空間檢核與幾何修復邏輯封裝為標準化 MCP Tools。
- **法規知識庫檢索 (RAG)**：整合向量資料庫 (Vector DB)，實時提供 3D 建物建置規範與拓撲規則依據。

---

## 2. 系統總體架構 (System Architecture)

### 2.1 架構圖 (Architecture Diagram)

```
+-----------------------------------------------------------------------------------+
|                                前端層 (Frontend)                                  |
|  [ 舊版 WebForm (4.8) ]  <--- (postMessage / iframe) --->  [ 新版 Vue 3 + Vite ] |
|                                                           (ArcGIS 3D + 對話UI)    |
+-----------------------------------------------------------------------------------+
                                          |
                                          | RESTful API / SignalR (WebSocket)
                                          v
+-----------------------------------------------------------------------------------+
|                        API Gateway (ASP.NET Core YARP)                            |
+-----------------------------------------------------------------------------------+
       |                                  |                                 |
       v                                  v                                 v
+-----------------------+      +-----------------------+      +---------------------+
|   MCP Tool 微服務     |      |  AI Agent 微服務      |      |   RAG 知識庫微服務  |
|  (ASP.NET Core API)   | <--> | (Semantic Kernel /    | <--> | (Semantic Kernel +  |
| - 3D 拓撲檢核工具     |      |  Copilot API)         |      |  Qdrant / Pgvector) |
| - 自動修復演算法      |      |                       |      |                     |
+-----------------------+      +-----------------------+      +---------------------+
       |                                                                  |
       v                                                                  v
+-----------------------------------------------------------------------------------+
|                              資料與存儲層 (Data Layer)                             |
|    [ PostGIS / Spatial DB ]   [ Vector DB (Qdrant) ]   [ ArcGIS Enterprise ]  |
+-----------------------------------------------------------------------------------+
```

### 2.2 技術棧選型 (Technology Stack)

| 層級 | 技術 / 框架 | 說明 |
| :--- | :--- | :--- |
| **前端 (Frontend)** | Vue 3, Vite, TypeScript, Pinia, Tailwind CSS | 現代化 SPA 應用與對話元件開發 |
| **3D 地圖 SDK** | ArcGIS Maps SDK for JavaScript (4.x) | 3D SceneView, SceneLayer, BuildingSceneLayer 渲染 |
| **後端核心 (Backend)** | ASP.NET Core 9 / 10 (Web API) | 高效能微服務架構與 RESTful API / SignalR 通訊 |
| **API Gateway** | YARP (Yet Another Reverse Proxy) | 負載平衡、路由切換、身份驗證轉發 |
| **AI Agent 框架** | Microsoft Semantic Kernel (.NET) | LLM 流程編排、Memory RAG 整合與 Tool/Function Calling |
| **LLM 服務** | GitHub Copilot API / Extension SDK | 對話生成、程式/工具調用決策 |
| **MCP 通訊** | Model Context Protocol (.NET SDK / JSON-RPC) | AI Agent 與 3D 空間運算工具間的標準化通訊協議 |
| **向量資料庫 (Vector DB)**| Qdrant / PostgreSQL (pgvector) | 空間規範、檢核標準文件與歷史修復案例向量化儲存 |
| **空間資料庫 (Spatial DB)**| PostgreSQL + PostGIS / ArcGIS Enterprise | 儲存 3D 建物 Point/Mesh/3D Tile 資料與邊界空間索引 |

---

## 3. 模組設計與 API 介面規格 (Module & API Design)

### 3.1 前端核心模組 (Frontend Modules)
1. **3D Scene View Coordinator**：負責 3D 建物流覽、點位高亮、圖層切換與幾何模型動態刷新。
2. **AI Agent Chat Widget**：對話介面、流式文字渲染 (Streaming UI)、修復意圖確認卡片。
3. **Bridge Interop Module**：透過 `window.postMessage` 處理新舊系統（WebForm <-> Vue 3）點位與 Session 傳遞。

### 3.2 後端微服務設計 (Backend Microservices)

#### (1) AI Agent Service
* **職責**：接收使用者對話、解析意圖、經由 RAG 補充背景知識、呼叫 MCP Tools 執行實體運算。
* **主要 API**：
  * `POST /api/v1/agent/chat` (HTTP / SignalR Stream)
  * `POST /api/v1/agent/session/reset`

#### (2) MCP Tool Service (MCP Server)
* **職責**：提供符合 MCP 規格之工具集，專門處理空間運算與 3D 模型維護。
* **實作工具定義 (MCP Tools)**：
  * `check_elevation_anomaly(building_id, threshold)`: 檢測建物底面高程是否懸空或沉降。
  * `check_mesh_overlap(building_ids)`: 檢測建物 3D Mesh 是否發生空間自交或重疊。
  * `repair_building_elevation(building_id, target_elevation)`: 自動修正建物高程並重新產生 3D 點位/幾何。
  * `repair_mesh_topology(building_id, repair_mode)`: 自動修復模型拓撲縫隙與重疊問題。

#### (3) RAG Knowledge Service
* **職責**：管理與檢索 3D 建物規範文檔（如《國家 3D 建物建置與檢核標準》）。
* **主要 API**：
  * `POST /api/v1/rag/query`: 輸入對話查詢語意最相符的法規條文。
  * `POST /api/v1/rag/documents/ingest`: 管理員上傳新修訂規範進行向量化 Embeddings。

---

## 4. 舊系統整合與安全機制 (Legacy Integration & Security)

### 4.1 舊版 WebForm (4.8) 串接機制
* **嵌入方式**：舊版 WebForm 以 `<iframe>` 或動態加載 Web Component 方式載入 Vue 3 前端。
* **跨視窗通訊協定 (PostMessage Protocol)**：
  ```json
  {
    "event": "SELECT_BUILDING_POINT",
    "payload": {
      "buildingId": "BLD-2026-0914-001",
      "coordinates": [121.5654, 25.0330, 15.2],
      "featureClass": "Building_3D"
    }
  }
  ```

### 4.2 身份驗證與授權 (Auth)
* 統一採用 **OAuth 2.0 / JWT (JSON Web Token)**。WebForm 登入後核發 JWT Token，Vue 3 與微服務各端點皆驗證該 Token，確保單一登入 (SSO) 與 RBAC 權限控管。

---

## 5. 詳細開發工項拆解 (WBS - Work Breakdown Structure)

以下為專案執行時之完整開發工項清單，分為六大階段：

### 階段一：專案準備與基礎架構搭建 (Infrastructure & Setup)
- [ ] **1.1 環境與 CI/CD 建置**
  - [ ] 建立 Git 儲存庫結構與分支策略 (Gitflow / GitHub Flow)
  - [ ] 配置 Docker / Kubernetes 部署環境與 API Gateway (YARP) 容器化腳本
  - [ ] 建置 CI/CD Pipeline (GitHub Actions / Azure DevOps)
- [ ] **1.2 基礎框架初始化**
  - [ ] 初始化 Vue 3 + Vite + TypeScript 專案與基本 UI 框架 (Tailwind / Pinia)
  - [ ] 初始化 ASP.NET Core 專案結構（採用 Clean Architecture / DDD 模式）
  - [ ] 配置 PostgreSQL + PostGIS 與 Qdrant 向量資料庫容器環境

### 階段二：舊系統整合與 3D 地圖模組開發 (Map & Legacy Integration)
- [ ] **2.1 WebForm 與 Vue 3 通訊橋樑**
  - [ ] 開發 `postMessage` 通訊封裝與事件訂閱機制 (Vue 3 & WebForm 雙向)
  - [ ] 實作 JWT Token 跨系統傳遞與身份認證 Handshake
- [ ] **2.2 ArcGIS 3D 地圖核心開發**
  - [ ] 整合 ArcGIS Maps SDK for JS，建立 3D SceneView 基礎視圖
  - [ ] 實作 3D 建物 FeatureLayer / SceneLayer 載入與視圖跳轉 (Fly-to)
  - [ ] 開發建物點位/模型選取與高亮 (Highlighting) 互動元件

### 階段三：MCP 工具庫與 3D 空間檢核演算法 (MCP Tool Server)
- [ ] **3.1 MCP Server 核心開發**
  - [ ] 基於 .NET 實作 MCP JSON-RPC 通訊協議與 Tool 註冊機制
  - [ ] 設計 MCP Tool 傳入參數與傳出格式標準 (Schema)
- [ ] **3.2 空間檢核與修復演算法**
  - [ ] 開發高程懸空與沉降檢核模組 (`check_elevation_anomaly`)
  - [ ] 開發 3D 幾何自交與重疊檢核模組 (`check_mesh_overlap`)
  - [ ] 開發自動高程對齊與幾何模型修正邏輯 (`repair_building_elevation`)
  - [ ] 實作修復結果回寫 PostGIS / ArcGIS FeatureServer 介面

### 階段四：RAG 知識庫服務開發 (RAG Microservice)
- [ ] **4.1 文件解析與向量化 Pipeline**
  - [ ] 開發 PDF/Word 建物標準法規文檔切分 (Text Chunking) 工具
  - [ ] 整合 Text Embedding 模型產生向量並寫入 Qdrant / Pgvector
- [ ] **4.2 檢索與 Semantic Kernel 整合**
  - [ ] 實作向量混合檢索 (Hybrid Search: Dense + Sparse Vector)
  - [ ] 基於 Semantic Kernel 撰寫 RAG Plugin 與 Prompt Template

### 階段五：AI Agent 服務與對話互動整合 (AI Agent Integration)
- [ ] **5.1 Agent 核心與 GitHub Copilot 整合**
  - [ ] 串接 GitHub Copilot API / Extension SDK 建立 Agent 對話 Engine
  - [ ] 配置 Semantic Kernel 意圖路由 (Intent Routing) 與 Tool Selector
- [ ] **5.2 前端 AI 對話 UI & 流式響應 (Streaming)**
  - [ ] 前端實作類 ChatGPT 之動態對話視窗與 SignalR / SSE 串流接收
  - [ ] 實作「檢核結果確認卡片」與「自動修復對比模式 (Before/After Slider)」

### 階段六：系統整合測試、優化與上線 (Testing, Optimization & Deployment)
- [ ] **6.1 整合與效能測試**
  - [ ] 執行端到端 (E2E) 流程測試：選取點位 -> 對話觸發檢核 -> RAG 說明 -> MCP 自動修復
  - [ ] 3D 模型大資料量載入與渲染效能調優 (3D Tiles / I3S 最佳化)
  - [ ] AI Agent 對話回應時間與 Token 費用/限流調控優化
- [ ] **6.2 部署與文件交付**
  - [ ] 完成 Docker 容器正式環境部署與 Health Check 機制
  - [ ] 撰寫系統操作手冊、API 文件 (Swagger) 與 MCP 工具擴充說明文件

---

## 6. 結論 (Conclusion)
本規格書確立了以 **Vue 3 + ASP.NET Core** 為核心技術棧的微服務轉型路線。透過 Semantic Kernel 串接 GitHub Copilot API，並結合標準化 MCP 工具庫與 RAG 知識庫，系統能在最低破壞現有 WebForm 業務邏輯的前提下，達到 3D 建物模型的高效智能檢核與自動修復。

# Centralized Memory Server — Hướng Dẫn Setup

Hướng dẫn triển khai agentmemory làm server bộ nhớ trung tâm cho nhiều máy client.

## Kiến Trúc

```
┌─────────────────────────────────────────┐
│   Máy Server (VPS / LAN host)           │
│   agentmemory bind 0.0.0.0:3111        │
│   SQLite DB lưu TẤT CẢ memory          │
└──────────────┬──────────────────────────┘
               │ HTTP
     ┌─────────┼──────────┐
     │         │          │
 ┌───┴───┐ ┌───┴───┐ ┌───┴───┐
 │ Máy A │ │ Máy B │ │ Máy C │
 │ IDE   │ │ IDE   │ │ IDE   │
 └───────┘ └───────┘ └───────┘
```

## 1. Setup Server

```bash
# Clone & build
cd /path/to/agentmemory
npm install
npm install @xenova/transformers   # local embeddings (free, offline)
npm run build

# Tạo config
mkdir -p ~/.agentmemory
cat > ~/.agentmemory/.env << 'EOF'
EMBEDDING_PROVIDER=local
AGENTMEMORY_TOOLS=all
CONSOLIDATION_ENABLED=true
EOF

# Start server
npm start
# → Server chạy tại 0.0.0.0:3111

# Verify
curl http://localhost:3111/agentmemory/livez
```

## 2. Setup Client (Mỗi máy IDE)

### Gemini CLI / Antigravity

Thêm vào `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "agentmemory": {
      "command": "node",
      "args": ["/path/to/agentmemory/dist/standalone.mjs"],
      "env": {
        "AGENTMEMORY_URL": "http://<server-ip>:3111",
        "AGENTMEMORY_FORCE_PROXY": "1"
      }
    }
  }
}
```

> **Lưu ý:** Chỉ cần copy file `dist/standalone.mjs` sang máy client. Không cần install toàn bộ project.

### Cursor / VS Code

Thêm MCP server trong settings với cùng cấu hình.

## 3. Cách Memory Hoạt Động

### Tự Động — Không cần gọi thủ công

Khi bạn mở IDE ở thư mục `~/myapp`:

1. **Auto-detect project**: MCP tự lấy basename thư mục → `project = "myapp"`
2. **Auto-session**: Tự tạo session trên server lần đầu
3. **Auto-observe**: Mỗi khi agent gọi bất kỳ MCP tool → tự lưu thành observation
4. **Auto-filter**: Search chỉ trả kết quả thuộc project hiện tại

### Project Scoping

```
~/myapp         → project = "myapp"
~/backend-api   → project = "backend-api"
~/web-frontend  → project = "web-frontend"
```

Cùng tên thư mục trên máy khác nhau = **cùng memory pool**.

### Override Project Name

Nếu muốn đặt tên project tùy ý (không dựa vào tên thư mục):

```json
{
  "env": {
    "AGENTMEMORY_URL": "http://...",
    "AGENTMEMORY_FORCE_PROXY": "1",
    "AGENTMEMORY_PROJECT": "my-custom-name"
  }
}
```

## 4. Troubleshooting

### Không kết nối được

```bash
# Từ máy client, test kết nối
curl http://<server-ip>:3111/agentmemory/livez

# Nếu timeout: kiểm tra firewall
# macOS: System Settings > Firewall
# Linux: sudo ufw allow 3111
```

### MCP fallback về local

Nếu thấy log:
```
[@agentmemory/mcp] no server reachable; falling back to local InMemoryKV
```

→ Kiểm tra `AGENTMEMORY_URL` đúng IP:port và `AGENTMEMORY_FORCE_PROXY=1` đã set.

### Viewer

Viewer web tại `http://<server-ip>:3113` — xem tất cả sessions, observations, memories.

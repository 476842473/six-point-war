# 六点战场（Six-Point War）

6×6 双人卡牌炮击战棋网页游戏。掷骰摸牌 → 打出同点数卡牌炮击格子 → 歼灭对方全部士兵获胜。支持人机（三档难度）与房间码联机对战（PeerJS P2P，无需服务器）。

## 快速开始

```bash
python3 -m http.server 8000
# 打开 http://localhost:8000
```

单文件原生 HTML/JS/CSS，无框架、无构建。联机需外网（加载 PeerJS CDN + 公共信令云）。

## 目录

| 文件 | 说明 |
|------|------|
| `index.html` | 游戏全部代码 |
| `sfx/` | AI 生成的音效与 BGM |
| `AGENTS.md` | 给 AI 编码助手读的项目交接文档（架构/规则/约定） |

## 开发

- 规则与架构约定见 `AGENTS.md`（改动前先读，勿回退已定稿规则）
- 测试：Playwright 双页面模拟联机双方

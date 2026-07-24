# 六点战场（Six-Point War）— 项目交接文档

6×6 双人卡牌炮击战棋网页游戏。**单文件原生 HTML/JS/CSS，无任何框架和构建步骤**，联机通过 PeerJS（WebRTC P2P + 公共信令云）实现，无后端。

## 文件结构

```
index.html   # 全部游戏（HTML + CSS + JS 一体，约 2000 行）
sfx/         # 11 个 AI 生成的音效/音乐 mp3
  dice cannon explosion pickup card shield win lose turn summon bgm
AGENTS.md    # 本文件
```

## 本地运行与测试

- 预览：`python3 -m http.server 8000` 后开 `http://localhost:8000`（必须走 http，file:// 音频和联机会有问题）
- 联机依赖外网（unpkg CDN 加载 PeerJS + 0.peerjs.com 信令云）
- 测试：Playwright（Chromium），可开双页面模拟联机双方。游戏状态挂在全局 `S`、配置在 `CONFIG`、联机在 `NET`，测试可直接 `page.evaluate` 操作

## 核心架构

### 状态对象 S（每局 initGame 重建）
`turn, cells[36]{idx,r,c,value,soldier,item,rareItem,rareTarget,crater,shield}, hand, items, selCard, selItem, swapSrc, played, needPlay, guidedNext, chainNext, revealAi, busy, over, endInfo, discarded, discardMode, discardSel, forcedDiscard, drawBonus, bonusLeft, bonusOn, rareOpen, rareSel, playedCards, stats{kills,items,rares,rounds,start}`
开局配置：`CONFIG = { difficulty:'easy|normal|hard', soldiers, boxes, plays }`

### 回合流程
`beginPlayerTurn → startTurn → diceAndDraw(掷骰摸牌+spawnBox刷箱) → 玩家行动(playCard/onItemClick/tryOpenRare/doDiscard) → endTurn → (人机: aiTurn / 联机: netSync 交接)`。
回合开始掷 N=2+bonusOn?drawBonus:0 颗骰子摸牌；抽牌无上限，**回合结束时手牌须 ≤5**，超出进强制弃牌（enterForcedDiscard）。

### 关键规则（已定稿，勿回退）
- 出牌点数须等于格子点数；**不能打己方士兵**；连环雷溅射会误杀己方
- 弃牌：每回合 1 次弃 2 抽 1（重抽必异）；**三连兑换：选 3 张相同点数的牌可换 1 张任意点数牌**（消耗同一次弃牌机会，玩家专属，AI 不用）
- 稀有箱：2 张牌点数和精确等于目标(7~12)；**出过牌不可开箱；开箱耗 1 次出牌**；精准制导可免牌直开任意箱
- 出牌次数用完且手里有道具时，不自动结束回合（等手动）；此时卡牌类道具(制导/连环)不可用，弹药补给可用
- 道具池：scout/chain/ammo/guided/shield/reinforce/desperate(变废为宝)/sacrifice(身先士卒,+1出牌)/swap(移型换位,护盾跟随士兵)
- **道具去重**：入手瞬间 pickNoDup() 检查，已持有同类则转换为未持有种类（9 种全持有才允许重复）
- **联机先后手**：CONFIG.firstMove = random(默认)/alternate(轮换,localStorage spw_first_last)/host(主机先手)；主机 initGame 决定 iFirst，后手方开局 1 兵带盾；主机后手时 S.turn='ai'+busy 锁定并 netSync 等对方先动
- **成就系统**：16 个成就（ACHIEVEMENTS 表），unlockAch(id) 判重+弹卡+记入 S.newAch；统计字段在 S.stats（shieldBlocks/turnKills/sacUsed/swapVictim/minMe）；结算类由 checkGameEndAch 统一判定；存储 localStorage spw_ach，联机连胜 spw_net_streak；成就纯本地判定不走网络同步
- 稀有道具：shieldSummon/doubleSummon/drawBonus(之后3回合+1抽)
- 无空地时按剩余兵力结算；人机/联机均随机先后手（联机由主机配置决定）；先后手平衡：先手方首回合出牌数 -1（至少 1 次，对 AI 同样生效），后手 1 兵开局带盾

### 联机（重要）
- **状态传递模型**：双方各自以本地视角存 S（自己永远 'player'），动作后 netSync 发完整状态，接收方 `swapState()` 翻转视角；日志经 `swapNames()`（你↔对手）转换
- **netSync 只能在动作点显式调用，绝不放 render()**（防收发乒乓死循环）
- 音效经 FXQ 队列随状态同步给对端播放；endTurn 分发器统管人机/联机
- 房间码 = PeerJS id `spw6-XXXX`；结算 endInfo 本地视角，接收方翻转为 showEndOverlay(ei.a,ei.p,kind,false)
- 客人只读配置（.guestView + JS 双重拦截），主机 sendCfg 实时同步
- swapState 新增字段时记得同步处理（见函数内注释）

### 修改约定（血泪教训）
- 对 index.html 的编辑**必须串行**（一次一个编辑），并行编辑会竞态丢失；每批改完 grep 验证 + node --check 提取的 JS
- 文案全中文；SIDE_NAME 控制「电脑/对手」称呼，勿硬编码"电脑"（联机复用）

### 音效
`sfx(name)` 本地播并入 FXQ 队列；`playSound(name)` 只播不入队（对端回放用）；BGM 单独开关。新音效可用音频生成插件补齐。

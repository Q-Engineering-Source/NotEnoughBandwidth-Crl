# 把 NotEnoughBandwidth 回移植到 MC 1.12.2 / Cleanroom-only：可行性分析（修订版 v2）

> 本文件是 <https://mclo.gs/88eKcWz> 的修订版。原文档的**结构、章节顺序和全部设计主张都保留**，只做两类改动：
> ① 纠正经一手源码核验后不成立或有偏差的**事实陈述**；② 给每条关键结论补上可复查的**证据出处**（仓库 / 文件 / 行号 / commit），并对版本不同源的证据加**版本保留**标注。
>
> 标注约定：
> - 【证实】——已用上游一手源码或官方元数据核验为真；
> - 【修正】——原文档陈述有误或过强，已改写；
> - 【版本保留】——证据取自相邻版本（如 MC 1.12 的 MCP 9.40 反编译源）而非 1.12.2 逐字原文；
> - 【推断】——由读码推理得出，未经运行时验证；
> - 【主张】——设计建议，无所谓真伪，只标可行性判断的依据强度。
>
> 核验只做只读取证，未克隆仓库、未修改任何文件、未运行游戏。核验时间与所引 commit 见文末「证据索引」。

## 修订记录（本版相对原文档的实质性改动）

| # | 原文档位置 | 原文陈述 | 修订后 | 依据 |
|---|---|---|---|---|
| 1 | 结论表最后一行 | `dev-1.12.2forge` **没有实现** | **曾完整实现约一个月（2026-04-12 ~ 05-22），2026-05-24 用一次空树提交整体删除并放弃** | `7b5d298` 的 tree = `4c…`空树、0 additions / **4703 deletions**、message `no 1.12.2!`；被删文件含真实实现 |
| 2 | 结论第 2 段 | 21.1.12「还继续调整 chunk-reference cache」 | 21.1.12（`21d8d48`，74 行）**没有新功能**：仅默认值 4096/2048→8192 + 配置版本迁移 + 黑名单加 `minecraft:keep_alive`；三功能属 21.1.11 | 该 commit 全量 diff |
| 3 | §2 裁决 | 第一版**没必要**移植 `NamespaceIndexManager` | 降级为**可争议**：被删的 1.12.2 分支确实移植过 `indextype/NamespaceIndexManager` 等 | `b481d9e` 的文件树 |
| 4 | §3 / §5 / §6 / §7 / §8 / §11 | 若干陈述措辞粗（"全局线程池""4MB""removePlayer 会重发全量"等） | 逐条精确化：**单线程**定时器、`2^21−1` 帧硬上限、`sentToPlayers` 才是门控 | 见各节 |

---

## 结论

**如果"更新"指上游主线、Minecraft 目标版本和核心协议演进，那么 `USS-Shenzhou/NotEnoughBandwidth` 明显更新。** 【证实】

我直接核对了两个仓库当前 Git 状态：

| 代码线 | 当前状态 | 最近提交 | 判断 |
|---|---|---|---|
| `USS-Shenzhou/NotEnoughBandwidth` `master` | Minecraft **26.2**、NeoForge **26.2.0.88**、Java 25、`mod_version=26.2-0+beta` | `e24108b`，2026-09-21T06:55:34Z，`port to 26.2` | **最新上游 NEB** |
| `xianaldai/NEBreload` `dev-1.21.1neoforge` | MC 1.21.1、NeoForge 21.1.21、NEBL `21.1.12-neoforge+1.21.1` | `21d8d48`，2026-08-19T15:41:56Z | 较新的 Legacy 分支 |
| `xianaldai/NEBreload` `dev-1.20.1forge` | MC 1.20.1 Forge 47.4.0、NEBL `20.1.10-forge+1.20.1+47.4.0` | `f74900d`，2026-06-11T05:26:52Z | **对 1.12.2 Forge 回移植最有参考价值** |
| `xianaldai/NEBreload` `dev-1.12.2forge` | **曾完整实现后被删除**：2026-04-12 `4a76efb "init"` → 05-22 `b481d9e` → 05-24 `7b5d298` 清空 | `7b5d298`，2026-05-24T13:17:50Z，提交信息 `no 1.12.2!`，tree 是 Git 空树（0 additions / **4703 deletions**） | 不能作为基线，**但是极有价值的"已知坑"清单** |

【修正】第 4 行。原文档据"空树 + `no 1.12.2!`"推断"没有实现"，实际上该分支在 2026-04-12 至 05-22 之间有**完整实现**并被主动废弃。证据：

- `7b5d29813069cf4c6f7565a71fda219a18e7b734`：`commit.tree.sha = 4b825dc642cb6eb9a060e54bf8d69288fbee4904`（git 规范空树），stats `0 additions / 4703 deletions`，`contents?ref=dev-1.12.2forge` 返回 `[]`。
- 被删的真实代码（非脚手架）：`src/main/java/cn/ussshenzhou/notenoughbandwidth/client/ClientLifecycleHandler.java`、`indextype/NamespaceIndexManager.java`、`indextype/ConnectionIndexTable.java`、`indextype/CustomPacketPrefixHelper.java`。
- 父提交的构建事实：`version = '12.2.02-forge+1.12.2+23.5.2860'`、`net.minecraftforge:forge:1.12.2-14.23.5.2860`、`com.github.luben:zstd-jni:1.5.5-5`、Java 8、Gradle 4.9、Mixin 0.8.5，CI 名为 "Build on Java 8"。
- 提交链：`4a76efb "init"`(04-12) → `017a217 "maybe have some bugs..."` → `cc3167a "fix some bugs and update more feature"`(04-14) → `606d679`/`682cda7`/`ac34a3a`/`24488da "fix some bugs, rewrite some classes"`(05-19~20) → `a880bbc`/`b481d9e`(05-22) → `7b5d298`(05-24) 清空。

**这条修订本身就把本文档的价值提高了**：那份 1.12.2 前作留下的 `ClientLifecycleHandler` 类注释直接印证了 §3 的判断（原文）：

> 「Forge 1.12.2 没有现代 Forge 的登录后协商补发钩子，这里在客户端连入世界后，若仍未拿到 NEB transport，则周期性补发一次轻量 setup 请求，兼容代理链路。」

常量 `MAX_SETUP_RETRIES = 12`、`RETRY_INTERVAL_TICKS = 10`，在 `TickEvent.ClientTickEvent`（Phase.END）里补发，且补发前要求 `channel().attr(NetworkManager.PROTOCOL_ATTRIBUTE_KEY).get() != EnumConnectionState.PLAY` 即直接 return。**它同样绕过了"用 EnumConnectionState 判断就绪"这个坑，但用的是"连入世界后轮询补发"这种更糙的办法。**

不过，**NEBL 并不是单纯"旧版 NEB"**。它已经分叉出了自己的功能：`dev-1.21.1neoforge` 有上游当前 README/源码里没有的 **packet dictionary、chunk-reference cache、streaming Zstd** 等扩展。因此：

> **上游 NEB = 更新的核心主线；NEBL = 较旧平台上的分叉增强版。**

【修正】原文档在此追加的"21.1.12 还继续调整 chunk-reference cache"应删去或改写。`21d8d48`（作者 MapleSugar365，2026-08-19，仅 74 行改动）的内容是：

- `chunkReferenceMaxServerEntries` 4096 → 8192、`chunkReferenceMaxClientCache` 2048 → 8192；
- 引入 `CURRENT_CONFIG_VERSION = 1.0` 与配置字段 `configVersion`，新增 `migrate()` 一次性迁移（`ConfigHelper` 调 `newInstance.migrate()`，`TConfig` 增 `default void migrate()`）；
- `COMMON_BYPASS_LIST` 增加 `"minecraft:keep_alive"`，README 同步。

三个新功能实际是 **21.1.11** 引入的（见 changelog 旧条目）。另注意 `streaming` 默认值是 `false`（`public boolean streaming = false;`）。

【证实 + 补强】上游目前也不是严格包含 NEBL 的所有修复。USS 当前 `ZstdHelper.useContext()` 对 `playersDoNotUseContext` 直接返回 `contains(uuid)`，从字段语义看**确实反了**；NEBL 写成 `!playersDoNotUseContext.contains(uuid)`。所以不能简单做"拿最新上游全部覆盖 NEBL"。逐字证据见 §1 末尾的「附：上游 `ZstdHelper.useContext()` 反转 bug」。

---

# 移植到 1.12.2 Cleanroom-only：可行，而且值得做

我的判断是： 【主张】

> **核心 NEB 功能可行性很高；完整复刻 NEBL 21.1.12 的全部扩展则是中高难度项目。**
>
> 最合理的方案不是把任意一个仓库机械降版本，而是为 **MC 1.12.2 / Cleanroom / Java 25** 重新实现一个网络适配层。

Cleanroom-only 反而比"兼容原版 Forge 1.12.2 + Java 8"容易，因为你可以固定 Java 25、Mixin/CleanMix、现代 Netty 环境和确定的加载器，不需要兼容所有旧启动方式。【证实前提】Cleanroom README 自述：1.12.2 生态、基于 MinecraftForge 的延续、Runtime **Java 25+**、LWJGL 3 + SDL 3、CleanMix（Mixin fork）、**Netty 4.2**。

【修正 + 版本保留】关于 `6bb602ec6f0c8c92545818655b1cdd80aba9a31f`（`pack: project overhaul (CleanroomGradle, LWJGLY, SDL 3)`，Rongmario，2026-09-23T19:15:44Z，`+39906 / −34158`，74064 行）：说它"对 NEB 几乎没有直接影响"**只对运行时成立**。该提交同时 `refactor: rewritten buildscripts`、`refactor: removed buildSrc/`、`fix(gradle): stop run tasks from inheriting mixin debug args by default`、`test(junit): updated to JUnit 6`。对本项目而言，**开发/测试流程会直接受影响**（构建脚本结构、`runClient` 的 mixin 调试参数默认值、测试框架），所以"影响小"要限定为"对 NEB transport core 的架构影响小"。NEB 核心碰的是 Netty、FML handshake、packet encoder/decoder、chunk tracking；只有统计界面的快捷键属于普通客户端输入，完全没必要直接碰 SDL。

---

## 1. 最应该移植的核心：聚合 + Zstd

这是我认为最确定的一部分，**高可行**。【证实】

现代 NEB 在 `Connection.send()` 截获 packet，每约 20 ms 聚合后 Zstd 压缩。核对：`AggregationFlushHelper.getFlushPeriodInMilliseconds()` 返回 `20`；`ConnectionMixin` 注入 `Connection.send(Packet, ChannelFutureListener, Z)` 的 `@At("HEAD")`、`cancellable = true`，命中后 `AggregationManager.takeOver(...)` + `ci.cancel()`；压缩在 `PacketAggregationPacket.encode()` 中，且只有 `rawSize >= 32` 才压缩。

但在 1.12.2 Forge/Cleanroom，我反而不建议照搬这个注入点。【证实】

因为 1.12.2 有一个很重要的结构：

```text
Mod IMessage
   ↓
SimpleNetworkWrapper
   ↓
FMLProxyPacket
   ↓
NetworkDispatcher
   ↓
SPacketCustomPayload / CPacketCustomPayload
   ↓
NettyPacketEncoder
```

而 `FMLProxyPacket.writePacketData()` 本身就是 **NOOP**。

【证实】逐字命中。类在 `net.minecraftforge.fml.common.network.**internal**.FMLProxyPacket`（注意是 `internal` 子包，`module/forge/.../fml/common/network/` 下**没有**这个文件），`implements Packet<INetHandler>`。其 `readPacketData` / `writePacketData` 方法体是纯注释：

```java
// NOOP - we are not built this way
```

**这里要补一句原文档没说的话**：它**不是**"没覆盖父类行为"，而是**显式 override 之后留空**。差别很重要——任何依赖 `Packet` 序列化路径的通用代码都会**静默产生 0 字节包**，不报错、不抛异常。真正承载数据的是 `payload()`（返回 `ByteBuf`）、`toC17Packet()`（→ `CPacketCustomPayload`）、`toS3FPackets()`（→ 分片的一组 `SPacketCustomPayload`）。

所以如果像现代 NEB 一样过早在：

```java
NetworkManager.sendPacket()
```

捕获所有包，就会碰到 `FMLProxyPacket`，不能按普通 vanilla packet 序列化。

这也解释了为什么 NEBL 的 Forge 1.20.1 分支不得不做很多 Forge-specific 绕路，甚至现在对绝大多数非 NEB CustomPayload 直接 bypass。【证实 + 加强】实际比"绝大多数"更强：NEBL 1.20.1 的 `ConnectionMixin.shouldSkipAggregation` 对**双向的所有 CustomPayload 一律直通**（命中即先 `AggregationManager.flushConnection(connection)`），注释原文：「其它 mod 的 Forge custom payload 往往承载 GUI、菜单、按钮交互、能力同步等强上下文消息。这些消息必须尽量保持 Forge 原生 NetworkHooks.onCustomPayload 分发语义，不能被 NEB 聚合后重建/回放。」另有 7 个硬编码直通包（`ClientboundAddEntity`/`MoveEntity`/`TeleportEntity`/`SetEntityMotion`/`RemoveEntities`/`BlockUpdate`/`SetCamera`）。

### 1.12.2 有一个更好的 Cleanroom-only 解法

在 FML handshake 完成之后，向真实 NetworkManager channel 插一个 **ChannelDuplexHandler**：

```text
decoder
   ↓
NEB handler
   ↓
fml:packet_handler
   ↓
packet_handler
```

管线方向非常合适。出站是逆序：

```text
packet_handler
   ↓
fml:packet_handler
   ↓
NEB handler
   ↓
encoder
```

【证实 + 可执行配方】

1. **1.12 的管线名与顺序**（服务端 `NetworkSystem` 初始化）：`timeout, legacy_query, splitter, decoder, prepender, encoder, packet_handler`（客户端同序、无 `legacy_query`）。其中 `splitter`/`prepender` 是**帧处理器**（`NettyVarint21FrameDecoder`/`Encoder`），`decoder`/`encoder` 是**包编解码器**（`NettyPacketDecoder`/`Encoder`），帧长前缀由 `prepender` 写。
2. **Forge 自己就用 `addBefore`**：`NetworkDispatcher.insertIntoChannel()` 做 `manager.channel().pipeline().addBefore("packet_handler", "fml:packet_handler", this);`，于是 modded 连接变成 `..., encoder, fml:packet_handler, packet_handler`。
3. **正确插入位置是 `addBefore("fml:packet_handler", "neb")`**：出站（tail→head：`packet_handler → fml:packet_handler → NEB → encoder`）看到的是 `NetworkDispatcher.write()` **转换后**的 vanilla `CPacketCustomPayload` / `SPacketCustomPayload`；入站（head→tail：`decoder → NEB → fml:packet_handler → packet_handler`）看到的是 FML **转换前**的 vanilla custom payload。**两个方向都不会出现 `FMLProxyPacket`。**用 `addAfter` 则入站会拿到已转换的 `FMLProxyPacket`，不行。
4. **前提条件**：`fml:packet_handler` 只在 modded 握手走到 `insertIntoChannel()` 之后才存在——所以"握手完成后再插"不是优化，而是**必要条件**（插入时机与 §3 同源）。
5. 参考先例：`PacketLoggingHandler` 里还有 `pipeline.replace("splitter", "splitter", …)` / `replace("prepender", "prepender", …)`——可见动帧处理器并非无人涉足，但这也意味着任何改动都要处理与它的插入顺序冲突。
6. 已存在的同类条件 handler：`decrypt/encrypt`（`addBefore("splitter","decrypt")`、`addBefore("prepender","encrypt")`）、`compress/decompress`（`addBefore("decoder","decompress")`、`addBefore("encoder","compress")`）。

所以 NEB 收到的已经不是 `FMLProxyPacket`，而是 Forge 转换完的 `SPacketCustomPayload` / `CPacketCustomPayload` / 普通 vanilla Packet，可以统一处理。

入站又是正序，NEB 解聚合以后直接 `ctx.fireChannelRead(innerPacket);`，那么 FML 自定义包继续走原本 `NetworkDispatcher`，不会绕过：SimpleNetworkWrapper / FMLEmbeddedChannel / IMessageHandler / FML channel semantics。

**这比照搬现代 NEB 的 `ConnectionMixin` 更适合 1.12.2。**

#### 附：上游 `ZstdHelper.useContext()` 反转 bug（本次核验的最强单点证据）

【证实，且为真 bug】`src/main/java/cn/ussshenzhou/notenoughbandwidth/zstd/ZstdHelper.java`（84 行，blob sha `1810e32b8c8f53f528b75780e2c939dfb73ad7bf`）第 64–83 行，**第 80 行**：

```java
var use = NotEnoughBandwidthConfig.get().playersDoNotUseContext.contains(uuid.toString());
```

缺 `!`。完整语义链：`getReceiving() == PacketFlow.CLIENTBOUND → true` → `server == null → true` → 查 `CONNECTION_USE_CONTEXT` 缓存提前返回 → 按 players 流找 `EntityPlayerMP`（找不到则 `put(connection, true); return true`）→ 取 UUID → 上面那行 → `CONNECTION_USE_CONTEXT.put(connection, use); return use;`。返回值喂给 `new Context(useContext(connection))`，`Context.compress` 中 `if (useContext) { …compressDirectByteBufferStream(dst, raw, EndDirective.FLUSH);… } return compressCtx.compress(raw);`，即 `true` = 复用流式 Zstd 上下文。

判为真 bug 的三条理由：

1. 与字段名和 README 语义相反，应为 `!contains`（README 说这是给 Replay 类"必须看到原始包流"的 mod 用的豁免名单）；
2. 字段与方法由**同一提交** `36f6fe5`（message `potential replay compatibility problem`）同时引入，该 diff 从落地起就是正向 `contains`，**从未出现过 `!`**；
3. 该特性的本意（Replay 等依赖包重放的 mod）恰恰需要这些玩家**不复用**上下文，当前实现是反效果。

但是【推断】：因两处提前 `return` 与缓存，UUID 分支实际很难触达（默认名单只含零 UUID，且全仓库**没有任何 add/remove 调用点**——无命令、无 GUI、无 mixin 写它，只能手改 JSON），所以这个反转**可能潜伏不可观测**。这也解释了它为什么能一直留在 master 上。NEBL 侧的对照实现是 `var use = cfg.streaming && !cfg.playersDoNotUseContext.contains(uuid.toString());`——**原文档摘录漏掉了 `cfg.streaming &&` 这一半条件**，补上后语义才完整（streaming 关闭时一律不复用上下文）。

---

# 2. 甚至不需要现代 NEB 的 PacketType/Identifier 体系

现代 NEB 很大一部分复杂度来自 `CustomPacketPayload.Type` / `Identifier, ResourceLocation` / `ProtocolInfo` / `IdDispatchCodec` / NeoForge NetworkRegistry —— 1.12.2 全都没有。但反而可以更简单：1.12.2 已有 `EnumConnectionState.PLAY` + `EnumPacketDirection` + `packet id`，因此一个聚合帧完全可以按 `count/sequence → (packet-id, payload-size, writePacketData…)` 组织，接收端 `packet-id → EnumConnectionState.PLAY 创建 packet → readPacketData() → ctx.fireChannelRead(packet)`。

### 这意味着一个很重要的裁决

【修正：从"没必要"降级为"可争议"】**第一版 1.12.2 NEB 可以先不移植 `NamespaceIndexManager` 与现代 CustomPayload header compression**——理由是收益判断（见下），而**不是**因为它在 1.12.2 上做不到。反证就在这个仓库自己的历史里：

`dev-1.12.2forge` 被删的前作**确实移植了** `indextype/NamespaceIndexManager.java`、`ConnectionIndexTable.java`、`CustomPacketPrefixHelper.java`。也就是说"1.12.2 不存在该体系、无从移植"这个隐含前提不成立（`indextype` 那一套是 NEBL 自造的 index/编号层，可在 1.12.2 上重写）。因此本节的结论应表述为：**收益 vs 风险的权衡 → 延后**，而不是"不需要"。

原版 packet ID 已经非常紧凑。Forge mod packet 虽然还有 `SPacketCustomPayload { channel String, payload }`，但 1.12.2 的 channel 名本来就很短，而且这些字符串被放进同一个 Zstd batch 后高度重复，Zstd 自己就会把它们压得很好。所以为了每个包再省几字节，去改 `FMLProxyPacket` / `SPacketCustomPayload` / `CPacketCustomPayload` / `REGISTER` / `UNREGISTER`，**风险远大于收益**，这部分可以放到最后。

---

# 3. 1.12.2 的 FML handshake 是最大的语义边界

**【证实，并且可以升级为"结构上排除"】**

Cleanroom 现在仍保留 1.12 Forge 的 `NetworkDispatcher` / `FML|HS` / `REGISTER` / `UNREGISTER` / `FMLProxyPacket` / `FMLEmbeddedChannel`。而且 Forge 在登录期间会**手动把 NetworkManager 提前切到 PLAY**，只是为了发送 FML handshake 包。逐字证据（`NetworkDispatcher.java`）：

- 服务端 `serverInitiateHandshake()` 内：`player.connection = null;` 后紧跟注释 `// manually for the manager into the PLAY state, so we can send packets later` 与 `this.manager.setConnectionState(EnumConnectionState.PLAY);`
- 客户端 `clientListenForServerHandshake()` 内：`manager.setConnectionState(EnumConnectionState.PLAY); this.netHandler = FMLCommonHandler.instance().getClientPlayHandler(); this.state = ConnectionState.AWAITING_HANDSHAKE;`
- 客户端的唯一调用点是 `FMLHandshakeClientState.START.accept()`：`cons.accept(HELLO); NetworkDispatcher dispatcher = ctx.channel().attr(NetworkDispatcher.FML_DISPATCHER).get(); dispatcher.clientListenForServerHandshake();`

时序因此是：vanilla LOGIN 照常 → **Forge 在握手第一个状态 `START` 就把 `EnumConnectionState` 写成 PLAY**，而此时 `NetworkDispatcher.state` 才刚到 `AWAITING_HANDSHAKE`。真正的进度在 `NetworkDispatcher.ConnectionState{OPENING, AWAITING_HANDSHAKE, HANDSHAKING, HANDSHAKECOMPLETE, FINALIZING, CONNECTED}` 与 `FMLHandshakeClientState{START,HELLO,WAITINGSERVERDATA,WAITINGSERVERCOMPLETE,PENDINGCOMPLETE,COMPLETE,DONE,ERROR}` / `FMLHandshakeServerState`。

**为什么是"结构上排除"而不是"有风险"**：`EnumConnectionState` 只是 vanilla 层的 channel attr（`NetworkManager.PROTOCOL_ATTRIBUTE_KEY`，仅 `HANDSHAKING/STATUS/LOGIN/PLAY` 四态）——**它的表达能力根本不足以描述 FML 握手进度**，不是"用它判断要小心"，而是"用它判断必然错"。旁证：NEBL 1.20.1 的 `ConnectionMixin` 中文注释写着「Forge20.1里,PLAY期前后,Forge自身握手残留,代理/补丁链路插入包,以及Connection.send()被调用时机不干净,很容易出现判定错误,不能按原先的移植,只能这样写了」，且它在 PLAY 之外还额外要求 `hasNegotiatedNebTransport()`。另外，`dev-1.12.2forge` 前作的 `ClientLifecycleHandler` 也在补发前显式排除 `PROTOCOL_ATTRIBUTE_KEY == PLAY`。

所以：

> **绝对不能只看 `EnumConnectionState == PLAY` 就认为可以开始 NEB。**

正确门槛是：

```text
TCP connected
    ↓
Minecraft login
    ↓
FML handshake
    ↓
FML connection established        ← 可执行锚点
    ↓
NEB protocol negotiation
    ↓
ACTIVE
```

**可执行锚点（前四步落到具体位置）**：`NetworkDispatcher.ConnectionState.CONNECTED` / `HANDSHAKECOMPLETE`（服务端 `completeServerSideConnection(...)`、客户端 `completeClientHandshake()` 置位，之后连接为 `ConnectionType.MODDED`）；接口侧可用 `FMLNetworkEvent.ClientConnectedToServerEvent` / `NetworkHandshakeEstablished` 作为挂钩。只有 `ACTIVE` 才安装/启用压缩。

这样可以完全不碰 `FML|HS` / `REGISTER` / `UNREGISTER` / mod list / registry sync / login。这会比 NEBL 1.20 那种大量黑名单安全很多。

入站的 Forge 侧细节（供实现时对齐，【证实】）：入站看到 `"FML|MP"` → 按部分序号重组；`"FML|HS"` / `REGISTER` / `UNREGISTER` → 送进 embed `handshakeChannel`（handler 顺序 `HandshakeInjector → ChannelRegistrationHandler → FMLHandshakeCodec → HandshakeMessageHandler<…>`，attr `FML_CHANNEL = "FML|HS"`）→ `FMLNetworkHandler.forwardHandshake(...)`。出站侧：`FMLOutboundHandler.write()`（`OutboundTarget.TOSERVER → targetDispatcher.sendProxy(pkt)`）→ `NetworkDispatcher.sendProxy()` → `manager.sendPacket(msg)`（channel 未开时先 `msg.copy()`）。

---

# 4. 我建议定义新的 1.12.2 wire protocol

【主张】不要尝试让 `NEB 26.2` / `NEBL 1.21` / `NEBL 1.20` / `Cleanroom 1.12` 互相 wire-compatible——没有实际价值，而且会增加大量歧义。

例如直接定义 `NEB12 protocol v1`，连接后双方交换 `protocolVersion` / `featureBits` / `maxAggregateBytes` / `zstdContextMode` / `dictionaryMode` / `dccMode`，然后：双方完全一致 → `ACTIVE`；不一致 → fail closed / 原生网络。

如果这是 Cleanroom-only 整合包基础设施，我甚至倾向于**要求客户端、服务端都安装完全匹配版本**，而不是花大量代码保持 vanilla peer compatibility。

【佐证】这个"fail closed"方向与 NEBL 现有做法一致：`NetworkRegistryMixin`（`@Mixin(HandshakeHandler.class)`）在 `handleServerModListOnClient` / `handleClientModListOnServer` 的 `@At("TAIL")` 上，先 `if (!message.getChannels().containsKey(PacketAggregationPacket.TYPE)) return;`，确认对方注册了 NEB 通道后才 `PayloadRegistry.buildSetup(...)` / `ChannelAttributes.setPayloadSetup(...)` / `NamespaceIndexManager.initForConnection(...)`。

---

# 5. Zstd 本身不是障碍

当前两个项目都使用 `com.github.luben:zstd-jni`。【证实 + 细化】

- USS master：`jarJar(implementation("com.github.luben:zstd-jni:1.+"))` —— 是**范围依赖** `1.+`（复现性风险）。
- NEBL `dev-1.20.1forge`：`implementation fg.deobf('com.github.luben:zstd-jni:1.5.7-7')`，另有一行 `jarJar('com.github.luben:zstd-jni:[1.5.7-7,1.5.8)')` —— 同样是**区间**而非精确固定（该仓库没有 `libs.versions.toml`）。
- 被删的 1.12.2 前作用的是 `1.5.5-5`。

对于 Java 25，算法/API 本身没有明显迁移障碍。真正应该验的是：Cleanroom nested dependency → native extraction → Windows x64 / Linux x64 → 重新连接后的 native/context 生命周期。可以使用 Cleanroom/Unimined 的 contained dependency 机制，而不是随便 unzip fat-jar。

每条连接应该有自己独立的 compress context / decompress context，并在 `channelInactive` / disconnect / world/server stop 时确定释放。

【待验证】本机与本次核验**没有**验证 zstd-jni 在 Cleanroom 下的 nested dependency 解析、native 解包路径、双平台可用性，也没有验证 Cleanroom 用的 zstd/native 加载方式是否与 NEB 自带 loader（NEBL 有 `AndroidZstdNativeLoader`）冲突。这一项仍是纯待办。

---

# 6. 聚合线程模型还可以比上游做得更稳

USS 当前 `AggregationManager` 的定时器是：

```java
private static final ScheduledExecutorService TIMER =
    Executors.newSingleThreadScheduledExecutor(
        new ThreadFactoryBuilder().setNameFormat("NEB-Flush-thread").setDaemon(true).build());
...
TIMER.scheduleAtFixedRate(AggregationManager::flush, 0, getFlushPeriodInMilliseconds(), TimeUnit.MILLISECONDS);
```

其中 `AggregationFlushHelper.getFlushPeriodInMilliseconds()` 返回 `20`，`flush()` 遍历 `WeakHashMap<Connection, ArrayList<AggregatedEncodePacket>> PACKET_BUFFER`。

【修正】原文档说"全局 `ScheduledExecutorService`"是对的，但若被读成"线程池"就错了：它是**单线程** ScheduledExecutorService（`NEB-Flush-thread` 一条线程），每 20 ms 扫所有 Connection。这一点对"为什么可以做得更稳"的论证其实是**加分**（单线程也意味着 flush 与 EventLoop 之间存在跨线程边界）。

在 1.12.2 Cleanroom 新实现里，我更建议：

```text
每条 Channel → 自己的 EventLoop → schedule flush
```

所有 enqueue / encode / flush / promise completion / disconnect cleanup 都发生在同一个 Netty EventLoop 上。好处是基本不需要 `synchronized WeakHashMap`、global flush thread、跨线程 `NetworkManager.sendPacket()`。

而且必须正确保留 `ChannelPromise` / `GenericFutureListener` / `flush()` / packet order：如果某次原始 write 带 promise，就等对应 aggregate 真正写成功后完成它。调用方执行 `flush()` 时应立即 `flush NEB pending batch → ctx.flush()`，而不是机械等下一次 20 ms tick。这一点对延迟和时序兼容很重要。

---

# 7. 包大小不要照搬现代版 4 MB

现代上游默认 `maxPacketSize = "4MB"`（注意它是**字符串**，运行时由 `parseByteSize` 解析，允许 2MB–64MB，越界时日志 "use default 4MB instead."）。【证实】

但 1.12.2 的网络约束不一样，Forge 自己的 `NetworkDispatcher.write()` 源码明确（逐字）：【证实】

```java
if (side == Side.CLIENT) {
    //Client to server large packets are not supported to prevent client being bad.
    ctx.write(((FMLProxyPacket) msg).toC17Packet(), promise);
} else {
    List<Packet<INetHandlerPlayClient>> parts = ((FMLProxyPacket)msg).toS3FPackets();
    ...
}
```

即：server → client 的大型 `FMLProxyPacket` 有 multipart 机制；client → server 的大型 FML packet **明确不支持**。分片常量：`FMLProxyPacket.PART_SIZE = 0x100000 - 0x50`（1048496）、`MAX_LENGTH = PART_SIZE * 255`（267366480），超过 255 片抛 `IllegalArgumentException("Payload may not be larger than " + MAX_LENGTH + " bytes")`；单包与分片的分界是 `data.length < PART_SIZE`；分片走明文通道 `"FML|MP"`，preamble 依次写 channel(string) / parts(byte) / data.length(int)，每片首字节是序号。

**【重要新增，原文档漏掉】还有一个与 Forge 无关、比 4MB 更致命的硬上限**：`NettyVarint21FrameDecoder` 最多读 3 字节长度前缀，末尾 `throw new CorruptedFrameException("length wider than 21-bit");`；`NettyVarint21FrameEncoder` 则 `int j = PacketBuffer.getVarIntSize(i); if (j > 3) { throw new IllegalArgumentException("unable to fit " + i + " into " + 3); }`。⇒ **帧上限 = 2^21−1 = 2097151 字节（≈2MB），与 NEB 无关、与 Forge 无关，由 vanilla 帧编解码器决定。**所以上游默认的 4MB 在 1.12.2 上"不改 frame codec 就根本发不出去"——这不是保守建议，而是硬约束。

因此第一版不应该顺便去改 `NettyVarint21FrameDecoder` / `NettyVarint21FrameEncoder`。我会直接让 NEB aggregate 保持在 1.12.2 原生安全范围内，并**按方向分批**。如果一个原始 packet 自己已经大于 NEB batch 上限：`flush pending → 原包直通`。这样首先把基础协议做正确。现代 NEB 那套"放宽 frame size"可以后续单独做。

---

# 8. DCC 可以做，但这是第二个真正困难的部分

Modern NEB 的 Delayed Chunk Cache 直接修改服务器 chunk tracking view。1.12.2 对应结构是 `PlayerChunkMap` / `PlayerChunkMapEntry` / `SPacketChunkData` / `SPacketUnloadChunk`。

Forge 源码里 `PlayerChunkMapEntry.removePlayer()` 会发送 `new SPacketUnloadChunk(x, z)`。【证实 + 版本保留 + 前提修正】

```java
public void removePlayer(EntityPlayerMP player) {
    if (this.players.contains(player)) {
        if (this.sentToPlayers) {
            player.connection.sendPacket(new SPacketUnloadChunk(this.pos.x, this.pos.z));
        }
        this.players.remove(player);
        if (this.players.isEmpty()) {
            this.playerChunkMap.removeEntry(this);
        }
    }
}
```

两点必须写清楚，否则后面会踩空：

1. **【版本保留】** 上面的方法体取自 **MC 1.12（MCP 9.40）反编译源，不是 1.12.2 的逐字原文**。原因：MCP 9.42 没有公开镜像（`mcp942` 类仓库搜索 total_count = 0），Forge 1.12.x 的 `.patch` 原文取不到（raw / blob API / jsdelivr 各有限制）。1.12.2 侧只有 **Forge javadoc 签名**可作凭据：`net.minecraft.server.management.PlayerChunkMapEntry.removePlayer(EntityPlayerMP)`、`getWatchingPlayers()`、`net.minecraft.network.play.server.SPacketUnloadChunk.getX()/getZ()`。旁证显示该类确有版本间改动：Forge 1.12.x 对 `PlayerChunkMapEntry.java` 有 **7749 字节** patch，且 1.12.2 javadoc 比 1.12 源码多出 `getWatchingPlayers()`。→ 方向性结论稳固（`removePlayer` 确实发卸载包），但**不要把它当逐字原文引用**，落地前请对 1.12.2 实际字节码/反编译复核。
2. **【前提修正】** 原文档"服务端仍会重新发送完整 `SPacketChunkData`"这一句需要一个前提才成立：发 `SPacketChunkData` 是由 `sentToPlayers` 标志门控的，而 `removePlayer` **只把玩家从 `players` 列表摘掉，并不会把 `sentToPlayers` 置回 false**；此后该玩家不再收到该 chunk 的更新，条目也只在列表空掉时才被移除。所以"服务器仍会重发全量"指的是**玩家回来后重新进入视距、条目被重新 `sendToPlayers()`** 的那条路径——结论不变（客户端单方面忽略卸载包省不到带宽），但机制描述要准确。

因此实现真正有省流量效果的 DCC，不能仅仅让客户端忽略 `SPacketUnloadChunk`，因为那样下一次进入范围，服务器仍然会重新发送完整 `SPacketChunkData`，网络并没有真正省下来。

### 更接近上游 NEB 的实现

玩家离开正常视距时：不要立即 `PlayerChunkMapEntry.removePlayer(player)`，而是建 `player DCC entry { chunk, expire tick, LRU }`，于是服务端仍然认为该玩家 watch 这个 chunk。玩家很快回来 → cache hit → 本来就在 watcher 中 → 不重新 `SPacketChunkData`；超时/太远/容量满 → 真正 `removePlayer()` → `SPacketUnloadChunk`。这个方案可行，而且与上游 DCC 原理一致。

但必须验证：block update / TileEntity update / entity tracking / dimension change / teleport / logout / respawn / view distance change / chunk unload / server memory / 每玩家额外 watched chunk 数量。因此我会把它放在 aggregation + Zstd 后面。

---

# 9. NEBL 21.1.12 的"字典引用"和"区块引用"不要第一轮一起搬

这是两个仓库差异最大的地方之一。NEBL 21.1.12 已增加 `Dictionary Reference Deduplication` / `Chunk Reference Cache` / `Streaming Zstd`。这些不是 USS 当前 master 的核心功能。

【修正】按上一节的核验，这三个功能是 **21.1.11** 引入、21.1.12 只调参数；且 `streaming` **默认 false**。分期结论不变：

| 功能 | 可行性 | 建议 |
|---|---|---|
| packet aggregation | **很高** | 第一阶段 |
| Zstd compression | **很高** | 第一阶段 |
| traffic statistics | **很高** | 第一阶段 |
| FML/Forge custom packet transparency | **高** | 第一阶段必须解决 |
| DCC | **高，但复杂** | 第二阶段 |
| compact custom header | **可行但收益低** | 延后 |
| packet dictionary | **可行，中高风险** | 第三阶段 |
| chunk reference | **可行，高风险** | DCC 稳定后 |
| streaming Zstd | **可行，高兼容风险** | 默认关闭 |
| Replay / proxy compatibility | **可行但必须实测** | 发布门槛 |

尤其不要把 DCC + chunk-reference + packet dictionary 混成一次实现。否则一旦出现"玩家回来后区块内容旧、TE 不更新、数据包重复、缓存 miss 不一致"，会很难判断是哪一层出了问题。

---

# 10. ReplayMod 和代理是必须单独验收的

NEB 最危险的地方不是压缩算法本身，而是它改变了**线上数据包的组合形状**。Replay 类模组可能记录 NEB 解压前，也可能记录 NEB 解压后；如果是前者，跨 batch Zstd context 就会让单独回放变得困难。因此第一版我甚至建议：**默认使用独立 batch frame，不启用跨 batch streaming/context dependency**，等 ReplayMod 1.12.2 路径完全确认后再开启复用。

【佐证 — 并且这个风险已被上游自己承认】上游配置里确实有 `playersDoNotUseContext`（面向 Replay 这类必须看到原始包流的 mod）、`compatibleMode`、`blackList`（默认含一系列 velocity 相关包），README 明确写"黑名单中的包会打断正在进行的聚合"；`dccSizeLimit` / `dccDistance` / `dccTimeout` 仅服务端；`contextLevel` 21~25（默认 23 = 8MB）。**再叠加 §1 附的那个反转 bug**：本应"不复用上下文"的玩家目前反而会复用——对 Replay 兼容性而言这是**双重不利**，也说明这条路径上游自己测得不透。

对于 BungeeCord / Velocity 一样：普通透明 TCP relay 大概率没问题；会检查或修改 Minecraft packet / plugin message 的 proxy 可能被 NEB aggregate 隐藏内部数据。所有代理自身需要读取的 custom channel 都应该直通。

---

# 11. `6bb602e` 对这个项目反而影响很小

刚才分析 Actinium 时，`6bb602e` 是巨大断层，因为 GLFW → SDL3。但 NEB 几乎不碰 GL/window。所以 Cleanroom 6bb+ / SDL3 / LWJGLY / Kirino 对 NEB transport core 都没有架构影响。【证实：提交内容确为输入/图形/构建层重写，SDK 无关】

统计屏幕快捷键也应该通过 Forge/Cleanroom `KeyBinding`/`Event` 而不是自己访问 SDL。

【修正】但"影响很小"要加限定：该提交同时重写构建脚本（CleanroomGradle）、`removed buildSrc/`、改变 `run*` task 继承 mixin debug args 的默认行为、升级到 JUnit 6。所以对**开发/测试流程**是有实际影响的；只是对 NEB 运行时架构影响小。

真正需要固定的是 `Cleanroom network patches` / `FML NetworkDispatcher` / `Netty version` / `Mixin target bytecode`，而不是图形栈。

---

## 最适合的代码来源不是二选一

如果让我为 Q-Engineering 做这个回移植，我不会"选一个仓库 fork 后一路删报错"。会把来源定义成：

```text
USS-Shenzhou/NotEnoughBandwidth master
        │
        ├─ 算法/行为的上游事实来源
        ├─ 最新核心 header 设计
        ├─ Zstd aggregation
        ├─ DCC 思路
        └─ 最新兼容修复（注意：含 §1 附的 useContext 反转 bug，不要照抄）
                 │
                 ▼
        新 Cleanroom 1.12.2 implementation
                 ▲
                 │
xianaldai/NEBreload dev-1.20.1forge
        │
        ├─ Forge port 的失败经验和兼容处理
        ├─ custom payload/FML 差异
        ├─ proxy / voicechat 等兼容经验
        └─ per-connection negotiation 思路

xianaldai/NEBreload dev-1.21.1neoforge
        │
        └─ dictionary/chunk-reference/streaming
           作为后续可选功能来源

xianaldai/NEBreload dev-1.12.2forge 的历史提交（4a76efb … b481d9e）   ← 本版新增
        │
        ├─ 1.12.2 上真实写过的 transport/indextype 实现（NamespaceIndexManager 等）
        ├─ ClientLifecycleHandler：无协商补发钩子时的轮询补发方案与其失败模式
        └─ 2026-05-24 放弃的原因（需进一步查 issue/Discord，仓库内无说明）
```

也就是说：

> **以 USS 最新版作为"NEB 应该做什么"的规范，以 NEBL Forge 分支作为"老 Forge 平台哪里会出坑"的参考，而底层 1.12.2 transport 重新实现。**

这是比直接回移植任一分支可靠得多的路线。【主张】

最后还有许可问题：两边都是 GPL-3.0 系列（上游 license API `spdx_id: GPL-3.0`，LICENSE 首行 `GNU GENERAL PUBLIC LICENSE / Version 3, 29 June 2007`；NEBL license key = `gpl-3.0`），上游 README 明确写「如果您有意愿帮助我们将NEB进行反向移植，请在提交任何PR之前到 Discord 与 USS_Shenzhou 讨论。」（仓库根目录**没有** CONTRIBUTING 文件，该要求只在 README）。如果做独立 Cleanroom 版本，代码分发需要继续满足 GPL、保留原作者版权和来源；如果采用 MapleSugar365 的 NEBL 专有新增实现，也应保留那部分具体 provenance，而不能只标 USS。【证实 + 细化】

【细化证据】NEBL README「版权和许可」目前只写 `Copyright (C) 2025 USS_Shenzhou`，未出现 MapleSugar365；但 `dev-1.21.1neoforge/gradle.properties` 是 `mod_authors=MapleSugar365, USS_Shenzhou` 且 1.21.1 线提交均由其签署；`dev-1.20.1forge` 仍是 `mod_authors=USS_Shenzhou`（最新提交人 xianaldai）。另外 NEBL README 自述"这是 NEB 的反向移植分支…"，并请求不要把 issue 提给上游。**也就是说：若复用 21.1 线的 dictionary/chunk-reference/streaming，provenance 归属 MapleSugar365；许可清单要按分支分别标注，不能一律写 USS。**

**总体上，我认为这个项目很适合 Cleanroom-only：核心版没有不可解的技术阻断（唯一的硬约束——`2^21−1` 帧上限——只需把 batch 上限设为 ≈2MB 以下即可绕开），甚至 1.12.2 的 packet-ID 系统能让一部分现代 NEB 代码变得更简单。真正应该谨慎的是 FML 网络管线位置、DCC 和后续字典/区块引用，而不是 Zstd 本身。**

---

## 附录 A：本次核验仍未确定的点

1. **zstd-jni 在 Cleanroom 下**的 nested dependency 解析、native 解包路径、Windows/Linux x64 可用性、重连后的 context 生命周期——全部未验。
2. **`FMLOutboundHandler` 在 SimpleNetworkWrapper 每个 channel 的 embedded pipeline 中的确切插入点**——未逐字确认，只确认了它在 `OutboundTarget.TOSERVER` 时会走 `targetDispatcher.sendProxy(pkt)`。
3. **1.12.2 逐字 vanilla 源码**：无法取得（MCP 9.42 无公开镜像、Forge 1.12.x `.patch` 取不到），§8 的 `PlayerChunkMapEntry` 方法体是 MC 1.12（MCP 9.40）版本。
4. **`dev-1.12.2forge` 为什么被废弃**：仓库内无说明（只有 `no 1.12.2!` 与 4703 行删除），需要 Discord/issue 侧信息。
5. `playersDoNotUseContext` 反转 bug 是否有可观测症状——受提前 return 与缓存保护，**推断为难以触达**，未做运行时验证。
6. 上游 master 是否有 1.12.2 相关痕迹——属"未发现"，不是"证明不存在"（GitHub 代码搜索需登录，返回 0 结果不可作证据）。

## 附录 B：证据索引（可复查）

- 上游仓库：<https://github.com/USS-Shenzhou/NotEnoughBandwidth>（master HEAD `e24108be868f143e9693a0653877cb602c6f3309`，2026-09-21T06:55:34Z `port to 26.2`；4 分支 `3dgm`/`master`/`multi-version-refactor`/`velocity-compat`；tags、releases 均空）
  - `src/main/java/cn/ussshenzhou/notenoughbandwidth/zstd/ZstdHelper.java`（84 行，全仓库无 `playersDoNotUseContext` 写入点）
  - `.../AggregationFlushHelper.java`（`return 20;`）、`.../AggregationManager.java`（`newSingleThreadScheduledExecutor` + `WeakHashMap<Connection, ArrayList<AggregatedEncodePacket>> PACKET_BUFFER`）
  - `.../mixin/ConnectionMixin.java`（`@Mixin(value = Connection.class, priority = 1)`，注入 `send(...ChannelFutureListener;Z)V` 的 HEAD）
  - `.../NotEnoughBandwidthConfig.java`（`public String maxPacketSize = "4MB";`、`compatibleMode`、`blackList`、`contextLevel` 默认 23、`dcc*`）
  - 提交 `36f6fe5`（`potential replay compatibility problem`，ZstdHelper 第 58–83 行新增）
- NEBL 仓库：<https://github.com/xianaldai/NEBreload>（6 分支；33 tags 无 1.12.2/cleanroom；建于 2026-03-30）
  - `dev-1.21.1neoforge`：HEAD `21d8d48a6f1bb6b2e16a5ba9709ef1ce552dea2e`，MapleSugar365，2026-08-19T15:41:56Z
  - `dev-1.20.1forge`：HEAD `f74900d32ace40bd43e7cee9cab54cde6bdb3740`，message `reborn`，2026-06-11T05:26:52Z（default branch、protected）；`mixin/ConnectionMixin.java`、`mixin/NetworkRegistryMixin.java`（`@Mixin(HandshakeHandler.class)`）
  - `dev-1.12.2forge`：`7b5d29813069cf4c6f7565a71fda219a18e7b734`（tree `4b825dc642cb6eb9a060e54bf8d69288fbee4904`，0/4703，2026-05-24T13:17:50Z，`no 1.12.2!`）；前作链 `4a76efb`→`b481d9e`，`client/ClientLifecycleHandler.java`（`MAX_SETUP_RETRIES = 12`、`RETRY_INTERVAL_TICKS = 10`）、`indextype/NamespaceIndexManager.java` 等
- Cleanroom 仓库：<https://github.com/CleanroomMC/Cleanroom>（default branch **main**，非 master）
  - `module/forge/src/main/java/net/minecraftforge/fml/common/network/internal/FMLProxyPacket.java`（218 行；`readPacketData`/`writePacketData` = `// NOOP - we are not built this way`；`PART_SIZE = 0x100000 - 0x50`、`MAX_LENGTH = PART_SIZE * 255`）
  - `.../fml/common/network/NetworkDispatcher.java`（`insertIntoChannel()` 的 `addBefore("packet_handler", "fml:packet_handler", this)`；`serverInitiateHandshake()` / `clientListenForServerHandshake()` 中手动 `setConnectionState(EnumConnectionState.PLAY)`；`write()` 的 `//Client to server large packets are not supported…`；`ConnectionState` 枚举；`PacketLoggingHandler` 的 `replace("splitter"/"prepender", …)`）
  - `module/minecraft` 的 `NettyVarint21FrameDecoder` / `NettyVarint21FrameEncoder` 补丁（21-bit 长度前缀，`length wider than 21-bit` / `unable to fit … into 3`）
  - 提交 `6bb602ec6f0c8c92545818655b1cdd80aba9a31f`（Rongmario，2026-09-23T19:15:44Z，`+39906/−34158`，`pack: project overhaul (CleanroomGradle, LWJGLY, SDL 3)`）
  - `module/forge/src/main/resources/forge_at.cfg`；README 的 Java 25+ / LWJGL 3+SDL 3 / CleanMix / Netty 4.2 说明
- 1.12.2 javadoc（ForgeJavaDocs-NG，1.12.2）：`net.minecraft.server.management.PlayerChunkMapEntry`（`removePlayer(EntityPlayerMP)`、`getWatchingPlayers()`）、`net.minecraft.network.play.server.SPacketUnloadChunk`（`getX()`/`getZ()`）
- MC 1.12（MCP 9.40）反编译源（§8 方法体的实际出处）：`WangTingZheng/mcp940` `src/minecraft/net/minecraft/server/management/PlayerChunkMapEntry.java`

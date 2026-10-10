---
name: wechat4-chat-export
description: 从本机微信 4.x（Windows）本地库导出指定联系人的聊天记录为 Markdown，并可解密聊天图片。当用户说「导出我和XX的微信聊天记录」「把微信聊天记录保存成 md」「微信聊天记录导出」「看看我和XX聊了什么并导出」「通过 WiFi 连手机导微信记录」时使用。已内置 SQLCipher 4 库解密 + V2 图片格式解密 + WAL 环形缓冲合并的完整实现与踩坑记录，覆盖密钥由内存扫描恢复的全流程，以及手机端（Android 无 root）不可直取的实测边界与官方迁移路径。另有 export_text.py 纯文本导出器（自动剔除图片/表情/视频/语音）。
agent_created: true
---

# 微信 4.x 聊天记录导出（Windows）

在**不安装任何第三方逆向工具**（PyWxDump 已被 DMCA 下架、chatlog 无 release、GitHub 常被墙）的前提下，
纯 Python + `cryptography` 完成从本机微信 4.x 本地库到 Markdown 的导出。

## 0. 前置事实（先确认，别猜）

| 项 | 位置/值 |
|---|---|
| 数据根 | `C:\Users\<user>\Documents\xwechat_files\<wxid>_<hash>\` |
| 库目录 | `db_storage\{contact,message,session,sns,favorite,general,...}\*.db` |
| 图片 | `msg\attach\<md5(对方wxid)>\<yyyy-mm>\Img\<md5>.dat`（另有 `_t` 缩略图 / `_h` 高清） |
| 文件 | `msg\file\<yyyy-mm>\<原文件名>`（**明文**，直接可用） |
| 视频 | `msg\video\<yyyy-mm>\<md5>.mp4` + `.jpg`（**明文**，直接可用） |
| 微信主程序 | `C:\Program Files\Tencent\Weixin\<版本>\Weixin.dll`、`VoipEngine.dll` |

多进程：主进程叫 `Weixin.exe`，还有一批 `WeChatAppEx.exe`（Chromium 式渲染进程），**扫内存必须两者都扫**。

## 1. 库解密（SQLCipher 4）

**参数（已实测于 4.1.15.13）**
- KDF：`PBKDF2-HMAC-SHA512(passphrase, salt, iterations=256000, dklen=32)`
- 页布局：`page_size=4096`，`reserve = ((16+64+15)//16)*16 = 80`
- 页 0：`salt(16) | AES-256-CBC密文[16:4016] | IV[4016:4032] | HMAC-SHA512[4032:4096]`
- 页 N：`AES密文[0:4016] | IV[4016:4032] | HMAC[4032:4096]`

**关键坑**：SQLCipher 把页 0 的**头 16 字节替换成了 salt**，所以解密出的明文**不是** `SQLite format 3\0`，
而是从 SQLite 头第 16 字节开始 —— 即第一个字节是 page size 高字节 `0x10`（4096）。
见到 `10 00 02 02 50 40 20 20 ...` 就是**成功**了，别误判为失败。
还原时把 `SQLite format 3\x00` 补在页 0 解密体前面，每页尾部补 `reserve` 个 `\x00`，即可被 `sqlite3` 正常读取。

**HMAC 校验（确认密钥用）**
```
mac_salt = salt XOR 0x3a
new_key  = PBKDF2(passphrase, salt, 256000, 32, sha512)
mac_key  = PBKDF2(new_key, mac_salt, 2, 32, sha512)
mac      = HMAC-SHA512(mac_key, buf[16:4032] + struct.pack('<I',1))  == buf[4032:4096]
```

**passphrase 从哪来**：账号级密钥不落盘，在 `Weixin.exe` 内存里以 **XOR 混淆态**存在。
- XOR key 从 `Weixin.dll` 代码段静态提取，特征码：`48 BA <8B×8> ×4` 后跟 `48 85 C0`（`mov rdx,imm64` ×4 + `test rax,rax`）
- 扫内存取候选 → 与 XOR key 逐字节异或 → PBKDF2/HMAC 校验 → 命中即 passphrase

## 2. WAL 合并（重要，且反直觉）

- WAL 头 32 字节**明文**（魔数 `37 7f 06 82`），帧 = `24B帧头 + 4096B页数据`，页数据与主库同布局。
- **帧头里的 salt 与 WAL 头的 salt 不一致**（微信改过），**不能拿 salt 做有效性校验**，否则一帧都套不上。
- WAL 是**环形缓冲**：文件开头那段是**最新一代**，往后是已被 checkpoint 落库的旧代残留。
- **只套用开头那段最新代**。重复套用旧代会把页回滚成旧内容 → `integrity_check` 报 `btreeInitPage() returns error code 11`。
- 若最新代的帧 `dbSize` 全为 0（无 commit 帧），说明事务未提交 → **SQLite 自己也会忽略**，此时主库即最新状态。
- 判断最新是否够新：`select max(create_time) from Msg_xxx`，与当前时间比对。

## 3. 消息表定位

- 消息按会话分表：表名 = `Msg_` + `md5(对方username)`，主库 `db_storage\message\message_0.db`
- `Name2Id(rowid, user_name, is_session)`：`Msg.*.real_sender_id` 指向 **Name2Id 的 rowid**
- `contact.db` 的 `contact(username, remark, nick_name, alias, ...)` 用来按昵称/备注搜人
- 字段：`local_id, local_type, real_sender_id, create_time, message_content, packed_info_data, source`
- **`local_type` 要 `& 0xFFFFFFFF`**：高位是标志位，例如 `25769803825 & 0xFFFFFFFF = 49`（应用消息）
- 常用类型：`1`文本 `3`图片 `34`语音 `43`视频 `47`表情 `49`应用消息(file/url/小程序) `10000`系统

**内容解压**：`message_content` / `source` 是 **zstd** 压缩的 BLOB（魔数 `28 b5 2f fd`，`WCDB_CT_* = 4`）。
用 `zstandard` 解压后再当 UTF-8 / XML 读。`packed_info_data` 是 protobuf，**别硬解**：
用正则从中抽 32 位十六进制串即可拿到图片/视频的 md5（正好等于 `.dat` / `.mp4` 的文件名）。
`<img_file_name>` 在 `source`（msgsource）里，不在 `message_content` 里。

## 4. 图片解密（V2 格式）

```
[0:6]   07 08 56 32 08 07     签名
[6:10]  aes_size   (LE u32)
[10:14] xor_size   (LE u32)
[14]    占位
[15:]   AES-128-ECB 密文，长度 = aes_size//16*16 + 16，PKCS7 去填充
        然后 明文段
        末尾 xor_size 字节 = 单字节异或段
```
- **xor_key 可离线反推**：缩略图 `_t` 是普通 JPEG/PNG。JPEG 尾 `FF D9` → `k = tail[-2]^0xFF`，验证 `tail[-1]^0xD9 == k`；PNG 尾 `60 82` 同理（`tail[-1]^0x82`）。多文件投票取众数，一次就能定。
- **aes_key 不落盘**，只在微信渲染图片时驻留内存 → 必须先让用户在微信里**点开 2~3 张图片放大**，
  然后立刻扫内存：正则 `(?<![0-9A-Za-z])[0-9A-Za-z]{16}(?![0-9A-Za-z])` 取候选，用密文首块做 AES-ECB 解密，
  看明文是否命中 `FF D8 FF` / `89PNG` / `GIF8` / `RIFF` / `wxgf`。
  写成**轮询脚本**（每 2~3 秒扫一轮，共 90 秒）比单次扫描命中率高得多。
- 解出的**原图**可能是 `wxgf`（微信自研容器，非标准图片），需调 `VoipEngine.dll` 导出函数
  `wxam_dec_wxam2pic_5(in, inlen, out, &outlen, &cfg)`，cfg.mode：`0`=jpeg `1/2`=png `3`=gif。
  拿不到就把 `_t` 缩略图当预览图用。

## 5. 工程坑（都是踩过的）

- **源文件是只读的**：微信目录里文件权限为 `-r--r--r--`，`shutil.copy2` 会把只读属性带到目标，
  第二次运行覆盖时 `PermissionError`。→ 用 `shutil.copyfile`（不继承属性），覆盖前先 `chmod`。
- **Windows 下写含中文路径的 .bat / .ps1**：必须以 **GBK（cp936）** 编码落盘，
  否则 cmd 下路径乱码。不要加 `chcp 65001`（反而更乱）。用 `%~dp0` 规避绝对路径。
- **`.bat` 里 `&` `|` `<` `>` 要转义**：`echo` 行里的括号要写 `^(` `^)`。
- **pip 装包**：本环境 pip 走沙箱代理常握手失败 → 用 `curl` 直下 wheel 再
  `pip install <本地.whl> --no-index --no-deps`。`cryptography` 装在
  `C:\Users\book\.workbuddy\binaries\python\envs\default\Scripts\python.exe` 这个 venv 里，
  用别的 python 会 `ModuleNotFoundError`。
- **Bash 里不能调 PowerShell / cmd.exe**（安全策略拦截）；`explorer.exe "路径"` 直接调返回 exit=1 属正常。
- 导出成品要按消息顺序编号（`01_`、`02_`…）＋ 时间戳命名，并附一个 `说明.txt`。

## 6. 交付形态（推荐）

```
D:\微信聊天记录导出\
├── <联系人>聊天记录_YYYYMMDD.md     正文（含附件/视频相对链接）
├── 附件\                            明文文件（pptx/docx/pdf/zip，按序号命名）
├── 视频\                            mp4 + 首帧图
├── 图片dat\                         原始加密 .dat（按序号命名，便于后补解密）
├── 图片\                            解密后的图片（抓到 aes_key 后生成）
├── 解密图片.bat                     一键：提示用户点图 → 轮询抓 key → 解密 → 重刷 MD
└── _工具\wxzeus.py                  自包含工具：export / unlock 两个子命令
```

`wxzeus.py export` 生成 Markdown 时**顺带把附件、视频拷贝过来**；
`unlock` 抓到密钥后解密图片并**重新生成** Markdown，此时图片以 `![](...)` 内嵌，
并把表头的「图片解密」一行从「未解密」改为「已解密 N 张」。

### 6.1 纯文本导出器 `export_text.py`（只要文字时用这个）

用户说「**只要文本，不要图片和表情包**」时，别用 `wxzeus.py`，用 `_工具\export_text.py`：

- 只保留 `local_type in (1, 49, 10000)`（文本 / 应用消息的文字标题 / 系统提示），
  自动剔除图片 3、语音 34、视频 43、表情 47、名片 42、通话 50
- 每条按 `**HH:MM:SS · 发送人**` 起头，按天分节，超 2000 字截断
- 同时打印基础分析：条数、双方分布、字数、活跃天数、高峰日、时段分布
- 改内置 `TARGETS = {昵称: wxid}` 或命令行 `python export_text.py <wxid> <昵称>` 即可导出别的联系人
- 表不存在时直接返回「电脑端本地库没有该会话的消息表」，这是**正常结论**不是 bug

## 7. 手机微信（Android）的记录能不能取？——**无 root 取不到，只能走官方迁移**

用户说「通过 WiFi 连手机，把手机微信的聊天记录导出来」时，**先看这一节**，别一头钻进 ADB。

### 7.1 实测边界（REDMI K80 Pro / HyperOS / Android 16 / 微信 8.0.78）

| 路径 | 结果 |
|---|---|
| 文本库位置 | `/data/data/com.tencent.mm/MicroMsg/<32位hash>/`（应用私有目录） |
| `adb shell ls /data/data/com.tencent.mm/MicroMsg/` | ❌ `Permission denied` |
| `adb -s <dev> root` | ❌ `adbd cannot run as root in production builds` |
| `adb shell run-as com.tencent.mm id` | ❌ `package not debuggable` |
| `adb backup` | ❌ 已废弃；且微信 `targetSdk 34`，Android 12+ 不再备份 app 数据 |
| `/storage/emulated/0/Android/data/com.tencent.mm/MicroMsg/` | ✅ **可读**（shell 属 `ext_data_rw` 组），但实测 918 MB **全是图片 / 小程序 / 状态文件，一条文字都没有** |

> 结论：**Wi-Fi + ADB 能连上手机、能读外部存储，但读不到任何聊天文本。** 与第 8 节「私密相册无 root 导不出明文」是同一类边界，
> 不要浪费时间扫描内存或找「隐藏的 db」—— 外部目录里 `find -iname "*.db"` 只有小程序的 `cache.db`。

### 7.2 唯一可行路径：手机微信官方「导出到电脑」→ 再从电脑库读

1. 电脑微信**登录同一账号**并保持在线（先 `contact.db` 里核对账号 wxid 是否为同一人）
2. 手机微信（与电脑连同一 Wi-Fi）：
   - 新版：**我 → 设置 → 通用 → 聊天记录管理 → 导入与导出 → 导出到电脑**
   - 旧版：**我 → 设置 → 聊天 → 聊天记录迁移与备份 → 迁移 → 迁移到电脑**
3. 手机上勾选目标会话 → 等迁移完成
4. 记录会**合并进电脑微信本地库** → 直接用本文档第 1–3 节流程解密读取

⚠️ **两个功能别搞混**（这是用户最容易做错的一步）：

| 功能 | 入口 | 产物 | 我们能否读 |
|---|---|---|---|
| 备份聊天记录到电脑 | **电脑**微信左下角「更多 → 备份与恢复」 | 加密备份包，只能恢复回手机 | ❌ 读不了 |
| 导出/迁移到电脑 | **手机**微信「聊天记录管理 → 导入与导出」 | 合并进电脑微信本地库 | ✅ 能读 |

### 7.3 备选（不想迁移、或只有少量几条）

让用户在手机上把目标聊天**逐条转发到「文件传输助手」** → 电脑微信里立刻能看到 → 再按第 1–3 节导出。
条数多时很笨，但零门槛、纯官方操作。

### 7.4 顺带可确认的信息

- `adb shell dumpsys package com.tencent.mm | grep versionName` → 手机微信版本
- 手机端 `MicroMsg/<hash>/` 的目录名**不是** `md5(wxid)`（两端算法不同），别拿它反推账号
- 但**会话级**标识两端一致：`md5(对方wxid)`（电脑端 `Msg_<md5>` 表名 / 手机端 `attach/<md5>` 目录）

## 8. 朋友圈(SNS)分析（已验证 · 4.1.15.13 · 本机该账号）

> 用户常要分析某好友朋友圈。下面是可复用的提取/分析流程（已在 `wxid_1zhumjlen7pj22` 账号实测）。

- **库位置**：`db_storage/sns/sns.db`（SQLCipher 4，**与 message 库共用同一账号 PASSPHRASE**，解密参数同第 1 节：PBKDF2-HMAC-SHA512 256000、page0 头 16B 为 salt、IV 在 `[4016:4032]`）。
- **本账号密钥已内联**：`wxzeus.py` 顶部 `PASSPHRASE = bytes.fromhex('61d047d3...b4dc91a0')`。**同账号重跑直接拿它解密，别再扫内存**——又快又稳。
- **关键表** `SnsTimeLine(tid INTEGER PK, user_name TEXT, content TEXT, pack_info_buf TEXT)`：每条朋友圈一行。`user_name`=发布者 wxid；`content`=完整 `TimelineObject` XML（含 `<createTime>`、`<contentDesc>` 正文、`<ContentObject><type>`、`<mediaList>` 图片/视频、`<finderFeed>` 视频号、`<contentUrl>` 转发文章）。
- **定位某人**：先解密 `db_storage/contact/contact.db`，按 `nick_name`/`remark` 搜 wxid，再 `SELECT * FROM SnsTimeLine WHERE user_name=?`。
- **ContentObject type 实测映射**：`1`=图文，`2`=纯文字，`0`=转发文章（带 contentUrl/同站链接），`28`=视频号（finderFeed），`7`/其他=图片或视频。分类规则：含 `<finderFeed>`→视频号；media 含视频或视频号→视频；`<contentUrl>` 或 `mp.weixin.qq.com`→转发文章；mediaList 有图+正文→图文；有图无文→纯图片；仅正文→纯文字。
- **覆盖量**：SnsTimeLine 是该好友相册的本地缓存（用户滑过其相册即入库）。实测某好友 1714 条 / 2023-01~2026-10 全在库内，无需再滚动加载。若某好友条数极少，提示用户去其主页相册滑到底再重提。
- **⚠️ XOR 掩码坑（重要）**：第 1 节文档所述「`48 BA`×4 + `48 85 C0`」掩码特征码，在此版本 Weixin.dll 里命中的是**占位常量**（`55 55 33 33 0f 0f 01 01…`），并非真实密钥掩码；纯 printable 暴力扫内存也极慢且对二进制密钥无效。**结论：同账号优先用内联 PASSPHRASE 解密，不要依赖内存扫密钥。**
- **中文分词**：无 jieba 时，用「去停用字单元 + 二元组」统计即可（见 `analyze_render.py` 思路）；表情抓 `[一-龥]{1,6}` 方括号码，话题抓 `#…#` 配对。

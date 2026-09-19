# 抖音七神签名体系

> 仅作安全研究与逆向工程框架梳理，不提供商业化调用工具

## 总览

抖音请求签名由 `libsscronet.so` 与 `libmetasec_ml.so` 协同生成。`libmetasec_ml.so` 在初始化阶段将签名函数指针注册到 `libsscronet.so` 的全局变量；请求发生时，网络层通过函数指针调用加密层

| 签名            | 定位       | 主要 SO              | 算法原语 / 输出                                                |
| ------------- | -------- | ------------------ | -------------------------------------------------------- |
| **X-Khronos** | 时间基准     | 请求头                | 秒级 Unix 时间戳                                              |
| **X-Gorgon**  | 请求完整性签名  | `libsscronet.so`   | MD5 + 魔改 RC4 + 半字节/比特后处理；52 字符 hex                       |
| **X-Helios**  | 设备指纹采集链  | `libsscronet.so`   | AES-OFB；Base64                                           |
| **X-Ladon**   | 接口级签权    | `libsscronet.so`   | AES-CBC / 魔改 AES；Base64 短密文                              |
| **X-Argus**   | 设备指纹信封   | `libmetasec_ml.so` | Protobuf → SM3 → SIMON-128/256 → AES-128-CBC；数百字节 Base64 |
| **X-Medusa**  | 视频流专项校验  | `libmetasec_ml.so` | Protobuf → SM3 + 魔改 AES + 4 轮 Feistel；768 字节级封包          |
| **X-Perseus** | 用户行为轨迹签名 | `libmetasec_ml.so  | 分段缓冲区组装 + 自定义变换                                          |
部分极速版或低版本客户端缺少 `X-Perseus`，为“六神”


## 签名简述

**X-Khronos**  
秒级 Unix 时间戳。同一请求中应与 query 的 `ts`、`X-SS-Req-Ticket` 时间部分保持 1～2 秒内偏差，作为其他签名的时间锚点

**X-Gorgon**  
主完整性签名。输入含 query、`MD5(body)[:4]`、时间戳。核心是魔改 RC4：KSA/PRGA 仅做单向赋值，非标准双向交换。输出经半字节拆分与比特重排为 52 字符 hex。存在 v4 / v5 差异

**X-Helios**  
AES-OFB。明文形式约为 `{khronos}-{device_id}-{aid}`；密钥与 IV 由 MD5 十六进制字符串的 ASCII 字节派生

**X-Ladon**  
输入为 `khronos`、`lc_id`、`aid`。加密链路为 AES-CBC 或魔改 AES，输出 Base64 短密文，用于接口级时间窗签权

**X-Argus**  
明文为 Protobuf，含 `device_id`、`install_id`、`app_id`、版本、Khronos、query/body 的 SM3 摘要、`license_id` 等。链路：Protobuf → SM3 → SIMON-128/256（72 轮）→ AES-128-CBC → Base64。所在 SO 受 OLLVM + VM + JIT 保护

**X-Medusa**  
视频流接口专项签名。核心为魔改 AES，外层 SM3 与 4 轮 Feistel，输出 Base64 768 字节级封包。Protobuf 明文中可见 `getpid()`、`gettimeofday()` 等进程/时间字段。

**X-Perseus**  
输入由头部、中间字段、主体三段拼接，经自定义变换输出



# 完整内容获取方式

由于合规及安全考量，本仓库不直接托管任何逆向代码、脱壳数据、完整调用链及最新版绕过思路。

如需以下完整资料，请联系获取：

- 各签名完整调用链伪代码
    
- 最新版 App 的适配偏移量数据
    
- 已脱壳 so 文件
    

覆盖：TikTok签名机46.7.3、抖音_TikTok国际版、抖音国内七神、抖音国内版Frida签名机、抖音设备签名、番茄小说、红果短剧、剪影、今日头条、皮皮虾、西瓜视频。

## 私域入口

因平台规则限制，请通过以下方式获取完整内容索引：

| 渠道     | 方式                  |
| ------ | ------------------- |
| QQ     | 340763383           |
| 添加时请备注 | **“算法签名”** ，否则不予通过。 |

## 免责声明

本页仅供安全研究、逆向工程技术交流与教育目的使用。严禁用于数据爬取、批量请求、绕过风控或违反平台协议的行为。使用者自行承担一切法律后果。

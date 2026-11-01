# ESP8266 + OneNET OTA（HTTP + Range）实测记录

> **用途**：离线回看用。忘了 OTA 的 HTTP 接口怎么调、AT 指令怎么发、参数填什么，直接翻这一份就能复现"成功把 OneNET 平台上的升级包 bin 收进 ESP8266"。
> **实测日期**：2026-10-23
> **结论**：✅ 全部打通。`check` 拿到任务 → `Range` 分段下载 → 收到 1024 字节固件头 → 向量表校验通过。
> **测试方式**：ESP-01S 拆下，单独接 USB-TTL 到电脑，用 Windows 自带 PowerShell + `System.IO.Ports.SerialPort` 驱动（无需装 Python / 串口助手），**不接开发板**。

---

## 0. 一句话结论

`iot-api.heclouds.com:80` 的 fuse-ota 接口在 **明文 HTTP** 下完全可用；ESP8266 用**裸 TCP（`CIPSTART`/`CIPSEND`/`+IPD`）**就能把升级包拉回来，且字节级无损。

**关键参数只有一个坑**：`check` 的 `type` 必须是 **1**（不是源码里写的 2）。

---

## 1. 环境与硬件

| 项 | 值 |
|---|---|
| 模组 | ESP-12F（ESP8266，1MB Flash） |
| AT 固件 | `AT version:2.3.0.0-dev`，SDK `v3.4-22-g967752e2`，`Bin version:2.2.0(ESP8266_1MB)`，Ai-Thinker |
| 供电 | **3.3V / ≥300mA**（USB-TTL 的 3.3V 常带不动，会出现乱码/重启） |
| 接线 | ESP TX→USB-TTL RX，ESP RX→USB-TTL TX，EN/CH_PD→3.3V，RST→3.3V |
| 串口 | 电脑 `COM3`，**115200 8N1**，无流控 |
| 工具 | PowerShell 5.x（Windows 自带，无需装任何东西） |
| 工作站 WiFi | SSID `nanGT8` / 密码 `20061225`（2.4GHz，ESP8266 不支持 5GHz） |

---

## 2. 固定参数（速查表）

### 2.1 OneNET 平台

| 项 | 值 | 备注 |
|---|---|---|
| 产品**名称** | `OTA_Test` | ⚠️ 只是显示名，**不能**填进 URL |
| 产品 **ID** | **`FRfr9m3Ppg`** | ✅ URL 里用这个 |
| 设备名称 | **`OTA_Test_drive`** | URL 里用这个 |
| token | 见下 | 有效期 `et=1809960178` ≈ **2027-05-10** |
| API 域名 | `iot-api.heclouds.com` | 实测 **80 端口明文 HTTP 可用** |
| 服务器 | nginx | |
| 升级任务 | `OTA_1`，批次 ID `1076481`，目标版本 `1.0.1`，静态升级，重试 10 次，超时 60 分钟 | |

**token（原样复制，不要改大小写/不要 URL 解码）**

```
version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D
```

> 这个 token **同时**用作 MQTT 的 password 和 HTTP 的 `Authorization` 头（f407 工程里 `OTA_HTTP_AUTH = OTA_MQTT_PASS`）。

### 2.2 接口清单

| 用途 | 方法 | 路径 | 附加头 | 请求体 |
|---|---|---|---|---|
| 上报版本 | POST | `/fuse-ota/{pid}/{dev}/version` | `Content-Type: application/json` | `{"s_version":"1.0.0","f_version":"1.0.0"}` |
| 查任务 | GET | `/fuse-ota/{pid}/{dev}/check?type=1&version=1.0.0` | — | — |
| 下载 | GET | `/fuse-ota/{pid}/{dev}/{tid}/download` | `Range: bytes=N-` | — |
| 报进度/状态 | POST | `/fuse-ota/{pid}/{dev}/{tid}/status` | `Content-Type: application/json` | `{"step":N}` |
| 统一鉴权头 | — | — | `Authorization: <token>` | — |

### 2.3 `type` 枚举（**实测出来的，别照抄源码**）

| 请求 | 平台返回 | 结论 |
|---|---|---|
| `type=1` | `{"code":0,...}` | ✅ **正确值**（SOTA 应用升级整包） |
| `type=2` | `{"code":12010,"msg":"task type error"}` | ❌ 源码里写的 2 是错的 |
| `type=3` | `{"code":12004,"msg":"format error"}` | ❌ 超范围 |
| 不带 type | `{"code":12001,"msg":"Required int parameter 'type' is not present"}` | 必填 |

---

## 3. 实测结果（成功证据存档）

### 3.1 `POST /version` → 成功

```json
{"code":0,"msg":"succ","request_id":"005048296ba34314a6034448e80cbe46"}
```

平台侧效果：控制台里设备的"当前版本号"从 `-` 变为 `1.0.0`。

### 3.2 `GET /check?type=1&version=1.0.0` → **拿到任务**

```json
{"code":0,"msg":"succ","data":{
  "target":"1.0.1",
  "tid":1528429,
  "size":72164,
  "md5":"a41bf71a92431e837b3b0ec7a80e0040",
  "status":1,
  "type":1
},"request_id":"84f4ff792e5446f0862c254c758d234f"}
```

**这是"平台能把任务下发"的铁证。**

### 3.3 `GET .../{tid}/download` + `Range: bytes=0-1023` → **收到固件**

响应头：

```
HTTP/1.1 206 Partial Content
Date: Fri, 23 Oct 2026 15:58:50 GMT
Content-Type: application/octet-stream;charset=UTF-8
Content-Length: 1024
Connection: close
Content-Disposition: attachment; filename="OTA1"
Content-Range: bytes 0-1023/72164
Ota-Errno: 0
Server: nginx
```

串口收到的原始帧：

```
Recv 287 bytes
SEND OK
+IPD,1385:HTTP/1.1 206 Partial Content ...（361 字节头）... + 1024 字节二进制 + CLOSED
```

`361 + 1024 = 1385` = `+IPD` 声明长度 → **一个字节都没丢**。

固件前 48 字节（小端）：

```
0000  38 DE 00 20 F9 03 01 08 7B 96 01 08 B1 54 01 08
0010  79 96 01 08 7D 21 01 08 1D AE 01 08 00 00 00 00
0020  00 00 00 00 00 00 00 00 00 00 00 00 39 03 01 08
```

解析出的向量表：

| 项 | 值 | 校验 |
|---|---|---|
| 初始 MSP | `0x2000DE38` | ✅ 在 SRAM `0x20000000~0x20030000` 内 |
| Reset_Handler | `0x080103F9` | ✅ 在 App 槽 `0x08010000` 之后 |
| 其余向量 | `0x0801967B / 0x080154B1 / 0x0801217D / 0x0801AE1D / 0x08010339` | 全部落在 `0x0801xxxx`，整表一致 |

> **顺带发现**：MSP 是 `0x2000DE38` 而不是 `0x20030000`，说明 App 的**主栈只有 1KB**（`startup_stm32f429xx.s` 里 `Stack_Size EQU 0x400`），且栈顶贴着 RW/ZI 区末尾，后面 138KB SRAM 空着。
> **影响**：① BL 的 `app_jump_validate` 要把 RAM 范围放宽到整个 `0x20000000~0x20030000`，不能假设等于 `0x20030000`；② App 里别在 `main()` 阶段放大局部数组（1KB 容易爆），建议把 `Stack_Size` 提到 `0x1000`。

---

## 4. 一键复现脚本（推荐，直接整段粘进 PowerShell）

做三件事：联网 → 上报版本 → **check + 下发校验**。`COM3` 换成实际串口。

```powershell
$PORT = "COM3"
$PROD = "FRfr9m3Ppg"
$DEV  = "OTA_Test_drive"
$TOK  = "version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D"

if ($sp -and $sp.IsOpen) { $sp.Close() }
$sp = New-Object System.IO.Ports.SerialPort $PORT,115200,"None",8,"One"
$sp.ReadTimeout=200; $sp.WriteTimeout=1000
$sp.DtrEnable=$false; $sp.RtsEnable=$false
$sp.Encoding = [Text.Encoding]::GetEncoding(28591)   # Latin-1：1 字符 = 1 字节，二进制安全
$sp.Open(); Write-Host "已打开 $PORT" -ForegroundColor Green

function Cmd([string]$c,[string]$wait="OK",[double]$tmo=6){
    Write-Host "`n>>> $c" -ForegroundColor Cyan
    $sp.Write($c + "`r`n")
    $buf=""; $sw=[Diagnostics.Stopwatch]::StartNew()
    while($sw.Elapsed.TotalSeconds -lt $tmo){
        try{$buf+=$sp.ReadExisting()}catch{}
        if($buf.Contains($wait)){break}
        Start-Sleep -Milliseconds 20
    }
    Write-Host $buf
    $buf
}

function HexDump([string]$s,[int]$max=48){
    $n=[Math]::Min($s.Length,$max)
    for($i=0;$i -lt $n;$i+=16){
        $line=""
        for($j=$i;$j -lt [Math]::Min($i+16,$n);$j++){ $line += ("{0:X2} " -f [int][char]$s[$j]) }
        Write-Host ("{0:X4}  {1}" -f $i,$line)
    }
}

function Http([string]$title,[string]$req,[double]$tmo=10){
    Write-Host "`n=========== $title ===========" -ForegroundColor Magenta
    Write-Host ("自检首行: " + $req.Split("`r`n")[0]) -ForegroundColor DarkGray
    $b=[Text.Encoding]::ASCII.GetBytes($req)
    Write-Host ("请求 {0} 字节" -f $b.Length) -ForegroundColor DarkGray
    Cmd "AT+CIPCLOSE" "OK" 1 | Out-Null
    $r = Cmd 'AT+CIPSTART="TCP","iot-api.heclouds.com",80' "OK" 8
    if(-not $r.Contains("CONNECT")){ Write-Host "CIPSTART 失败" -ForegroundColor Red; return }
    $sp.Write("AT+CIPSEND=$($b.Length)`r`n")
    $buf=""; $sw=[Diagnostics.Stopwatch]::StartNew()
    while($sw.Elapsed.TotalSeconds -lt 5){ try{$buf+=$sp.ReadExisting()}catch{}; if($buf.Contains(">")){break}; Start-Sleep -Milliseconds 20 }
    $sp.Write($b,0,$b.Length)
    $resp=""; $sw=[Diagnostics.Stopwatch]::StartNew()
    while($sw.Elapsed.TotalSeconds -lt $tmo){
        try{$resp += $sp.ReadExisting()}catch{}
        if($resp.Contains("CLOSED")){break}
        Start-Sleep -Milliseconds 5
    }
    Write-Host $resp
    $script:lastResp = $resp
}

# ---------- 1. 联网 ----------
Cmd "AT"          | Out-Null
Cmd "ATE0"        | Out-Null
Cmd "AT+CWMODE=1" | Out-Null
Cmd 'AT+CWJAP="nanGT8","20061225"' "OK" 20 | Out-Null
Cmd "AT+CIPMUX=0" | Out-Null

# ---------- 2. 上报版本 ----------
$vb = '{"s_version":"1.0.0","f_version":"1.0.0"}'
Http "POST /version" ("POST /fuse-ota/$PROD/$DEV/version HTTP/1.1`r`n" +
    "Host: iot-api.heclouds.com`r`nContent-Type: application/json`r`n" +
    "Authorization: $TOK`r`nContent-Length: $($vb.Length)`r`n" +
    "Connection: close`r`n`r`n" + $vb)

# ---------- 3. check（type 必须是 1） ----------
Http "GET /check?type=1" ("GET /fuse-ota/$PROD/$DEV/check?type=1&version=1.0.0 HTTP/1.1`r`n" +
    "Host: iot-api.heclouds.com`r`nAuthorization: $TOK`r`n" +
    "Connection: close`r`n`r`n")

$sp.Close()
Write-Host "`n完成" -ForegroundColor Green
```

看到 `"code":0` 且 `data` 里有 `tid/size/md5/target` = 成功。

---

## 5. 下载 1KB 固件并校验（Range 验证）

```powershell
$PORT = "COM3"
$PROD = "FRfr9m3Ppg"
$DEV  = "OTA_Test_drive"
$TID  = 1528429        # ← 换成 check 返回的 tid
$TOK  = "version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D"

if ($sp -and $sp.IsOpen) { $sp.Close() }
$sp = New-Object System.IO.Ports.SerialPort $PORT,115200,"None",8,"One"
$sp.ReadTimeout=200; $sp.WriteTimeout=1000
$sp.DtrEnable=$false; $sp.RtsEnable=$false
$sp.Encoding = [Text.Encoding]::GetEncoding(28591)
$sp.Open()

function Cmd([string]$c,[string]$wait="OK",[double]$tmo=6){
    Write-Host "`n>>> $c" -ForegroundColor Cyan
    $sp.Write($c + "`r`n")
    $buf=""; $sw=[Diagnostics.Stopwatch]::StartNew()
    while($sw.Elapsed.TotalSeconds -lt $tmo){
        try{$buf+=$sp.ReadExisting()}catch{}
        if($buf.Contains($wait)){break}
        Start-Sleep -Milliseconds 20
    }
    Write-Host $buf; $buf
}
function HexDump([string]$s,[int]$max=48){
    $n=[Math]::Min($s.Length,$max)
    for($i=0;$i -lt $n;$i+=16){
        $line=""
        for($j=$i;$j -lt [Math]::Min($i+16,$n);$j++){ $line += ("{0:X2} " -f [int][char]$s[$j]) }
        Write-Host ("{0:X4}  {1}" -f $i,$line)
    }
}

Cmd "AT" | Out-Null
Cmd "ATE0" | Out-Null
Cmd "AT+CWMODE=1" | Out-Null
Cmd 'AT+CWJAP="nanGT8","20061225"' "OK" 20 | Out-Null
Cmd "AT+CIPMUX=0" | Out-Null
Cmd "AT+CIPCLOSE" "OK" 1 | Out-Null
Cmd 'AT+CIPSTART="TCP","iot-api.heclouds.com",80' "OK" 8 | Out-Null

$req = "GET /fuse-ota/$PROD/$DEV/$TID/download HTTP/1.1`r`n" +
       "Host: iot-api.heclouds.com`r`n" +
       "Authorization: $TOK`r`n" +
       "Range: bytes=0-1023`r`n" +
       "Connection: close`r`n`r`n"
$b = [Text.Encoding]::ASCII.GetBytes($req)
Write-Host ("`n请求 {0} 字节 : {1}" -f $b.Length, $req.Split("`r`n")[0]) -ForegroundColor DarkGray

$sp.Write("AT+CIPSEND=$($b.Length)`r`n")
$buf=""; $sw=[Diagnostics.Stopwatch]::StartNew()
while($sw.Elapsed.TotalSeconds -lt 5){ try{$buf+=$sp.ReadExisting()}catch{}; if($buf.Contains(">")){break}; Start-Sleep -Milliseconds 20 }
$sp.Write($b,0,$b.Length)

$resp=""; $sw=[Diagnostics.Stopwatch]::StartNew()
while($sw.Elapsed.TotalSeconds -lt 20){
    try{$resp += $sp.ReadExisting()}catch{}
    if($resp.Contains("CLOSED")){break}
    Start-Sleep -Milliseconds 5
}
$sp.Close()

# 先找 +IPD, 再按声明长度精确截取（★ 不要直接找第一个 CRLFCRLF）
$i = $resp.IndexOf("+IPD,")
if($i -lt 0){ Write-Host "没发现 +IPD：" -ForegroundColor Red; Write-Host $resp; return }
$c = $resp.IndexOf(":", $i)
$ipdLen  = [int]$resp.Substring($i+5, $c-$i-5)
$payload = $resp.Substring($c+1, [Math]::Min($ipdLen, $resp.Length-$c-1))
$e = $payload.IndexOf("`r`n`r`n")
$httpH = $payload.Substring(0,$e)
$bin   = $payload.Substring($e+4)

Write-Host "`n===== HTTP 响应头 =====" -ForegroundColor Yellow
Write-Host $httpH
Write-Host ("`n+IPD 声明: {0} / 头长度: {1} / 正文: {2} 字节" -f $ipdLen, ($e+4), $bin.Length) -ForegroundColor Yellow
Write-Host "`n===== 固件前 48 字节 =====" -ForegroundColor Yellow
HexDump $bin 48

if($bin.Length -ge 8){
    $tmp = New-Object byte[] 8
    for($k=0;$k -lt 8;$k++){ $tmp[$k] = [byte][int][char]$bin[$k] }
    $msp   = [BitConverter]::ToUInt32($tmp,0)
    $reset = [BitConverter]::ToUInt32($tmp,4)
    Write-Host ("`nMSP   = 0x{0:X8}" -f $msp)   -ForegroundColor Cyan
    Write-Host ("Reset = 0x{0:X8}" -f $reset) -ForegroundColor Cyan
    if($msp -ge 0x20000000 -and $msp -le 0x20030000){ Write-Host "  MSP   OK" -ForegroundColor Green } else { Write-Host "  MSP   异常" -ForegroundColor Red }
    if($reset -ge 0x08010000 -and $reset -lt 0x08100000){ Write-Host "  Reset OK" -ForegroundColor Green } else { Write-Host "  Reset 异常" -ForegroundColor Red }
}
```

### 通过的判据（三条全中才算成功）

| 判据 | 期望 | 实测值 |
|---|---|---|
| ① 状态码 | `206 Partial Content` | ✅ 206 |
| ② 响应头 | 有 `Content-Range: bytes 0-1023/72164` 且 `Ota-Errno: 0` | ✅ |
| ③ 正文 | 1024 字节，且开头是 `38 DE 00 20 F9 03 01 08`（MSP/Reset） | ✅ |

---

## 6. 纯 AT 手动流程（没 PowerShell 时用）

> **纪律：一条一条发，发一条等应答再发下一条。** 整段粘贴必失败（见第 8 节坑 1）。

| # | 发送 | 等什么 |
|---|---|---|
| 1 | `AT` | `OK` |
| 2 | `ATE0` | `OK`（关回显） |
| 3 | `AT+CWMODE=1` | `OK` |
| 4 | `AT+CWJAP="nanGT8","20061225"` | `WIFI GOT IP` + `OK` |
| 5 | `AT+CIPMUX=0` | `OK` |
| 6 | `AT+CIPSTART="TCP","iot-api.heclouds.com",80` | `CONNECT` + `OK` |
| 7 | `AT+CIPSEND=276` | **先 `OK`，再 `>`**（两个都要等到） |
| 8 | 粘贴下面 5 行 | `Recv 276 bytes` → `SEND OK` |
| 9 | **什么都别发**，等 3~8 秒 | `+IPD,...` → `CLOSED` |

**第 8 步内容（check，共 276 字节）**

```
GET /fuse-ota/FRfr9m3Ppg/OTA_Test_drive/check?type=1&version=1.0.0 HTTP/1.1
Host: iot-api.heclouds.com
Authorization: version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D
Connection: close

```

**下载请求（共 287 字节，tid 为 7 位时）**

```
GET /fuse-ota/FRfr9m3Ppg/OTA_Test_drive/1528429/download HTTP/1.1
Host: iot-api.heclouds.com
Authorization: version=2018-10-31&res=products%2FFRfr9m3Ppg%2Fdevices%2FOTA_Test_drive&et=1809960178&method=sha1&sign=kLYbgpjIMWGT9klNJKWXX15G3GI%3D
Range: bytes=0-1023
Connection: close

```

### 各请求的字节数（手动填 `AT+CIPSEND=N` 用）

| 请求 | 字节数 | 备注 |
|---|---|---|
| `GET /` 最小连通性测试 | **65** | 只有 Host + Connection |
| `check?type=N&version=1.0.0` | **276** | pid 10 字符 |
| `download`（tid 7 位 = 1528429） | **287** | 长度 = `287 + (tid 位数 - 7)` |
| `POST /version` | **351** | body 41 字节 |

字节数构成（以 check 276 为例，方便自己重算）：

| 内容 | 字节 |
|---|---|
| 请求行 | 75 |
| CRLF | 2 |
| `Host: iot-api.heclouds.com` | 26 |
| CRLF | 2 |
| `Authorization: ` | 15 |
| token | 133 |
| CRLF | 2 |
| `Connection: close` | 17 |
| CRLF | 2 |
| 空行 CRLF | 2 |
| **合计** | **276** |

> 发完看模块回的 `Recv <n> bytes`：**数字对得上才算请求完整**。

---

## 7. 响应与错误码速查

| code | msg | 含义 | 处理 |
|---|---|---|---|
| `0` | `succ` | 成功 | `data` 为空 = 当前无任务（正常） |
| `10403` | `auth failed: pid not equal` | **URL 里的 pid ≠ token 里的 pid** | 检查 URL 里的产品 ID 是不是 `FRfr9m3Ppg`（不是产品名 `OTA_Test`，也不是进程号） |
| `12010` | `task type error` | `type` 值不对 | 改成 `type=1` |
| `12004` | `format error` | `type` 超范围（如 3） | 只能用 1 |
| `12001` | `parameter required ... 'type' is not present` | 少了 `type` | 加上 `type=1` |
| HTTP `401` | — | token 过期/错误 | 用设备密钥重算 token（`method=sha1`） |

下载接口特有的响应头：

| 头 | 含义 |
|---|---|
| `Ota-Errno` | `0` = 正常；>0 = 平台侧业务错误（f407 的 `http_client` 会解析它） |
| `Content-Range: bytes 0-1023/72164` | 当前区间 / 总大小 |
| `Content-Disposition: attachment; filename="OTA1"` | 平台上的升级包文件名 |

---

## 8. 七个坑（含当时真实报错）

### 坑 1：整段粘贴 → `busy p...` → 后面全 `ERROR`

**现象**

```
AT+CIPSEND=65
OK
GET / HTTP/1.1          ← 请求粘在了 > 之前
...
>
busy p...
Recv 65 bytes
SEND OK
POST /fuse-ota/...      ← 此时已回到命令模式，被当 AT 命令解析
ERROR
ERROR
```

**原因**：ESP-AT 不是流式接口，`AT+CIPSEND` 之后要**等 `>`**；发送期间收到的多余字节会回 `busy p...`；`SEND OK` 之前发任何命令都会被打回。

**对策**：用第 4 节的脚本（每条命令都等应答、`SEND OK` 后单独等 `+IPD`），或手动严格一条一条发。

### 坑 2：`>` 提示符"看不到"

`>` 是 `0x3E` 一个字节，**没有换行**，紧跟在 `OK` 后面（可能显示成 `OK>` 或单独一行开头），部分串口助手会把它当控制字符过滤。
**对策**：用 Tera Term / SecureCRT 看原始数据；或直接观察后续是否出现 `Recv N bytes`。

### 坑 3：`$PID` 是 PowerShell 只读变量

```powershell
$PID = "FRfr9m3Ppg"      # ← 报"无法覆盖变量 PID"
```

`$PID` 是当前进程号，赋值被静默拒绝，URL 就变成 `/fuse-ota/12345/...`，平台回 **`10403 auth failed: pid not equal`**。
**对策**：变量名用 `$PROD`（本文脚本已改）。

### 坑 4：`SEND OK` 后立刻发下一条命令，把 `+IPD` 丢了

**现象**：只看到 `SEND OK`，没有 `+IPD`；紧接着 `AT+CIPCLOSE`/`AT+CIPSTART` 报 `ERROR`/`CLOSED`。
**对策**：`SEND OK` 之后**等 3~8 秒**，看到 `+IPD` 和 `CLOSED` 再发下一条。

### 坑 5：解析时找第一个 `\r\n\r\n` → 错位

**现象**：正文字节数 1406（期望 1024），HEX 打出来是 `53 45 4E 44 20 4F 4B`（"SEND OK"）。
**原因**：`SEND OK` 后面也有一个空行，被当成了 HTTP 头结束。
**对策**：先定位 **`+IPD,`** → 读出声明的长度 → **按长度精确截取**，再在 payload 内部找头/体分隔。

### 坑 6：串口助手与 PowerShell 抢串口

串口是独占的。助手没关 → PowerShell `Open()` 报 `拒绝访问`。
**对策**：一次只用一个工具。

### 坑 7：3.3V 供电不足

**现象**：一上电就刷 `ets Jan 8 2013`，或 AT 无响应。
**对策**：独立 3.3V/≥300mA 供电；脚本里已设 `DtrEnable/RtsEnable = $false`，避免 USB-TTL 的 DTR/RTS 把模块顶进下载模式。

---

## 9. 对工程移植的结论（**必改项**）

| # | 位置 | 改动 | 依据 |
|---|---|---|---|
| 1 | `App/Core/ota/ota_config.h` | `OTA_TASK_TYPE` **2 → 1** | `type=2` 返回 `12010`；`type=1` 返回 `code:0` |
| 2 | 同上 | `OTA_API_HOST="iot-api.heclouds.com"`、`OTA_API_PORT=80` | 实测可用（明文 HTTP） |
| 3 | 同上 | `OTA_HTTP_AUTH` = 上面的 token | 实测 `code:0` |
| 4 | 同上 | `OTA_APP_VERSION_STR` 必须与平台升级包版本一致（本次对比：设备 `1.0.0`、目标 `1.0.1`） | check 才能匹配到任务 |
| 5 | `Boot/boot_config.h` / `app_jump.c` | RAM 校验范围必须是 `0x20000000~0x20030000` 整区间 | 实测 MSP = `0x2000DE38`，不是 SRAM 顶端 |
| 6 | `startup_stm32f429xx.s`（建议） | `Stack_Size 0x400` → `0x1000` | App 主栈只有 1KB，加 OTA 代码容易爆 |
| 7 | 结论 | `esp01s.c` + `http_client.c` + `mqtt_client.c` 可以直接复用到 F429 | 裸 TCP / `+IPD` / HTTP / Range 全部实测通过 |

**本次未验证、留到真机联调的部分**（别误以为已经验过）：

- `POST /{tid}/status` 上报进度（`{"step":N}`）与 201 成功上报 —— 会改变平台任务状态，单测阶段故意没调；
- 多帧 `+IPD` 拼接、几十 KB 连续下载的吞吐与丢包（PC 端 1KB 测试覆盖不到）；
- MQTT `ota/inform` 通知（本次只走 HTTP 轮询）。

---

## 10. 复现检查清单（下次照着打勾）

- [ ] 模块插好：3.3V / TX-RX 交叉 / EN 拉高 / 115200 8N1
- [ ] `[System.IO.Ports.SerialPort]::GetPortNames()` 查端口（如 COM3）
- [ ] 关掉串口助手（避免独占）
- [ ] token 未过期（`et=1809960178` ≈ 2027-05-10）
- [ ] 平台上有该设备的升级任务（目标版本 > 设备当前版本）
- [ ] 跑第 4 节脚本 → `POST /version` 回 `code:0`
- [ ] `check?type=1` 回 `code:0` 且 `data.tid` 非空
- [ ] 拿 `tid` 跑第 5 节脚本 → `206 Partial Content` + `Ota-Errno: 0`
- [ ] 正文 1024 字节，开头 `38 DE 00 20 F9 03 01 08`（MSP/Reset 合理）
- [ ] 把 `tid/size/md5` 记下来，与平台控制台显示一致

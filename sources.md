# Исходники: скрипты сервера (без паролей и ключей)

Скрипты рабочие, но **пароли и секреты вырезаны** — перед запуском замените плейсхолдеры:

- `YOUR_SERVER_DOMAIN` — ваш домен;
- `PASSWORD_ADMIN`, `PASSWORD_FAMILY`, `PASSWORD_USER3` — пароли аккаунтов;
- `YOUR_API_KEY`, `YOUR_API_SECRET`, `YOUR_IP` — из `livekit generate-keys`.

Скрипты написаны под Windows PowerShell 5.1: кодировка ASCII, кириллицу в выводе не используйте (в консоли будет «моjибака»), обращения `$pid` нет, команд `head`/`cut`/`tail` нет.

---

## 1. set-direct.ps1 — делает комнаты личками (чинит окно звонка)

Находит комнаты, где ровно два участника, и прописывает каждую в `m.direct` под ключом второго участника. Существующие записи сохраняются. Логины кэшируются в `%TEMP%\mx-tokens.json`, повторные попытки ждут (у Synapse есть rate limit на логин — HTTP 429).

```powershell
# Adds rooms with exactly two members to the "m.direct" account data of each user.
# Effect: Element X treats such rooms as DMs, so autoLeaveWhenOthersLeft becomes true
#         and the call window closes by itself after the other side hangs up.
# Side effect: those rooms appear as personal chats in the room list.
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$ErrorActionPreference = "Stop"
$hs = "https://YOUR_SERVER_DOMAIN"
$tf = "$env:TEMP\mx-tokens.json"

function Login([string]$u, [string]$p) {
    $body = @{ identifier = @{ type = "m.id.user"; user = $u }; password = $p; type = "m.login.password" } | ConvertTo-Json -Depth 5
    $sleep = 20
    for ($i = 0; $i -lt 5; $i++) {
        try {
            $r = Invoke-RestMethod -Uri "$hs/_matrix/client/v3/login" -Method Post -Body $body -ContentType "application/json" -TimeoutSec 30
            return $r.access_token
        } catch {
            $code = 0
            try { $code = [int]$_.Exception.Response.StatusCode } catch { }
            Write-Host ("    login try " + ($i + 1) + " http=" + $code + " - wait " + $sleep + "s")
            Start-Sleep -Seconds $sleep
            $sleep = $sleep + 20
        }
    }
    return $null
}

$toks = @{}
if (Test-Path $tf) { try { $toks = Get-Content $tf -Raw | ConvertFrom-Json } catch { $toks = @{} } }

$users = @(
    @{ u = "admin";  p = "PASSWORD_ADMIN" },
    @{ u = "family"; p = "PASSWORD_FAMILY" },
    @{ u = "user3";  p = "PASSWORD_USER3" }
)

# pass 1: tokens (reuse cache, login only if needed)
foreach ($u in $users) {
    $name = $u.u
    $tok = $toks.$name
    if ($tok) {
        try {
            Invoke-RestMethod -Uri "$hs/_matrix/client/v3/account/whoami" -Headers @{ Authorization = "Bearer $tok" } | Out-Null
            Write-Host ("token ok: " + $name)
            continue
        } catch { Write-Host ("token dead: " + $name) }
    }
    $tok = Login $name $u.p
    if ($tok) {
        $toks | Add-Member -NotePropertyName $name -NotePropertyValue $tok -Force
        $toks | ConvertTo-Json -Depth 5 | Set-Content -Path $tf -Encoding ASCII
        Write-Host ("token new: " + $name)
    } else { Write-Host ("LOGIN FAILED: " + $name) }
}

# pass 2: room membership report (seen as admin)
$tok = $toks.admin
if ($tok) {
    $h = @{ Authorization = "Bearer $tok" }
    $rooms = @((Invoke-RestMethod -Uri "$hs/_matrix/client/v3/joined_rooms" -Headers $h).joined_rooms)
    Write-Host "--- room membership"
    foreach ($rid in $rooms) {
        $jm = $null
        try { $jm = Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/" + $rid + "/joined_members") -Headers $h } catch { Write-Host ("  " + $rid + " ERR"); continue }
        $m = @($jm.joined.PSObject.Properties.Name)
        Write-Host ("  " + $rid + "  [" + ($m -join ", ") + "]")
    }
}

# pass 3: add rooms to m.direct
foreach ($u in $users) {
    $name = $u.u
    $tok = $toks.$name
    if (-not $tok) { Write-Host (">>> " + $name + " : no token, skipped"); continue }
    $h = @{ Authorization = "Bearer $tok" }
    $mxid = $null
    try { $mxid = (Invoke-RestMethod -Uri "$hs/_matrix/client/v3/account/whoami" -Headers $h).user_id } catch { }
    if (-not $mxid) { Write-Host (">>> " + $name + " : whoami failed"); continue }
    Write-Host (">>> " + $mxid)

    # NOTE: /joined_rooms returns {"joined_rooms": [...]}, NOT {"rooms": [...]}
    $rooms = @((Invoke-RestMethod -Uri "$hs/_matrix/client/v3/joined_rooms" -Headers $h).joined_rooms)
    Write-Host ("  joined=" + $rooms.Count)

    $url = "$hs/_matrix/client/v3/user/" + [uri]::EscapeDataString($mxid) + "/account_data/m.direct"
    $cur = $null
    try { $cur = Invoke-RestMethod -Uri $url -Headers $h } catch { $cur = $null }
    $md = @{}
    if ($null -ne $cur) { foreach ($pp in $cur.PSObject.Properties) { $md[$pp.Name] = @($pp.Value) } }
    $before = ($md | ConvertTo-Json -Depth 8 -Compress); if (-not $before) { $before = "{}" }
    Write-Host ("  before: " + $before)

    $changed = $false
    foreach ($rid in $rooms) {
        $jm = $null
        try { $jm = Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/" + $rid + "/joined_members") -Headers $h } catch { continue }
        $joined = @($jm.joined.PSObject.Properties.Name)
        if ($joined.Count -ne 2) { Write-Host ("  " + $rid + " members=" + $joined.Count + " -> not 2, skip"); continue }
        if ($joined -notcontains $mxid) { continue }
        $others = @($joined | Where-Object { $_ -ne $mxid })
        if ($others.Count -ne 1) { continue }
        $other = $others[0]
        $already = $false
        if ($md.ContainsKey($other)) { $already = (@($md[$other]) -contains $rid) }
        if ($already) { Write-Host ("  " + $rid + " -> " + $other + " [already]"); continue }
        if ($md.ContainsKey($other)) { $md[$other] = @($md[$other]) + $rid } else { $md[$other] = @($rid) }
        $changed = $true
        Write-Host ("  " + $rid + " -> " + $other + " [ADDED]")
    }

    if ($changed) {
        $json = ($md | ConvertTo-Json -Depth 8)
        Invoke-RestMethod -Uri $url -Method Put -Headers $h -Body $json -ContentType "application/json" | Out-Null
        $back = Invoke-RestMethod -Uri $url -Headers $h
        Write-Host ("  PUT ok, after: " + ($back | ConvertTo-Json -Depth 8 -Compress))
    } else { Write-Host "  no changes" }
    Start-Sleep -Seconds 2
}
Write-Host "DONE"
```

---

## 2. verify-direct.ps1 — проверка результата

Читает `m.direct` у каждого аккаунта и показывает комнаты вместе с числом участников.

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$ErrorActionPreference = "Stop"
$hs = "https://YOUR_SERVER_DOMAIN"
$tf = "$env:TEMP\mx-tokens.json"

$toks = Get-Content $tf -Raw | ConvertFrom-Json

foreach ($name in @("admin", "family", "user3")) {
    $tok = $toks.$name
    if (-not $tok) { Write-Host ("=== " + $name + " : no token"); continue }
    $h = @{ Authorization = "Bearer $tok" }
    $mxid = $null
    try { $mxid = (Invoke-RestMethod -Uri "$hs/_matrix/client/v3/account/whoami" -Headers $h).user_id } catch { }
    if (-not $mxid) { Write-Host ("=== " + $name + " : token dead"); continue }

    $url = "$hs/_matrix/client/v3/user/" + [uri]::EscapeDataString($mxid) + "/account_data/m.direct"
    $md = $null
    try { $md = Invoke-RestMethod -Uri $url -Headers $h } catch { $md = $null }
    Write-Host ("=== " + $mxid)
    if ($null -eq $md) { Write-Host "   m.direct = (absent)"; continue }

    foreach ($pp in $md.PSObject.Properties) {
        $rooms = @($pp.Value)
        Write-Host ("   " + $pp.Name)
        foreach ($r in $rooms) {
            $jm = $null
            $cnt = "?"
            try {
                $jm = Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/" + $r + "/joined_members") -Headers $h
                $cnt = @($jm.joined.PSObject.Properties.Name).Count
            } catch { $cnt = "ERR" }
            Write-Host ("      " + $r + "  members=" + $cnt)
        }
    }
}
Write-Host "DONE"
```

---

## 3. room-info.ps1 — обзор комнат

Имя комнаты, личная она или группа, число участников, включено ли шифрование.

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$ErrorActionPreference = "SilentlyContinue"
$hs = "https://YOUR_SERVER_DOMAIN"
$user = "admin"
$pass = "PASSWORD_ADMIN"
$body = @{identifier = @{type = "m.id.user"; user = $user}; password = $pass; type = "m.login.password"} | ConvertTo-Json -Depth 5
try {
    $r = Invoke-RestMethod -Uri "$hs/_matrix/client/v3/login" -Method Post -Body $body -ContentType "application/json" -TimeoutSec 30
    $tok = $r.access_token
    Write-Host "LOGIN OK"
} catch {
    Write-Host ("LOGIN FAIL: " + $_.Exception.Message)
    exit 1
}
$rooms = (Invoke-RestMethod -Uri "$hs/_matrix/client/v3/joined_rooms" -Headers @{Authorization="Bearer $tok"}).joined_rooms
Write-Host ("rooms=" + $rooms.Count)
foreach ($rv in $rooms) {
    $enc = "?"
    try {
        $ev = Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/$([uri]::EscapeDataString($rv))/state/m.room.create") -Headers @{Authorization="Bearer $tok"}
        $enc = if ($ev.encryption) { "enc" } else { "plain" }
    } catch { $enc = "err" }
    $direct = "group"
    $n = -1
    try {
        $mem = Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/$([uri]::EscapeDataString($rv))/members") -Headers @{Authorization="Bearer $tok"}
        $vals = @($mem.chunk | ForEach-Object { $_.content.is_direct })
        if ($vals -contains $true) { $direct = "DIRECT" }
        $n = @($mem.chunk).Count
    } catch { }
    $name = "-"
    try {
        $nm = (Invoke-RestMethod -Uri ("$hs/_matrix/client/v3/rooms/$([uri]::EscapeDataString($rv))/state/m.room.name") -Headers @{Authorization="Bearer $tok"}).name
        if ($nm) { $name = $nm }
    } catch { }
    Write-Host ($rv + "  name=" + $name + "  " + $direct + "  members=" + $n + "  " + $enc)
}
```

> Важно: `m.room.name` не ставится для личных комнат, и это нормально. Считать личку по `is_direct` на событии участника бесполезно — matrix-rust-sdk для присоединённых комнат смотрит только на `m.direct`.

---

## 4. show-mdirect.ps1 — что лежит в m.direct у всех аккаунтов

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$ErrorActionPreference = "SilentlyContinue"
$hs = "https://YOUR_SERVER_DOMAIN"

function Get-Token($user, $pass) {
    $body = @{identifier = @{type = "m.id.user"; user = $user}; password = $pass; type = "m.login.password"} | ConvertTo-Json -Depth 5
    try {
        $r = Invoke-RestMethod -Uri "$hs/_matrix/client/v3/login" -Method Post -Body $body -ContentType "application/json" -TimeoutSec 30
        return $r.access_token
    } catch {
        Write-Host ("LOGIN FAIL " + $user + ": " + $_.Exception.Message)
        return $null
    }
}

foreach ($u in @(@("admin","PASSWORD_ADMIN"), @("family","PASSWORD_FAMILY"), @("user3","PASSWORD_USER3"))) {
    $tok = Get-Token $u[0] $u[1]
    if (-not $tok) { continue }
    Write-Host ("=== " + $u[0])
    $mxid = (Invoke-RestMethod -Uri "$hs/_matrix/client/v3/account/whoami" -Headers @{Authorization="Bearer $tok"}).user_id
    Write-Host ("mxid=" + $mxid)
    try {
        $d = Invoke-RestMethod -Uri "$hs/_matrix/client/v3/user/$([uri]::EscapeDataString($mxid))/account_data/m.direct" -Headers @{Authorization="Bearer $tok"}
        Write-Host ("m.direct = " + ($d | ConvertTo-Json -Depth 6 -Compress))
    } catch {
        Write-Host ("m.direct ERR " + $_.Exception.Message)
    }
    try {
        $p = Invoke-RestMethod -Uri "$hs/_matrix/client/v3/profile/$([uri]::EscapeDataString($mxid))" -Headers @{Authorization="Bearer $tok"}
        Write-Host ("displayname=" + $p.displayname)
    } catch {}
}
```

---

## 5. add-ring-rule.ps1 — звук входящего звонка

Push-правило на событие `org.matrix.msc4075.rtc.notification` (входящий звонок MatrixRTC). Скрипту нужны файлы логина `mx-login.json` / `mx-fam.json` в `%TEMP%` — формат обычного запроса Synapse на логин.

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$dir = "$env:TEMP"

function Add-RingRule($user, $jsonFile) {
    $resp = curl.exe -s -X POST "https://YOUR_SERVER_DOMAIN/_matrix/client/v3/login" -H "Content-Type: application/json" --data-binary "@$jsonFile"
    Write-Host ("  raw login for " + $user + ": " + ($resp -join ""))
    $j = $resp | ConvertFrom-Json
    if (-not $j.access_token) { Write-Host ("  LOGIN FAILED for " + $user); return }
    $tok = $j.access_token
    $ruleBody = '{"conditions":[{"kind":"event_match","key":"type","pattern":"org.matrix.msc4075.rtc.notification"}],"actions":["notify",{"set_tweak":"sound","value":"ring"},{"set_tweak":"highlight","value":false}]}'
    $out = "$dir\ring-rule.json"
    Out-File $out -InputObject $ruleBody -Encoding ascii -NoNewline
    $r = curl.exe -s -X PUT "https://YOUR_SERVER_DOMAIN/_matrix/client/v3/pushrules/global/override/msc4075_rtc_ring" -H "Authorization: Bearer $tok" -H "Content-Type: application/json" --data-binary "@$out"
    Write-Host ("  " + $user + " add-rule result: [" + ($r -join "") + "]")
    $chk = curl.exe -s "https://YOUR_SERVER_DOMAIN/_matrix/client/v3/pushrules/global/override/msc4075_rtc_ring" -H "Authorization: Bearer $tok"
    Write-Host ("  " + $user + " verify: " + ($chk -join "").Substring(0, [Math]::Min(220, ($chk -join "").Length)))
}

Write-Host "=== adding ring push rule for msc4075 rtc notifications ==="
Add-RingRule "admin" "$dir\mx-login.json"
Add-RingRule "family" "$dir\mx-fam.json"
```

> Громкость звонка — это настройка телефона, а не сервера. Правило только включает сигнал при входящем звонке.

---

## 6. Ошибки, на которые стоит знать

| Что случилось | Причина |
|---|---|
| Скрипт пишет `joined=1`, а комнат шесть | Ответ `/joined_rooms` — `{"joined_rooms": [...]}`, а не `{"rooms": [...]}`. Обращение к несуществующему полю молча даёт `@($null)` из одного элемента. |
| HTTP 429 при логине | Rate limit Synapse на логин. Ждать (успешные логины не чаще ~3 в секунду-полторы), кэшировать токены. |
| Звонок без видео/звука после перезагрузки сервера | LiveKit встал. `Restart-Service MatrixLiveKit`. |
| Все RTC-токены отваливаются с `M_UNAUTHORIZED` | Делегирование federation смотрит на закрытый 8448 — поправить `/.well-known/matrix/server` на `:443`. |
| Служба Caddy не перезапускается (`Access denied`) | Использовать `caddy reload --config ... --adapter caddyfile`. |

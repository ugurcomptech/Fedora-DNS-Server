# Fedora DNS Server

Fedorada DNS server kurulumu, sistem yöneticileri için önemli bir adımdır. Bu adım, DNS (Domain Name System) üzerinden alan adlarını IP adreslerine çözümleyen ve internet üzerindeki iletişimi kolaylaştıran bir hizmeti sağlar. Bu yazıda, Fedora üzerinde DNS server kurulumu ve yapılandırması için kullanılan Bind (Berkeley Internet Name Domain) hizmeti ele alınacaktır.

## Kurulum ve Konfigürasyon

Aşağıdaki komutu yazarak **BIND** rolünü kuruyoruz:

```
sudo dnf install bind bind-utils
```

bind rolü kurulduktan sonra servisin statusunu kontrol ediyoruz:

```
systemctl status named.service
```

Bir uyarı veya hata almadıysanız devam ediyoruz.

Gerekli konfigürasyonları yapmak için `nano /etc/named.conf` yoluna gidip gerekli tanımları yapıyoruz. `named.conf` dosyasını açtığınızda içerisinin dolu olduğunu göreceksiniz hazır olarak tanımlar yapılmış. İleride dosyada karmaşıklıklar, yanlışlıklar gibi şeyler olabileceği için backup alabilirsiniz. BIND bu durumun farkında olduğu için zaten bir tane backup almıştır. `nano /usr/share/doc/bind/named.conf.default` dosya yoluna giderek görebilirsiniz.


`nano /etc/named.conf` dosyasını açıyoruz ve gerekli tanımları aşağıda ki gibi yapıyoruz:

```
# 'trusted' adında bir ACL (Access Control List) oluşturuyoruz. Bu ACL, belirtilen IP aralığına izin verecektir.
acl trusted {
    172.16.1.20/24 
};

options {
        listen-on port 53 { any; }; # DNS server'ın 53 numaralı porttan dinlemesini sağlıyoruz. 'any', tüm IP adreslerine izin verir.
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; trusted;}; # Sorgu izinleri, localhost ve 'trusted' ACL'yi içerir.
        allow-transfer  { localhost;};  # Transfer izinleri, sadece localhost'a izin verilmiştir.


        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

Ortamda bir DHCP Server varsa `nano /etc/dhcp/dhcpd.conf` dosya yoluna giderek gerekli düzenlemeleri yapalım:


```
subnet 172.16.1.0 netmask 255.255.255.0 {
    range 172.16.1.150 172.16.1.200;
    option routers 172.16.1.253;
    option domain-name-servers 172.16.1.20; # DNS Serverın IP Adresi
}
```

DHCP servisini yeniden başlatıyoruz:
```
systemctl restart dhcpd
```


Bu haliyle IP Adreslerine ping atabilir fakat google.com, facebook.com gibi domainlere ping atamaz bunun için DNS Serverda Firewall ayarlarını düzenlememiz gerek:


```
[root@fedoraserver master]# firewall-cmd --add-service=dns --perm
success
[root@fedoraserver master]# firewall-cmd --reload
success
```

DNS Servisimizi çalıştıralım:

```
[root@fedoraserver master]# systemctl enable --now named
```

Şimdi Windows Makinamızda test edelim

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/bd0d98e4-a178-48d8-816b-6fcea24f62e4)

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/1ff0ebeb-9fad-4f6b-ae1e-134f281d056b)

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/cb67ced4-cf76-4ec8-a303-1db23674e053)

DNS Serverımız başarılı bir şekilde çalıştı.



## LOG

Şimdi LOG işlemlerini halledelim bunun için BIND'ın dokumanına gideceğiz. [ Buraya tıklayarak](https://kb.isc.org/docs/aa-01526) gidebilirsiniz.

`Sample Logging Configuration` kısmına gelip bir tanesini beraber inceliyoruz.

```
logging {
     channel default_log {
          file "/var/named/log/default" versions 3 size 20m; # /var/named/log diye bir dosya yolu oluşturacağız ve gerekli yetkileri vereceğiz.
          print-time yes;
          print-category yes;
          print-severity yes;
          severity info;
     };
```

Aşağıdaki komutu yazarak gerekli klasörü oluşturuyoruz ve yetkileri veriyoruz:

```
root@fedoraserver master]# mkdir -p /var/named/log
root@fedoraserver master]# chown named: /var/named/log
```


Şimdi `Sample Logging Configuration` kısmındaki tüm Configleri kopyalıyoruz ve `nano /etc/named.conf` kısmına yapıştırıyoruz

```
# LOG START

Buraya yapıştırınız.

# LOG END
```

Named Servisini yeniden başlatıyoruz:

```
systemctl restart named
```


Log dosyalarımızın geldiğini görüyoruz şimdi test edelim

```
[root@fedoraserver master]# ls -l /var/named/log
total 68
-rw-r--r--. 1 named named 45958 Dec 25 13:47 auth_servers
-rw-r--r--. 1 named named     0 Dec 25 13:25 client_security
-rw-r--r--. 1 named named     0 Dec 25 13:25 ddns
-rw-r--r--. 1 named named  1116 Dec 25 13:25 default
-rw-r--r--. 1 named named   122 Dec 25 13:25 dnssec
-rw-r--r--. 1 named named     0 Dec 25 13:25 dnstap
-rw-r--r--. 1 named named 10451 Dec 25 13:47 queries
-rw-r--r--. 1 named named     0 Dec 25 13:25 query-errors
-rw-r--r--. 1 named named     0 Dec 25 13:25 rate_limiting
-rw-r--r--. 1 named named     0 Dec 25 13:25 rpz
-rw-r--r--. 1 named named     0 Dec 25 13:25 zone_transfers
[root@fedoraserver master]#
```



Sanal Makinamızdan Ping atalım veya tarayıcıdan youtube.com adresine gidelim. Ben youtube.com adresini test edicem:

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/b0a635c5-76e4-4d0a-a92a-bfcc3c7ac750)



Terminale/konsola aşağıdaki komutu yazarak anlık olarak logları izleyebilirsiniz:

```
tail -f /var/named/log/queries
```

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/40d82b1d-26a9-4b69-ac1a-4b5933c45082)

Log işlemi başarıyla tamamlandı.


## Forward ve Reverse Lookup Zone Oluşturma

`nano /etc/named.conf` dosya yoluna giderek gerekli zone tanımlarını yapalım:

```
# ZONES
zone "sirket.local" IN {
        type master;
        file "var/named/zones/forward-sirket.local"; # Tanımlanacak dosya
        allow-update { none; };
};

zone "1.16.172.in-addr.arpa" IN { # Reverse DNS
        type master;
        file "var/named/zones/reverse-sirket.local"; # Tanımlanacak dosya
        allow-update { none; };
};
# ZONES END
```

**NOT:** Aslında ilk önce gerekli dosyaları oluşturmamız ve gerekli tanımları yapmamız daha iyi olur fakat mantığını kavrayın diye sırayla gidiyoruz.

Şimdi gerekli dosyaları ve klasörleri oluşturalım oluşturalım:

```
mkdir -p /var/named/zones
touch /var/named/zones/forward-sirket.local
touch /var/named/zones/reverse-sirket.local
```

Gerekli dosyaları ve klasörleri oluşturduktan sonra  `forward-sirket.local` dosyasını açıp gerekli yapılandırmaları yapıyoruz:

```
$ORIGIN sirket.local.
$TTL 12h

@               IN      SOA  fedora.sirket.local.  info.sirket.local. (
                             1000         ;Serial
                             1h           ;Refresh
                             30m          ;Retry
                             72h          ;Expire
                             6h           ;Minimum TTL
                               )

                IN       NS   ns1.sirket.local.
ns1             IN       A    172.16.1.20
fedora          IN       A    172.16.1.20


;;;OTHER RRs;;;

windows          IN       A      172.16.1.155
www              IN       CNAME  fedora
```
Aşağıdaki komutu yazarak check edebiliriz:
```
named-checkzone sirket.local /var/named/zones/forward-sirket.local
```

Şimdi Reverse zone'nin dosyasına gerekli yapılandırmaları yapalım:

```
$ORIGIN 1.16.172.in-addr.arpa.
$TTL 12h

@               IN              SOA     fedora.sirket.local.     info.sirket.local. (
                                        1000                    ;Serial
                                        1h                      ;Refresh
                                        30m                     ;Retry
                                        72h                     ;Expire
                                        6h                      ;Minimum TTL
                                )

                IN              NS      ns1.sirket.local.
ns1             IN              A       172.16.1.20
fedora          IN              A       172.16.1.20

;;; OTHER RRs ;;;

20              IN              PTR     fedora.sirket.local.
155             IN              PTR     windows.sirket.local.
```

Aşağıdaki komutu yazarak check edebiliriz:

```
named-checkzone 1.16.172.in-addr.arpa /var/named/zones/reverse-sirket.local
```


servisi yeniden başlatıyoruz:


```
systemctl restart named
```

Eğer bir hata çıkmadıysa devam ediyoruz.

Windows Makinamızda  `nslookup fedora.sirket.local` komutunu çalıştırıyoruz:

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/d3372858-06ed-444b-8664-ad6dda0bc820)

DNS kaydını çözdüğünü görüyoruz.



## Domain Suffix Tanımı

DHCP serverdan `nano /etc/dhcp/dhcpd.conf` dosya yoluna giderek gerekli yapılandırmaları ekleyelim:


```
subnet 172.16.1.0 netmask 255.255.255.0 {
    range 172.16.1.150 172.16.1.200;
    option routers 172.16.1.253;
    option domain-name-servers 172.16.1.20;
    option domain-search       "sirket.local"; # Arama sorgusu için domain tanımlaması
    option domain-name         "sirket.local"; # İstemcilere atanacak olan domain adı tanımlaması

}
```


Servisi `systemctl restart dhcpd` komutuyla yeniden başlatalım.



Windows makinamıza gelip sorgu atıyoruz:

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/5a0a3c03-180f-403a-94ca-af655efad195)


![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/88480144-6327-4e26-888e-53f0d9828356)


İşlemimiz başarıyla tamamlanmıştır


## DDNS

DDNS, dinamik IP adresleriyle ilişkilendirilmiş alan adlarını güncellemek için kullanılan bir sistemdir. Internet servis sağlayıcıları genellikle abonelere dinamik IP adresleri atar, bu da IP adreslerinin zamanla değişebileceği anlamına gelir. DDNS, bu değişen IP adreslerini yöneterek bir alan adını güncel tutar. Bu sayede, kullanıcılar ev ağlarına veya küçük işletme ağlarına dinamik IP adresleri yerine sabit bir alan adı üzerinden erişebilirler. Bu özellik, uzaktan erişim gerektiren senaryolarda yaygın olarak kullanılır.


Terminale gelip aşağıdaki komutu yazarak `key` oluşturuyoruz:

```
[root@fedoraserver /]# [root@fedoraserver /]# rndc-confgen -ab 512
wrote key file "/etc/rndc.key"
```

Dosyaımızı açıp içini açtığımızda `key` geldiğini görüyoruz. Algoritma olarak `SHA256` kullanılmış

```
key "rndc-key" {
        algorithm hmac-sha256;
        secret "9IIRaGTKvAsfrjPhV3Li7iGfNLiYbE5HgJJlht7Wl8KxXJCWjf98f7AJ7GtqzpkuoRpPZu+pqxSlvdhfr3AIqA==";
};
```

Şimdi forward ve reverse zone yaptığımız dosyaları symbolic link kullanacağız:

```
[root@fedoraserver named]# ln -s /var/named/zones/forward-sirket.local forward-sirket.local
[root@fedoraserver named]# ln -s /var/named/zones/reverse-sirket.local reverse-sirket.local
```

Yukarıda yapmış olduğumuz işlem /var/named/ dosya yoluna /var/named/reverse-sirket.local ve /var/named/forward-sirket.local dosyasını symbolic link yaptık.


Named'in reverse ve forward dosyalarına erişim izni var fakat /var/named/ dizinine izni yok. 

Gerekli izinleri vermek için `chown named: /var/named/zones` kodunu yazabilirsiniz.


```
[root@fedoraserver named]# chown named: /var/named/zones
[root@fedoraserver named]# ls -l /var/named/
total 764
drwxrwx---. 2 named named     23 Dec 25 12:29 data
drwxrwx---. 2 named named     60 Dec 27 17:13 dynamic
lrwxrwxrwx. 1 root  root      37 Dec 27 17:54 forward-sirket.local -> /var/named/zones/forward-sirket.local
drwxr-xr-x. 2 named named   4096 Dec 25 13:25 log
-rw-r-----. 1 root  named   3312 Nov 16 03:00 named.ca
-rw-r-----. 1 root  named      0 Nov 16 03:00 named.ca.rpmsave
-rw-r-----. 1 root  named    152 Nov 16 03:00 named.empty
-rw-r-----. 1 root  named      0 Nov 16 03:00 named.empty.rpmsave
-rw-r-----. 1 root  named    152 Nov 16 03:00 named.localhost
-rw-r-----. 1 root  named      0 Nov 16 03:00 named.localhost.rpmsave
-rw-r-----. 1 root  named    168 Nov 16 03:00 named.loopback
-rw-r-----. 1 root  named      0 Nov 16 03:00 named.loopback.rpmsave
-rw-r--r--. 1 named named 757514 Dec 27 17:36 named.run
lrwxrwxrwx. 1 root  root      37 Dec 27 17:54 reverse-sirket.local -> /var/named/zones/reverse-sirket.local
drwxrwx---. 2 named named      6 Nov 16 03:00 slaves
drwxr-xr-x. 2 named named    103 Dec 26 01:03 zones
```

Key ve link hazırlıklarımız bitti devam edelim.

Keyimizi `named.conf` dosyasına tanımlayalım:

```
zone "." IN {
        type hint;
        file "named.ca";
};

zone "sirket.local" IN {
        type master;
        file "/var/named/zones/forward-sirket.local";
        allow-update { key "rndc-key"; }; # keyimizin ismini yazıyoruz
};

zone "1.16.172.in-addr.arpa" IN {
        type master;
        file "/var/named/zones/reverse-sirket.local";
        allow-update { key "rndc-key" ;}; # keyimizin ismini yazıyoruz
};



include "/etc/rndc.key"; # dosya yolunu belirliyoruz.
```

DHCP de DDNS Config yapacağız. `nano /etc/dhcp/dhcpd.conf` dosya yoluna giderek gerekli yapılandırmaları yapalım:

```
# DDNS Update ayarları
ddns-updates on;                      # DDNS güncelleme özelliği aktif
ddns-update-style standard;           # DDNS güncelleme stili standard
ddns-domainname "sirket.local.";      # DDNS etki alanı adı
ddns-rev-domainname "in-addr.arpa.";  # DDNS ters etki alanı adı
update-static-leases on;               # Statik IP adreslerini güncelleme
update-conflict-detection off;         # Güncelleme çakışmalarını kontrol etmeyi kapat

# rndc-key konfigürasyonu
key "rndc-key" {
    algorithm hmac-sha256;             # Kullanılan algoritma
    secret "9IIRaGTKvAsfrjPhV3Li7iGfNLiYbE5HgJJlht7Wl8KxXJCWjf98f7AJ7GtqzpkuoRpPZu+pqxSlvdhfr3AIqA==";  # HMAC anahtarı
};

# sirket.local etki alanı konfigürasyonu
zone sirket.local. {
    primary 172.16.1.20;                # DDNS güncelleme için başlangıç sunucusu
    key "rndc-key";                     # Kullanılan rndc anahtarı
}

# Ters etki alanı konfigürasyonu
zone 1.16.172.in-addr.arpa. {
    primary 172.16.1.20;                # DDNS güncelleme için başlangıç sunucusu
    key "rndc-key";                     # Kullanılan rndc anahtarı
}

# rndc.key dosyasını dahil et
include "/etc/rndc.key";
```

### DDNS Update Sürecinin Takibi


Yaptığımız işlemlerden sonra servislerimizi restart edelim

```
systemctl restart dhcpd
```
```
systemctl restart named
```

Eğer bir hata almadıysanız devam edelim.


3 Farklı Terminal açarak bu işlemi daha rahat yapabilirsiniz.


1. Terminalde `/var/log/dhcp.log` dosyasına bakacağız:

```
tail -f /var/log/dhcp.log
```

2. Terminalde `DDNS` loglarına bakacağız:


```
tail -f /var/named/log/ddns -n 30 # Şuanlık 30 tane bizim için yeterlidir
```

Şimdi yeni bir `client` ortama sokalım. Eski kullanmış olduğumuz Windows Sanal Makinasından Clone oluşturarak bunu sağlayabilirsiniz. 

Loglarda oluşturmuş olduğumuz bilgisayarın IP adresini görebiliyoruz.

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/4135e4b9-3e14-401c-b495-deac8e6b122f)


Bu işlemden sonra `ls -l /var/named/zones/` komutunu yazarak oluşmuş olan **.jnl** uzantılı dosyaları görebilirsiniz. İçeriği bizim okuyabileceğimiz bir türden değil.

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/90ce29a6-b4df-494d-ab99-1536385a9cc3)


Terminale `rndc sync -clean` komutunu yazarak güncelleyelim.

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/635be424-02a1-4f69-bf63-f50f6174e050)


Gördüğünüz gibi hiçbir kayıt girmeden otomatik bir şekilde gelmiş bulunmaktadır.


Şimdi şöyle bir test yapalım. DHCP Serverda bu bilgisayara özel bir IP adresi tanımlayalım ve bakalım onu tanımlayacak mı?

**Rezervasyon işlemi:**

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/c4a51f65-a365-453b-9854-8a755326e04d)


`systemctl restart dhcpd ` komutunu yazarak DHCP servisini yeniden başlatalım.

Windows bilgisayarımızda `ipconfig /release` ve `ipconfig /renew` komutlarını sırasıyla yazalım.


Bilgisayarımız `172.16.1.159` IP adresini almış bulunmakta.

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/d5020b45-f13c-4f0f-a08f-4b34e7eb2092)


Servera dönüp tekrardan `rndc sync -clean` komutunu yazınız ve ardından `nano /var/named/zones/forward-sirket.local` dosyasını açınız.

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/1b14f6af-a69e-44ef-80be-ad301aec5315)



İşlemlerimiz başarıyla tamamlandı.



## Root Hits Güncelleme ve Forwarders Belirleme



`nano /var/named/named.ca` DNS sunucusunun kök DNS sunucularının bilgilerini içeren bir dosyadır. Burayı güncelleyeceğiz.

Terminale aşağıdaki komutu giriniz:

```
wget ftp://ftp.rs.internic.net/domain/db.cache -O /var/named/named.ca && rndc reload
```

![image](https://github.com/ugurcomptech/Fedora-DNS-Server/assets/133202238/6b854350-ccca-4d64-92f2-28ca493485e8)

Güncelleme tamamlandı içerisini açıp kontrol edebilirsiniz.


Şimdi forwarders kısmını güncelleyelim. Terminalden `nano /etc/named.conf` açınız ve options kısmına aşağıdaki kodu giriniz:

```
        forwarders {
                1.1.1.1;
                8.8.8.8;
        };
```

Eğer DNS'lerimiz çalışmaz ise gelip buradaki adreslere soracak, bunlarda çalışmaz veya bulamaz ise diğer DNS'lerin yaptığı gibi başka kök DNS'lere soracaktır.


DNS Server yapılandırmamız bu kadardı, okuduğunuz için teşekkürler.



















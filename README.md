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
$ORIGIN 1.16.172.in-addr-arpa.
$TTL 12h

@               IN              SOA     fedora.sirket.local     info.sirket.local. (
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














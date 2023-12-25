# Fedora-DNS-Server

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

Şimdi LOG işlemlerini halledelim bunun için BIND'ın dokumanına gideceğiz ![Tıklayarak gidebilirsiniz](https://kb.isc.org/docs/aa-01526)




















# Cisco Packet Tracer: Dinamik NAT (Dynamic NAT) Yapılandırması ve Yönlendirme Çözümü

Bu proje, bir yerel ağdaki (LAN) cihazların dış ağa (İnternet/WAN) çıkarken belirli bir IP havuzundan (Pool) dinamik olarak IP adresi almasını sağlayan Dinamik NAT yapılandırmasını içermektedir. 

Aynı zamanda, Dinamik NAT havuzu için dış yönlendiricide (ISP) eksik olan geri dönüş rotasının (return route) nasıl tespit edilip çözüldüğü de adım adım belgelenmiştir.

## 📌 Topoloji Özeti
* **İç Ağ (LAN):** 192.168.1.0/24 
* **WAN Bağlantısı (R1 - ISP):** 209.165.201.16/30
* **Dinamik NAT Havuzu:** 200.200.200.10 - 200.200.200.15
* **Hedef Ağ (Web Server):** 192.31.7.0/24

## ⚙️ Bölüm 1: Eski Konfigürasyonun Temizlenmesi
Yeni yapılandırmaya geçmeden önce, R1 yönlendiricisindeki aktif çeviriler temizlendi ve eski Statik NAT kuralı kaldırıldı:
```bash
R1# clear ip nat translation *
R1# configure terminal
R1(config)# no ip nat inside source static 192.168.1.2 209.165.201.18
```

🚀 Bölüm 2: Dinamik NAT Yapılandırması (R1 Router)

Yerel ağdaki (192.168.1.0/24) cihazların, oluşturulan yeni IP havuzunu (YENIPOOL) kullanabilmesi için gerekli ACL (Erişim Kontrol Listesi) ve NAT komutları girildi:

```text
! 1. Adım: Arayüzlerin NAT yönlerinin belirlenmesi (Daha önceden yapılmıştı)
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip nat inside
R1(config-if)# exit

R1(config)# interface serial 0/1/0
R1(config-if)# ip nat outside
R1(config-if)# exit
```

! 2. Adım: Hangi iç IP'lerin NAT'a gireceğini belirleyen ACL'in yazılması
```text
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255
```

! 3. Adım: Çeviri için kullanılacak dış IP havuzunun (Pool) oluşturulması
```text
R1(config)# ip nat pool YENIPOOL 200.200.200.10 200.200.200.15 netmask 255.255.255.0
```


! 4. Adım: ACL ile IP Havuzunun birbirine bağlanması (Dinamik NAT Kuralı)
```text
R1(config)# ip nat inside source list 1 pool YENIPOOL
```

🛠️ Bölüm 3: Troubleshooting - Geri Dönüş Rotası (Return Route) Problemi

Sorun: Dinamik NAT kuralı doğru yazılmasına rağmen, LAN cihazlarından Web Server'a atılan pingler Request timed out hatası verdi.
Kök Neden: Hedef sunucu paketleri başarıyla aldı ve Dinamik NAT havuzundaki IP'ye (örn: 200.200.200.10) cevap döndü. Ancak ISP router'ı, 200.200.200.0/24 ağının nerede olduğunu (R1'in arkasında olduğunu) bilmediği için paketleri düşürdü.
Çözüm: ISP router'ına, NAT havuzunu işaret eden statik bir geri dönüş rotası eklendi.
```text
ISP(config)# ip route 200.200.200.0 255.255.255.0 209.165.201.18
```

✅ Bölüm 4: Doğrulama (Verification)

ISP'ye rota eklendikten sonra PC0 ve PC1'den eşzamanlı ping testleri başlatıldı. R1 üzerinde show ip nat translations komutu ile havuzdaki IP'lerin cihazlara sırayla ve dinamik olarak atandığı doğrulandı:
```text
R1# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
icmp 200.200.200.10:1   192.168.1.2:1      192.31.7.100:1     192.31.7.100:1
icmp 200.200.200.11:1   192.168.1.3:1      192.31.7.100:1     192.31.7.100:1
```

# Cisco Packet Tracer: Static NAT & Routing Troubleshooting Lab

Bu proje, temel bir ağ topolojisinde Statik NAT (Network Address Translation) yapılandırmasını ve karşılaşılan yönlendirme (routing) hatalarının adım adım nasıl çözüldüğünü (troubleshooting) içermektedir. 


## 📌 Topoloji Özeti
* **İç Ağ (LAN):** 192.168.1.0/24 
* **WAN Bağlantısı (R1 - ISP):** 209.165.201.16/30 (Serial Link)
* **Hedef Ağ (Web Server):** 192.31.7.0/24
* **Kullanılan Cihazlar:** Cisco 1941 Router'lar, 2960 Switch, PC'ler ve Server.

## 🛠️ Bölüm 1: Troubleshooting (Sorun Giderme) Süreci
**Sorun:** İç ağdaki PC1'den (192.168.1.3), dış ağdaki Web Server'a (192.31.7.100) atılan ping istekleri `Request timed out` hatası verdi.

**Çözüm Adımları:**
1. **Layer 1/2 Kontrolü:** R1 ve ISP arasındaki Seri bağlantıların fiziksel olarak kapalı olduğu fark edildi. Arayüzlere girilerek `no shutdown` komutu ile hat ayağa kaldırıldı. (R1 tarafında DCE clock rate ayarlandı).
2. **Routing Analizi:** ISP router'ının yönlendirme tablosu (`show ip route`) incelendiğinde, LAN ağına geri dönüş rotasının eksik olduğu tespit edildi.
3. **Kök Neden (Root Cause):** ISP üzerindeki `show running-config` incelendiğinde, Next-Hop IP adresinde bir yazım hatası yapıldığı fark edildi:
   * *Hatalı girilen:* `ip route 192.168.1.0 255.255.255.0 209.15.201.18`
   * *Olması gereken:* `ip route 192.168.1.0 255.255.255.0 209.165.201.18` 
4. **Çözüm:** Hatalı rota silindi, doğrusu eklendi ve uçtan uca ping iletişimi başarıyla sağlandı.

## ⚙️ Bölüm 2: Statik NAT Yapılandırması
İç ağdaki PC0'ın (192.168.1.2) internete çıkabilmesi için R1 üzerinde Statik NAT yapılandırıldı.

```bash
R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip nat inside
R1(config)# interface serial 0/1/0
R1(config-if)# ip nat outside
R1(config)# ip nat inside source static 192.168.1.2 209.165.201.18
```


✅ Canlı NAT Çevirisi Doğrulaması

Packet Tracer'da NAT çevirilerini canlı görebilmek için PC0 üzerinden Web Servera peş peşe paketler gönderildi (ping -n 100 192.31.7.100) ve anlık olarak R1 üzerinde tablonun dolduğu doğrulandı:

```text
R1# show ip nat translations 
Pro  Inside global     Inside local       Outside local      Outside global
icmp 209.165.201.18:10 192.168.1.2:10     192.31.7.100:10    192.31.7.100:10
icmp 209.165.201.18:11 192.168.1.2:11     192.31.7.100:11    192.31.7.100:11
icmp 209.165.201.18:12 192.168.1.2:12     192.31.7.100:12    192.31.7.100:12
...
---  209.165.201.18    192.168.1.2        ---                ---
```

Sonuç: PC0'ın yerel IP adresi (192.168.1.2), başarılı bir şekilde R1'in dış bacak IP adresine (209.165.201.18) çevrilerek hedef sunucuya ulaştı.

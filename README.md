# IPsec

# Лабораторная работа: IPsec

# Задание:
1. Настроите GRE поверх IPSec между офисами Москва и С.-Петербург
2. Настроите DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги

Схема сегмента сети:
![](https://github.com/dmitriyklimenkov/DMVPN/blob/main/%D0%A1%D1%85%D0%B5%D0%BC%D0%B0%20%D1%81%D0%B5%D0%B3%D0%BC%D0%B5%D0%BD%D1%82%D0%B0.PNG)

# 1. Настроим GRE поверх IPSec между офисами Москва и С.-Петербург.

Настроим туннели, isakmp, transform set, IPSec-profile и trustpoint.
Сеть туннеля - 10.10.10.0/24.
Внешний IP адрес офиса Москва 194.14.123.5, адрес туннеля 10.10.10.1.
Внешний IP адрес офиса Москва 194.14.123.9, адрес туннеля 10.10.10.2.

Конфигурация R15:
```
!
interface Tunnel1
 ip address 10.10.10.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 194.14.123.5
 tunnel destination 194.14.123.9
 tunnel protection ipsec profile GRE_P
end
!
crypto pki server CA
!
crypto pki trustpoint CA
 revocation-check crl
 rsakeypair CA
!
crypto pki trustpoint VPN
 enrollment url http://10.10.10.1:80
 serial-number
 ip-address 10.10.10.2
 subject-name CN=R4,OU=VPN,O=Server,C=RU
 revocation-check none
 rsakeypair VPN
!
crypto pki trustpoint R15_CA
 enrollment url http://10.10.10.1:80
 serial-number
 ip-address 10.10.10.1
 subject-name CN=R4,OU=VPN,O=Server,C=RU
 revocation-check none
 rsakeypair R15_CA
!
!
crypto isakmp policy 5
 encr 3des
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key cisco address 194.14.123.9
crypto isakmp key cisco address 0.0.0.0
!
!
crypto ipsec transform-set SET1 esp-3des
 mode tunnel
 
crypto ipsec profile GRE_P
 set transform-set SET1
!
!
!
crypto map MAP 5 ipsec-isakmp
 set peer 194.14.123.9
 set transform-set SET1
 match address 105
!
access-list 105 permit gre host 194.14.123.5 host 194.14.123.9
```

Конфигурация R18:
```
!
interface Tunnel1
 ip address 10.10.10.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 194.14.123.9
 tunnel destination 194.14.123.5
 tunnel protection ipsec profile GRE_P
end
crypto pki trustpoint VPN
 enrollment url http://194.14.123.5:80
 serial-number
 ip-address 10.10.10.2
 subject-name CN=R4,OU=VPN,O=Server,C=RU
 revocation-check none
 rsakeypair VPN

!
!
crypto pki certificate chain VPN
!
redundancy
!
!
!
!
!
!
!
crypto isakmp policy 5
 encr 3des
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key cisco address 194.14.123.5
!
!
crypto ipsec transform-set SET1 esp-3des
 mode tunnel
!
crypto ipsec profile GRE_P
 set transform-set SET1
!
!
!
crypto map MAP 5 ipsec-isakmp
 set peer 194.14.123.5
 set transform-set SET1
 match address 105
!
access-list 105 permit gre host 194.14.123.9 host 194.14.123.5
```

Проверим на R18:
```
R18#sh crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
194.14.123.5    194.14.123.9    QM_IDLE           1004 ACTIVE
```

# 2. Настроим DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги.

Конфигурация R15:
```
!
interface Tunnel2
 ip address 10.10.20.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp map multicast dynamic
 ip nhrp network-id 50
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 255
 tunnel source Ethernet0/2
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
!
!
crypto ipsec profile DMVPN
 set transform-set SET2
!
crypto ipsec transform-set SET2 esp-3des
 mode transport

crypto pki trustpoint VPN1
 enrollment url http://10.10.20.1:80
 serial-number
 ip-address 10.10.20.3
 subject-name CN=R4,OU=VPN,O=Server,C=RU
 revocation-check none
 rsakeypair VPN
!
!
crypto isakmp policy 5
 encr 3des
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key cisco address 194.14.123.9
crypto isakmp key cisco address 0.0.0.0
!
!
crypto pki server CA
!
crypto pki trustpoint CA
 revocation-check crl
 rsakeypair CA
!
```
Конфигурация R28:
```
!
interface Tunnel2
 ip address 10.10.20.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp map multicast 194.14.123.5
 ip nhrp map 10.10.20.1 194.14.123.5
 ip nhrp network-id 50
 ip nhrp nhs 10.10.20.1
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
!
crypto pki trustpoint VPN1
 enrollment url http://194.14.123.5:80
 serial-number
 ip-address 10.10.20.2
 subject-name CN=R4,OU=VPN,O=Server,C=RU
 revocation-check none
 rsakeypair VPN1
!
crypto isakmp policy 5
 encr 3des
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key cisco address 0.0.0.0
!
!
crypto ipsec transform-set SET2 esp-3des
 mode transport
 !
crypto ipsec profile DMVPN
 set transform-set SET2

```

Конфигурация R27:
```
!
interface Tunnel2
 ip address 10.10.20.3 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp map multicast 194.14.123.5
 ip nhrp map 10.10.20.1 194.14.123.5
 ip nhrp network-id 50
 ip nhrp nhs 10.10.20.1
 ip tcp adjust-mss 1360
 ip ospf network broadcast
 ip ospf priority 0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
!
!
crypto isakmp policy 5
 encr 3des
 hash sha256
 authentication pre-share
 group 19
crypto isakmp key cisco address 0.0.0.0
!
!
crypto ipsec transform-set SET2 esp-3des
 mode transport
!
crypto ipsec profile DMVPN
 set transform-set SET2
```

Проверим на R15:
```
R15#sh crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
194.14.123.5    194.14.123.9    QM_IDLE           1018 ACTIVE
194.14.123.34   194.14.123.5    QM_IDLE           1016 ACTIVE
194.14.123.26   194.14.123.5    QM_IDLE           1017 ACTIVE
```
Проверим на R15 выданные серфитикаты:
```
R15#sh crypto pki server CA requests
Enrollment Request Database:

Subordinate CA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

RA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
6      pending    42768EFAAEEC24AE271544FFF95085B2 serialNumber=67109296+hostname=R27+ipaddress=10.10.20.3,cn=R4,ou=VPN,o=Server,c=RU
5      pending    2C0CACE7D3F299ED6DC77D4B56EFC700 serialNumber=67109312+hostname=R28+ipaddress=10.10.20.2,cn=R4,ou=VPN,o=Server,c=RU
4      pending    551A1CA15D0DF6E8A9E516683A10B1D0 serialNumber=67109152+hostname=R18+ipaddress=10.10.10.2,cn=R4,ou=VPN,o=Server,c=RU
3      pending    4CD2CD150BBBF5C44A42D74B7436E1F4 serialNumber=67109104+ipaddress=10.10.10.1+hostname=R15.server.ru,cn=R4,ou=VPN,o=Server,c=RU
2      pending    0AA4B6FD745F7FB1BC76DDD1F8BA892D serialNumber=67109104+ipaddress=10.10.10.2+hostname=R15.server.ru,cn=R4,ou=VPN,o=Server,c=RU
```
Проверим сертификат на R28:
```
R28#sh crypto pki certificates
CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=CA
  Subject:
    cn=CA
  Validity Date:
    start date: 20:15:59 EET Feb 25 2021
    end   date: 20:15:59 EET Feb 25 2024
  Associated Trustpoints: VPN1


Certificate
  Subject:
    Name: R28
    IP Address: 10.10.20.2
    Serial Number: 67109312
   Status: Pending
   Key Usage: General Purpose
   Certificate Request Fingerprint MD5: 2C0CACE7 D3F299ED 6DC77D4B 56EFC700
   Certificate Request Fingerprint SHA1: 31DC7289 631C069D B24FACDB 364D6127 05473DAE
   Associated Trustpoint: VPN1
```

Проверим связность:

Запустим пинг с VPC7 из Москвы до VPC30 Чокурдах:
```
VPCS> ping 201.193.45.2

84 bytes from 201.193.45.2 icmp_seq=1 ttl=57 time=3.274 ms
84 bytes from 201.193.45.2 icmp_seq=2 ttl=57 time=4.757 ms
84 bytes from 201.193.45.2 icmp_seq=3 ttl=57 time=3.231 ms
84 bytes from 201.193.45.2 icmp_seq=4 ttl=57 time=2.570 ms
```

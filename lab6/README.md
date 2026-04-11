
# Лабораторная работа: BGP. Основы

## **Тема работы**
Настройка eBGP между автономными системами. Организация IP-доступности между офисами Москва и С.-Петербург.

## **Цель**
Настроить BGP между автономными системами и организовать доступность между пограничными роутерами офисов Москва и С.-Петербург.

## **Описание/Пошаговая инструкция выполнения домашнего задания**

1. Настроить eBGP между офисом Москва (R14, R15) и провайдерами Киторн (R22) и Ламас (R21)
2. Настроить eBGP между провайдерами Киторн (R22) и Ламас (R21)
3. Настроить eBGP между Ламас (R21) и Триада (R24)
4. Настроить eBGP между офисом С.-Петербург (R18) и провайдером Триада (R24, R26)
5. Организовать IP-доступность между пограничными роутерами офисов Москва и С.-Петербург

---

## **Общая топология сети**
![Топология сети](topology/lab.png)

## **Топология сети лабораторной №5**
![Топология сети для lab2](topology/lab-6.png)

### **Участники и их роли**

| Устройство | Роль | Автономная система |
|------------|------|---------------------|
| R14, R15 | Пограничные роутеры офиса Москва | AS 1001 |
| R22 | Провайдер Киторн | AS 101 |
| R21 | Провайдер Ламас | AS 301 |
| R23, R24, R26 | Провайдер Триада | AS 520 |
| R18 | Пограничный роутер офиса С.-Петербург | AS 2042 |

### **BGP-пиринг (eBGP стыки)**

| Роутер 1 | AS 1 | Интерфейс 1 | IP 1 | Сеть | IP 2 | Интерфейс 2 | AS 2 | Роутер 2 |
|:---|:---:|:---|:---|:---|:---|:---|:---:|:---|
| R14 | 1001 | Eth0/2 | 192.168.98.42 | **192.168.98.40/30** | 192.168.98.41 | Eth0/0 | 101 | R22 |
| R15 | 1001 | Eth0/2 | 192.168.98.45 | **192.168.98.44/30** | 192.168.98.46 | Eth0/0 | 301 | R21 |
| R22 | 101 | Eth0/1 | 192.168.98.50 | **192.168.98.48/30** | 192.168.98.49 | Eth0/1 | 301 | R21 |
| R22 | 101 | Eth0/2 | 192.168.98.57 | **192.168.98.56/30** | 192.168.98.58 | Eth0/0 | 520 | R23 |
| R21 | 301 | Eth0/2 | 192.168.98.53 | **192.168.98.52/30** | 192.168.98.54 | Eth0/0 | 520 | R24 |
| R24 | 520 | Eth0/3 | 192.168.98.74 | **192.168.98.72/30** | 192.168.98.73 | Eth0/2 | 2042 | R18 |
| R26 | 520 | Eth0/3 | 192.168.98.78 | **192.168.98.76/30** | 192.168.98.77 | Eth0/3 | 2042 | R18 |

---

## **План работы и реализация**

### **1. Настройка eBGP на R14 и R15 (Москва, AS 1001)**

#### **R14 (стык с Киторн AS 101)**

```cisco
router bgp 1001
 bgp router-id 14.14.14.14
 bgp log-neighbor-changes
 neighbor 192.168.98.41 remote-as 101
 neighbor 192.168.98.41 description eBGP_to_Kitorn_R22
 !
 address-family ipv4
  network 192.168.98.40 mask 255.255.255.252
  neighbor 192.168.98.41 activate
 exit-address-family
```

#### **R15 (стык с Ламас AS 301)**

```cisco
router bgp 1001
 bgp router-id 15.15.15.15
 bgp log-neighbor-changes
 neighbor 192.168.98.46 remote-as 301
 neighbor 192.168.98.46 description eBGP_to_Lamas_R21
 !
 address-family ipv4
  network 192.168.98.44 mask 255.255.255.252
  neighbor 192.168.98.46 activate
 exit-address-family
```

---

### **2. Настройка eBGP на R22 (Киторн, AS 101)**

```cisco
router bgp 101
 bgp router-id 22.22.22.22
 bgp log-neighbor-changes
 neighbor 192.168.98.42 remote-as 1001
 neighbor 192.168.98.42 description eBGP_to_Moscow_R14
 neighbor 192.168.98.49 remote-as 301
 neighbor 192.168.98.49 description eBGP_to_Lamas_R21
 neighbor 192.168.98.58 remote-as 520
 neighbor 192.168.98.58 description eBGP_to_Triada_R23
 !
 address-family ipv4
  network 192.168.98.40 mask 255.255.255.252
  network 192.168.98.48 mask 255.255.255.252
  network 192.168.98.56 mask 255.255.255.252
  neighbor 192.168.98.42 activate
  neighbor 192.168.98.49 activate
  neighbor 192.168.98.58 activate
 exit-address-family
```

---

### **3. Настройка eBGP на R21 (Ламас, AS 301)**

```cisco
router bgp 301
 bgp router-id 21.21.21.21
 bgp log-neighbor-changes
 neighbor 192.168.98.45 remote-as 1001
 neighbor 192.168.98.45 description eBGP_to_Moscow_R15
 neighbor 192.168.98.50 remote-as 101
 neighbor 192.168.98.50 description eBGP_to_Kitorn_R22
 neighbor 192.168.98.54 remote-as 520
 neighbor 192.168.98.54 description eBGP_to_Triada_R24
 !
 address-family ipv4
  network 192.168.98.44 mask 255.255.255.252
  network 192.168.98.48 mask 255.255.255.252
  network 192.168.98.52 mask 255.255.255.252
  neighbor 192.168.98.45 activate
  neighbor 192.168.98.50 activate
  neighbor 192.168.98.54 activate
 exit-address-family
```

---

### **4. Настройка eBGP на R23, R24, R26 (Триада, AS 520)**

#### **R23 (стык с Киторн)**

```cisco
router bgp 520
 bgp router-id 23.23.23.23
 bgp log-neighbor-changes
 neighbor 192.168.98.57 remote-as 101
 neighbor 192.168.98.57 description eBGP_to_Kitorn_R22
 !
 address-family ipv4
  network 192.168.98.56 mask 255.255.255.252
  neighbor 192.168.98.57 activate
 exit-address-family
```

#### **R24 (стык с Ламас и С.-Петербург)**

```cisco
router bgp 520
 bgp router-id 24.24.24.24
 bgp log-neighbor-changes
 neighbor 192.168.98.53 remote-as 301
 neighbor 192.168.98.53 description eBGP_to_Lamas_R21
 neighbor 192.168.98.73 remote-as 2042
 neighbor 192.168.98.73 description eBGP_to_Piter_R18
 !
 address-family ipv4
  network 192.168.98.52 mask 255.255.255.252
  network 192.168.98.72 mask 255.255.255.252
  neighbor 192.168.98.53 activate
  neighbor 192.168.98.73 activate
 exit-address-family
```

#### **R26 (стык с С.-Петербург)**

```cisco
router bgp 520
 bgp router-id 26.26.26.26
 bgp log-neighbor-changes
 neighbor 192.168.98.77 remote-as 2042
 neighbor 192.168.98.77 description eBGP_to_Piter_R18
 !
 address-family ipv4
  network 192.168.98.76 mask 255.255.255.252
  neighbor 192.168.98.77 activate
 exit-address-family
```

---

### **5. Настройка eBGP на R18 (С.-Петербург, AS 2042)**

```cisco
router bgp 2042
 bgp router-id 18.18.18.18
 bgp log-neighbor-changes
 neighbor 192.168.98.74 remote-as 520
 neighbor 192.168.98.74 description eBGP_to_Triada_R24
 neighbor 192.168.98.78 remote-as 520
 neighbor 192.168.98.78 description eBGP_to_Triada_R26
 !
 address-family ipv4
  network 192.168.98.72 mask 255.255.255.252
  network 192.168.98.76 mask 255.255.255.252
  neighbor 192.168.98.74 activate
  neighbor 192.168.98.78 activate
 exit-address-family
```

---

## **Тестирование и проверка**

### **1. Проверка BGP-соседств**

#### **R14**

```
R14#show ip bgp summary
BGP router identifier 14.14.14.14, local AS number 1001

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.41   4          101      91      91        8    0    0 01:18:21        7
```

#### **R15**

```
R15#show ip bgp summary
BGP router identifier 15.15.15.15, local AS number 1001

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.46   4          301      88      86        8    0    0 01:15:08        7
```

#### **R22**

```
R22#show ip bgp summary
BGP router identifier 22.22.22.22, local AS number 101

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.42   4         1001      88      88        8    0    0 01:15:31        1
192.168.98.49   4          301      88      87        8    0    0 01:15:27        5
192.168.98.58   4          520      66      70        8    0    0 00:56:51        1
```

#### **R21**

```
R21#show ip bgp summary
BGP router identifier 21.21.21.21, local AS number 301

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.45   4         1001      86      88        8    0    0 01:15:40        1
192.168.98.50   4          101      87      88        8    0    0 01:15:39        3
192.168.98.54   4          520      79      80        8    0    0 01:05:45        3
```

#### **R24**

```
R24#show ip bgp summary
BGP router identifier 24.24.24.24, local AS number 520

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.53   4          301      80      79        8    0    0 01:06:10        5
192.168.98.73   4         2042      77      79        8    0    0 01:06:00        2
```

#### **R18**

```
R18#show ip bgp summary
BGP router identifier 18.18.18.18, local AS number 2042

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.98.74   4          520      80      78        8    0    0 01:06:24        6
192.168.98.78   4          520      74      79        8    0    0 01:03:50        1
```

**✅ Результат:** Все eBGP соседства установлены (State = число полученных префиксов).

---

### **2. Проверка IP-доступности между офисами**

#### **С R18 (С.-Петербург) до R14 и R15 (Москва)**

```
R18#ping 192.168.98.42
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.98.42, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/5 ms

R18#ping 192.168.98.45
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.98.45, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/4 ms
```

#### **С R14 (Москва) до R18 (С.-Петербург)**

```
R14#ping 192.168.98.73
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.98.73, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/4 ms

R14#ping 192.168.98.77
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.98.77, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/6 ms
```

#### **С R15 (Москва) до R18 (С.-Петербург)**

```
R15#ping 192.168.98.73
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.98.73, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/3 ms
```

**✅ Результат:** IP-доступность между пограничными роутерами Москвы и С.-Петербурга обеспечена.

---

### **3. Трассировка маршрутов**

#### **R18 → R14 (через R24 → R21 → R22)**

```
R18#traceroute 192.168.98.42
Tracing the route to 192.168.98.42
  1 192.168.98.74 2 msec 1 msec 1 msec
  2 192.168.98.53 [AS 520] 3 msec 2 msec 3 msec
  3 192.168.98.50 [AS 301] 3 msec 3 msec 4 msec
  4 192.168.98.42 [AS 101] 4 msec *  3 msec
```

**Путь:** R18 → R24 (AS 520) → R21 (AS 301) → R22 (AS 101) → R14

#### **R18 → R15 (через R24 → R21 напрямую)**

```
R18#traceroute 192.168.98.45
Tracing the route to 192.168.98.45
  1 192.168.98.74 2 msec 2 msec 1 msec
  2 192.168.98.53 [AS 520] 2 msec 3 msec 3 msec
  3 192.168.98.45 [AS 301] 4 msec *  9 msec
```

**Путь:** R18 → R24 (AS 520) → R21 (AS 301) → R15

#### **R14 → R18 (через R22 → R21 → R24)**

```
R14#traceroute 192.168.98.73
Tracing the route to 192.168.98.73
  1 192.168.98.41 2 msec 1 msec 1 msec
  2 192.168.98.49 [AS 101] 2 msec 2 msec 3 msec
  3 192.168.98.54 [AS 301] 4 msec 5 msec 4 msec
  4 192.168.98.73 [AS 520] 6 msec *  6 msec
```

**Путь:** R14 → R22 (AS 101) → R21 (AS 301) → R24 (AS 520) → R18

#### **R15 → R18 (через R21 → R24)**

```
R15#traceroute 192.168.98.73
Tracing the route to 192.168.98.73
  1 192.168.98.46 1 msec 5 msec 1 msec
  2 192.168.98.54 [AS 301] 2 msec 2 msec 2 msec
  3 192.168.98.73 [AS 520] 13 msec *  5 msec
```

**Путь:** R15 → R21 (AS 301) → R24 (AS 520) → R18

**✅ Результат:** Маршруты проходят через провайдеров в соответствии с топологией.

---

### **4. Сводная таблица анонсируемых P2P-сетей**

| Роутер | Анонсируемые сети в BGP |
|:---|:---|
| **R14** | 192.168.98.40/30 |
| **R15** | 192.168.98.44/30 |
| **R22** | 192.168.98.40/30, 192.168.98.48/30, 192.168.98.56/30 |
| **R21** | 192.168.98.44/30, 192.168.98.48/30, 192.168.98.52/30 |
| **R23** | 192.168.98.56/30 |
| **R24** | 192.168.98.52/30, 192.168.98.72/30 |
| **R26** | 192.168.98.76/30 |
| **R18** | 192.168.98.72/30, 192.168.98.76/30 |

---

## **Выводы**

В ходе лабораторной работы были выполнены следующие задачи:

1. ✅ **eBGP между офисом Москва и провайдерами настроен** — R14 ↔ R22 (AS 101), R15 ↔ R21 (AS 301)
2. ✅ **eBGP между провайдерами Киторн и Ламас настроен** — R22 ↔ R21
3. ✅ **eBGP между Ламас и Триада настроен** — R21 ↔ R24
4. ✅ **eBGP между офисом С.-Петербург и Триада настроен** — R18 ↔ R24, R18 ↔ R26
5. ✅ **IP-доступность между пограничными роутерами обеспечена** — успешные пинги R14/R15 ↔ R18
6. ✅ **Маршруты проходят через провайдеров** согласно топологии
7. ✅ **BGP-таблицы содержат все необходимые префиксы** P2P-сетей



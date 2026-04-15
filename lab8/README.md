# Лабораторная работа: BGP. Фильтрация

## **Тема работы**
Настройка фильтрации BGP для офисов Москва и С.-Петербург с целью предотвращения транзитного трафика и ограничения анонсов от провайдеров.

## **Цель**
1. Настроить фильтрацию в офисе Москва (AS-Path) для запрета транзитного трафика
2. Настроить фильтрацию в офисе С.-Петербург (Prefix-list) для запрета транзитного трафика
3. Настроить провайдера Киторн (AS 101) для передачи в Москву только маршрута по умолчанию
4. Настроить провайдера Ламас (AS 301) для передачи в Москву маршрута по умолчанию и сетей С.-Петербурга
5. Обеспечить полную IP-связность всех клиентских сетей

---

## **Общая топология сети**
![Топология сети](topology/lab.png)

## **Топология сети лабораторной №8**
![Топология сети для lab2](topology/lab-6.png)

### **Участники и их роли**

| Устройство | Роль | Автономная система |
|:---|:---|:---:|
| R14, R15 | Пограничные роутеры офиса Москва | AS 1001 |
| VPC1, VPC7 | Клиенты Москвы (10.10.10.0/24, 10.10.20.0/24) | — |
| R22 | Провайдер Киторн | AS 101 |
| R21 | Провайдер Ламас | AS 301 |
| R23, R24, R25, R26 | Провайдер Триада | AS 520 |
| R18 | Пограничный роутер офиса С.-Петербург | AS 2042 |
| VPCS, VPC8 | Клиенты С.-Петербурга (10.20.10.0/24, 10.20.20.0/24) | — |

---

## **План работы и реализация**

### **1. Фильтрация в офисе Москва (AS-Path)**

**Задача:** Запретить транзитный трафик через AS 1001. Москва не должна анонсировать провайдерам маршруты, полученные от другого провайдера.

#### **1.1. Конфигурация AS-Path фильтра**

**R14:**
```cisco
ip as-path access-list 10 permit ^$
ip as-path access-list 10 permit ^1001$
!
router bgp 1001
 address-family ipv4
  neighbor 192.168.98.41 filter-list 10 out
```

**R15:**
```cisco
ip as-path access-list 10 permit ^$
ip as-path access-list 10 permit ^1001$
!
router bgp 1001
 address-family ipv4
  neighbor 192.168.98.46 filter-list 10 out
```

**Пояснение:**
- `^$` — разрешает маршруты с пустым AS-Path (локальные)
- `^1001$` — разрешает маршруты, где AS-Path состоит только из AS 1001

#### **1.2. Проверка фильтрации**

**R14:**
```
R14# show ip as-path-access-list
AS path access list 10
    permit ^$
    permit ^1001$

R14# show ip bgp neighbors 192.168.98.41 advertised-routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.10.10.0/24    192.168.98.17           21         32768 i
 *>  10.10.20.0/24    192.168.98.17           21         32768 i
 *>  192.168.98.40/30 0.0.0.0                  0         32768 i
 r>i 192.168.98.44/30 15.15.15.15              0    200      0 i
```

**R15:**
```
R15# show ip bgp neighbors 192.168.98.46 advertised-routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.10.10.0/24    192.168.98.25           21         32768 i
 *>  10.10.20.0/24    192.168.98.25           21         32768 i
 r>i 192.168.98.40/30 14.14.14.14              0    100      0 i
 *>  192.168.98.44/30 0.0.0.0                  0         32768 i
```

✅ Маршруты с `r>i` (RIB-failure) **не анонсируются** провайдерам — транзитный трафик заблокирован.

---

### **2. Фильтрация в офисе С.-Петербург (Prefix-list)**

**Задача:** R18 должен анонсировать в Триаду только свои локальные сети.

#### **2.1. Конфигурация Prefix-list**

**R18:**
```cisco
ip prefix-list LOCAL_PITER seq 5 permit 10.20.10.0/24
ip prefix-list LOCAL_PITER seq 10 permit 10.20.20.0/24
ip prefix-list LOCAL_PITER seq 15 permit 192.168.98.72/30
ip prefix-list LOCAL_PITER seq 20 permit 192.168.98.76/30
!
router bgp 2042
 address-family ipv4
  neighbor 192.168.98.74 prefix-list LOCAL_PITER out
  neighbor 192.168.98.78 prefix-list LOCAL_PITER out
```

#### **2.2. Проверка фильтрации**

```
R18# show ip bgp neighbors 192.168.98.74 advertised-routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.20.10.0/24    192.168.98.101     1541120         32768 i
 *>  10.20.20.0/24    192.168.98.101     1541120         32768 i
 *>  192.168.98.72/30 0.0.0.0                  0         32768 i
 *>  192.168.98.76/30 0.0.0.0                  0         32768 i

Total number of prefixes 4
```

✅ R18 анонсирует только 4 локальных префикса. Транзитный трафик заблокирован.

---

### **3. Киторн (AS 101): только default в Москву**

**Задача:** R22 должен анонсировать в Москву (R14) только маршрут по умолчанию.

#### **3.1. Конфигурация R22**

```cisco
ip prefix-list DEFAULT_ONLY seq 5 permit 0.0.0.0/0
!
router bgp 101
 address-family ipv4
  neighbor 192.168.98.42 default-originate
  neighbor 192.168.98.42 prefix-list DEFAULT_ONLY out
```

#### **3.2. Проверка на R22**

```
R22# show ip bgp neighbors 192.168.98.42 advertised-routes
Originating default network 0.0.0.0

Total number of prefixes 0
```

✅ R22 анонсирует только default (отображается как `Originating default network`).

#### **3.3. Проверка на R14**

```
R14# show ip bgp neighbors 192.168.98.41 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          192.168.98.41                          0 101 i

Total number of prefixes 1

R14# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "bgp 1001", distance 20, metric 0, candidate default path
  * 192.168.98.41, from 192.168.98.41
```

✅ R14 получает от Киторн только default-маршрут.

---

### **4. Ламас (AS 301): default + сети Питера в Москву**

**Задача:** R21 должен анонсировать в Москву (R15) маршрут по умолчанию и сети офиса С.-Петербург.

#### **4.1. Конфигурация R21**

```cisco
ip prefix-list TO_MOSCOW seq 5 permit 0.0.0.0/0
ip prefix-list TO_MOSCOW seq 10 permit 10.20.10.0/24
ip prefix-list TO_MOSCOW seq 15 permit 10.20.20.0/24
ip prefix-list TO_MOSCOW seq 20 permit 192.168.98.72/30
ip prefix-list TO_MOSCOW seq 25 permit 192.168.98.76/30
!
router bgp 301
 address-family ipv4
  neighbor 192.168.98.45 default-originate
  neighbor 192.168.98.45 prefix-list TO_MOSCOW out
```

#### **4.2. Проверка на R21**

```
R21# show ip bgp neighbors 192.168.98.45 advertised-routes
Originating default network 0.0.0.0

     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.20.10.0/24    192.168.98.54                          0 520 2042 i
 *>  10.20.20.0/24    192.168.98.54                          0 520 2042 i
 *>  192.168.98.72/30 192.168.98.54            0             0 520 i
 *>  192.168.98.76/30 192.168.98.54                          0 520 i

Total number of prefixes 4
```

✅ R21 анонсирует default + 4 сети Питера.

#### **4.3. Проверка на R15**

```
R15# show ip bgp neighbors 192.168.98.46 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          192.168.98.46                          0 301 i
 *>  10.20.10.0/24    192.168.98.46                          0 301 520 2042 i
 *>  10.20.20.0/24    192.168.98.46                          0 301 520 2042 i
 *>  192.168.98.72/30 192.168.98.46                          0 301 520 i
 *>  192.168.98.76/30 192.168.98.46                          0 301 520 i

Total number of prefixes 5
```

✅ R15 получает от Ламас default + сети Питера.

---

### **5. Проверка полной IP-связности**

После применения всех фильтраций проверяем связность между клиентами.

**Москва → Питер:**
```
VPC1> ping 10.20.20.10
84 bytes from 10.20.20.10 icmp_seq=1 ttl=55 time=13.929 ms
84 bytes from 10.20.20.10 icmp_seq=2 ttl=55 time=12.800 ms

VPC7> ping 10.20.10.10
84 bytes from 10.20.10.10 icmp_seq=1 ttl=56 time=22.279 ms
84 bytes from 10.20.10.10 icmp_seq=2 ttl=56 time=12.860 ms
```

**Питер → Москва:**
```
VPC8> ping 10.10.10.10
84 bytes from 10.10.10.10 icmp_seq=1 ttl=56 time=21.873 ms
84 bytes from 10.10.10.10 icmp_seq=2 ttl=56 time=15.064 ms

VPCS> ping 10.10.20.10
84 bytes from 10.10.20.10 icmp_seq=1 ttl=56 time=34.612 ms
84 bytes from 10.10.20.10 icmp_seq=2 ttl=56 time=22.071 ms
```

✅ Полная IP-связность всех сетей обеспечена.

---

## **Выводы**

В ходе лабораторной работы были выполнены следующие задачи:

| Задача | Статус |
|:---|:---:|
| Фильтрация в Москве (AS-Path) — запрет транзита | ✅ |
| Фильтрация в С.-Петербурге (Prefix-list) — запрет транзита | ✅ |
| Киторн → Москва: только default | ✅ |
| Ламас → Москва: default + сети Питера | ✅ |
| Полная IP-связность всех клиентских сетей | ✅ |

**Дополнительно:**
- AS-Path фильтр `^$` и `^1001$` блокирует транзит чужих маршрутов
- Prefix-list `LOCAL_PITER` ограничивает анонсы из Питера 4 локальными префиксами
- `default-originate` в сочетании с prefix-list позволяет гибко управлять анонсами
- Все клиентские сети остаются доступными, несмотря на фильтрацию


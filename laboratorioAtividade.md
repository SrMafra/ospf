# Laboratório: Empresa 1 + Empresa 2 — VLANs, ACLs e NAT entre empresas

## Topologia (conforme seu Packet Tracer)

```
Server-PT DNS(ext) --\                    /-- Server-PT ERP
Server-PT WEB(ext) ---PROVEDOR(2911)--- ROTEADOR-EMPRESA(2901) --- Switch4 --- Server-PT SERVER DNS
                        |                                                \-- PC5
                        |
                      Router0(2901) --- Switch0 --- PC0
                                                  \-- Server2
                                                  \-- Server3
```

- **Empresa 1** = ROTEADOR-EMPRESA + Switch4 + PC5 + Server-PT ERP + Server-PT SERVER DNS
- **Empresa 2** = Router0 + Switch0 + PC0 + Server2 + Server3
- **PROVEDOR** = simula a internet/ISP que interliga as duas empresas (e os servidores externos DNS/WEB do exercício anterior)

## Plano de endereçamento

| Segmento | Rede | Gateway / hosts |
|---|---|---|
| PROVEDOR ↔ Server DNS (externo) | 200.0.0.0/30 | PROVEDOR .1, DNS .2 |
| PROVEDOR ↔ Server WEB (externo) | 200.0.1.0/30 | PROVEDOR .1, WEB .2 |
| PROVEDOR ↔ ROTEADOR-EMPRESA (WAN Empresa 1) | 200.200.200.0/30 | PROVEDOR .1, ROTEADOR-EMPRESA .2 |
| PROVEDOR ↔ Router0 (WAN Empresa 2) | 200.200.201.0/30 | PROVEDOR .1, Router0 .2 |
| VLAN 10 ADMINISTRATIVO (Empresa 1) | 192.168.10.0/24 | gw .1 — PC5 (DHCP) |
| VLAN 20 RH (Empresa 1) | 192.168.20.0/24 | gw .1 — (adicionar PC depois) |
| VLAN 30 SERVIDORES (Empresa 1) | 192.168.30.0/24 | gw .1 — ERP .10, SERVER DNS .20 |
| VLAN 40 ADMINISTRATIVO (Empresa 2) | 192.168.40.0/24 | gw .1 — PC0 (DHCP) |
| VLAN 50 SERVIDORES (Empresa 2) | 192.168.50.0/24 | gw .1 — Server2 (DNS) .10, Server3 (HTTP) .20 |

> Os nomes das interfaces abaixo (Gig0/0, Gig0/1, Gig0/2, Gig0/3...) podem variar conforme os módulos instalados nos seus roteadores/switches no Packet Tracer — confira com `show ip interface brief` e ajuste os comandos se os nomes forem diferentes.

---

## 1. PROVEDOR (2911)

```
enable
configure terminal
hostname PROVEDOR
interface GigabitEthernet0/0
 ip address 200.0.0.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 200.0.1.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 ip address 200.200.200.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/3
 ip address 200.200.201.1 255.255.255.252
 no shutdown
```

Nenhuma rota estática é necessária aqui: as 4 redes já estão diretamente conectadas, então o PROVEDOR já sabe rotear entre Empresa 1, Empresa 2 e os servidores externos.

---

## 2. EMPRESA 1 — ROTEADOR-EMPRESA (2901)

Router-on-a-stick: uma única porta física (Gig0/1) em trunk, com 3 subinterfaces (uma por VLAN).

```
enable
configure terminal
hostname ROTEADOR-EMPRESA

interface GigabitEthernet0/0
 ip address 200.200.200.2 255.255.255.252
 ip nat outside
 no shutdown

interface GigabitEthernet0/1
 no ip address
 no shutdown

interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 ip access-group 110 in

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
 ip access-group 100 in

interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip nat inside

ip route 0.0.0.0 0.0.0.0 200.200.200.1

! DHCP - VLAN 10 ADMINISTRATIVO
ip dhcp excluded-address 192.168.10.1
ip dhcp pool ADMINISTRATIVO
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.30.20

! DHCP - VLAN 20 RH (já deixado pronto pro PC que você for adicionar)
ip dhcp excluded-address 192.168.20.1
ip dhcp pool RH
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.30.20

! ACL 100 - aplicada na VLAN 20 (RH): bloqueia ERP e bloqueia toda comunicação com a VLAN 10
access-list 100 deny ip 192.168.20.0 0.0.0.255 host 192.168.30.10
access-list 100 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 100 permit ip any any

! ACL 110 - aplicada na VLAN 10 (Administrativo): só bloqueia comunicação com a VLAN 20 (ERP continua liberado)
access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip any any

! NAT overload - saída geral para a "internet" (as 3 VLANs internas)
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
access-list 1 permit 192.168.30.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! NAT estático - é isso que libera a Empresa 2 acessar o ERP de fora
ip nat inside source static tcp 192.168.30.10 80 200.200.200.2 80 extendable
```

**Por que assim:** a ACL 100 bloqueia especificamente RH→ERP e também RH↔Administrativo; a ACL 110 bloqueia Administrativo→RH (mas não toca no ERP, então Administrativo continua acessando os servidores normalmente). O NAT estático mapeia porta 80 do IP público do roteador (200.200.200.2) direto pro ERP (192.168.30.10:80) — é essa linha que permite outra empresa alcançar o ERP sem estar na rede interna.

---

## 3. EMPRESA 1 — Switch4 (2960-24TT)

```
enable
configure terminal
hostname Switch4
vlan 10
 name ADMINISTRATIVO
vlan 20
 name RH
vlan 30
 name SERVIDORES

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 30

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 30

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
```
(Fa0/1 = Server-PT ERP, Fa0/2 = Server-PT SERVER DNS, Fa0/3 = PC5 — ajuste as portas conforme os cabos reais)

## 4. EMPRESA 1 — Servidores

**Server-PT ERP** (HTTP)
- IP: `192.168.30.10 / 255.255.255.0`, gateway `192.168.30.1`
- Services → HTTP → ligar, editar `index.html` (ex: "Sistema ERP - Empresa 1")

**Server-PT SERVER DNS**
- IP: `192.168.30.20 / 255.255.255.0`, gateway `192.168.30.1`
- Services → DNS → ligar
- Registro A: `erp.empresa1.local` → `192.168.30.10`

## 5. EMPRESA 1 — PC5

- IP Configuration → DHCP → recebe endereço em 192.168.10.0/24 (VLAN Administrativo)
- **Para testar o bloqueio da VLAN 20 (RH)**, adicione outro PC na Switch4, porta em `vlan 20`, com DHCP (vai pegar 192.168.20.0/24) — a topologia atual só tem PC5.

---

## 6. EMPRESA 2 — Router0 (2901)

```
enable
configure terminal
hostname Router0

interface GigabitEthernet0/0
 ip address 200.200.201.2 255.255.255.252
 ip nat outside
 no shutdown

interface GigabitEthernet0/1
 no ip address
 no shutdown

interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/1.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0
 ip nat inside

ip route 0.0.0.0 0.0.0.0 200.200.201.1

ip dhcp excluded-address 192.168.40.1
ip dhcp pool ADMINISTRATIVO
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 192.168.50.10

access-list 1 permit 192.168.40.0 0.0.0.255
access-list 1 permit 192.168.50.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```

Repare que aqui **não** existe `ip access-group` nas subinterfaces — é justamente isso que faz a VLAN 40 e a VLAN 50 se comunicarem livremente, ao contrário da Empresa 1.

## 7. EMPRESA 2 — Switch0 (2960-24TT)

```
enable
configure terminal
hostname Switch0
vlan 40
 name ADMINISTRATIVO
vlan 50
 name SERVIDORES

interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 40

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 50

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 50
```
(Fa0/1 = PC0, Fa0/2 = Server2, Fa0/3 = Server3 — ajuste conforme os cabos reais)

## 8. EMPRESA 2 — Servidores

**Server2** (DNS)
- IP: `192.168.50.10 / 255.255.255.0`, gateway `192.168.50.1`
- Services → DNS → ligar
- Registros A:
  - `www.empresa2.local` → `192.168.50.20`
  - `erp.empresa1.local` → `200.200.200.2` (assim o PC0 acessa o ERP da Empresa 1 pelo nome, via NAT)

**Server3** (HTTP)
- IP: `192.168.50.20 / 255.255.255.0`, gateway `192.168.50.1`
- Services → HTTP → ligar, editar `index.html` (ex: "Site - Empresa 2")

## 9. EMPRESA 2 — PC0

- IP Configuration → DHCP → recebe endereço em 192.168.40.0/24

---

## 10. Testes para a apresentação

1. **Intranet Empresa 1**: PC5 → `ping 192.168.30.10` (ERP) → deve funcionar.
2. **Bloqueio ERP**: PC da VLAN 20 (RH) → `ping 192.168.30.10` ou navegador → deve **falhar**.
3. **DNS ainda acessível pro RH**: PC da VLAN 20 → `ping 192.168.30.20` → deve funcionar (só o ERP é bloqueado, não o servidor inteiro).
4. **VLAN 10 x VLAN 20 isoladas**: `ping` entre PC5 e o PC da VLAN 20 → deve **falhar** nos dois sentidos.
5. **Empresa 2 integrada**: PC0 → `ping` em Server2 e Server3 → deve funcionar (VLANs 40/50 se comunicam).
6. **Empresa 2 acessando o ERP da Empresa 1**: no PC0, abrir o navegador em `http://erp.empresa1.local` (ou direto `http://200.200.200.2`) → deve abrir a página do ERP, através do NAT estático.
7. No **ROTEADOR-EMPRESA**, rodar `show ip nat translations` → deve aparecer a entrada estática `192.168.30.10:80 ↔ 200.200.200.2:80`, além das entradas dinâmicas (overload) quando alguém navegar pra fora.
8. `show vlan brief` em cada switch, pra conferir se as portas caíram nas VLANs certas.

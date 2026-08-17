# Laboratório: NAT + VLAN (Intranet/Extranet) — Cisco Packet Tracer

## Topologia

```
Server-PT DNS ---\
                   PROVEDOR (2911) --- ROTEADOR-EMPRESA (2901) --- Switch3 (2960-24TT) --- PC0..PC3
Server-PT WEB ---/
```

- **VLAN única**: VLAN 10 ("INTRANET"), aplicada a todas as portas do Switch3 (PCs e link para o roteador).
- **NAT**: feito no ROTEADOR-EMPRESA (overload/PAT), traduzindo a rede interna 192.168.10.0/24 para o IP público 200.200.200.2.
- **Intranet**: comunicação dos PCs dentro da VLAN 10.
- **Extranet/Internet (simulada)**: acesso dos PCs, via NAT, aos servidores DNS e WEB atrás do PROVEDOR.

## Endereçamento IP

| Segmento | Rede | Interface |
|---|---|---|
| PROVEDOR ↔ Server DNS | 200.0.0.0/30 | Gig0/0 PROVEDOR = .1, DNS = .2 |
| PROVEDOR ↔ Server WEB | 200.0.1.0/30 | Gig0/1 PROVEDOR = .1, WEB = .2 |
| PROVEDOR ↔ ROTEADOR-EMPRESA | 200.200.200.0/30 | Gig0/2 PROVEDOR = .1, Gig0/0 ROTEADOR = .2 |
| ROTEADOR-EMPRESA ↔ Switch3 (VLAN 10) | 192.168.10.0/24 | Gig0/1 ROTEADOR = .1, PCs via DHCP |

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
```

## 2. ROTEADOR-EMPRESA (2901)

```
enable
configure terminal
hostname ROTEADOR-EMPRESA
interface GigabitEthernet0/0
 ip address 200.200.200.2 255.255.255.252
 ip nat outside
 no shutdown
interface GigabitEthernet0/1
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.200.200.1

ip dhcp excluded-address 192.168.10.1
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 200.0.0.2

access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```

## 3. Switch3 (2960-24TT) — uma única VLAN

```
enable
configure terminal
hostname Switch3
vlan 10
 name INTRANET
interface range FastEthernet0/1-4
 switchport mode access
 switchport access vlan 10
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
```

> Ajuste os números de porta conforme os cabos reais usados no seu Packet Tracer.

## 4. Server-PT DNS

- IP: `200.0.0.2 / 255.255.255.252`, gateway `200.0.0.1`
- Services → DNS → ligar serviço
- Registro A: `www.empresa.com` → `200.0.1.2`

## 5. Server-PT WEB

- IP: `200.0.1.2 / 255.255.255.252`, gateway `200.0.1.1`
- Services → HTTP → ligar serviço
- Editar `index.html` com algo como "Site simulando a Internet"

## 6. PC0–PC3

- IP Configuration → DHCP
- Devem receber automaticamente: IP em 192.168.10.0/24, gateway 192.168.10.1, DNS 200.0.0.2

## Testes para apresentação

1. `ping` entre PCs (192.168.10.x) → comunicação **intranet** direta.
2. No PC, abrir *Web Browser* e digitar `www.empresa.com` (ou `200.0.1.2`) → navegação até o servidor atrás do PROVEDOR, simulando **internet/extranet**.
3. No ROTEADOR-EMPRESA, rodar `show ip nat translations` depois do acesso ao site → mostra o IP interno (192.168.10.x) traduzido para 200.200.200.2, comprovando o NAT.

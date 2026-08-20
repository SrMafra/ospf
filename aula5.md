# Continuação — Empresa 1 (VLANs) + Empresa 2 + NAT entre empresas

Este arquivo **não repete** o que já está pronto em [REDE-NAT-VLAN.md](REDE-NAT-VLAN.md). Ele parte do estado atual:

- **PROVEDOR**: Gig0/0 → Server DNS externo (200.0.0.x), Gig0/1 → Server WEB externo (200.0.1.x), Gig0/2 → ROTEADOR-EMPRESA (200.200.200.1)
- **ROTEADOR-EMPRESA**: Gig0/0 outside = 200.200.200.2, Gig0/1 inside = 192.168.10.1/24 (direto, sem trunk), NAT overload já ligado, DHCP pool VLAN10
- **Switch4**: VLAN 10 "INTRANET" em todas as portas, **PC5** nela

Não mexe em nada disso — só evolui em cima.

---

## PARTE A — EMPRESA 1

### A1. Switch4 — criar VLAN Servidores e virar trunk o link pro roteador

```
enable
configure terminal
vlan 30
 name SERVIDORES
exit

interface FastEthernet0/5
 switchport mode access
 switchport access vlan 30
description Server-PT-ERP

interface FastEthernet0/6
 switchport mode access
 switchport access vlan 30
description Server-PT-SERVER-DNS

interface GigabitEthernet0/1
 switchport mode trunk
```
(Fa0/1-4 continuam VLAN 10, como já estava. Fa0/5-10 = faixa reservada pra VLAN 30 Servidores: Fa0/5 = ERP, Fa0/6 = SERVER DNS. Gig0/1 = link pro ROTEADOR-EMPRESA, já trunk.)

### A2. ROTEADOR-EMPRESA — migrar Gig0/1 pra sub-interfaces

```
configure terminal
interface GigabitEthernet0/1
 no ip address
 no shutdown

interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip nat inside

interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip nat inside

! adicionar a rede nova na ACL de NAT que já existe (access-list 1)
access-list 1 permit 192.168.30.0 0.0.0.255
```
> A rede 192.168.10.0/24, o DHCP pool VLAN10 e a `access-list 1 permit 192.168.10.0 0.0.0.255` continuam exatamente como estavam — só migraram de interface física pra sub-interface `.10`.

### A3. Servidores internos (Empresa 1)

**Server-PT ERP**
- IP: `192.168.30.10 / 255.255.255.0`, gateway `192.168.30.1`
- Services → HTTP → ligar, editar `index.html` (ex: "Sistema ERP - Empresa 1")

**Server-PT SERVER DNS**
- IP: `192.168.30.20 / 255.255.255.0`, gateway `192.168.30.1`
- Services → DNS → ligar
- Registro A: `erp.empresa1.local` → `192.168.30.10`

Teste rápido: no PC5, `ping 192.168.30.10` e abrir `http://erp.empresa1.local` no navegador.

### A4. Renomear VLAN 10 para ADMINISTRATIVO

Só o nome muda — rede e IPs continuam os mesmos.
```
configure terminal
vlan 10
 name ADMINISTRATIVO
```

### A5. Adicionar VLAN 20 (RH)

```
! No Switch4
vlan 20
 name RH
! faixa reservada pra VLAN 20: Fa0/11-15 (adicione um PC novo no Switch4 pra testar)
interface FastEthernet0/11
 switchport mode access
 switchport access vlan 20
```

```
! No ROTEADOR-EMPRESA
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside

access-list 1 permit 192.168.20.0 0.0.0.255

ip dhcp excluded-address 192.168.20.1
ip dhcp pool RH
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.30.20
```

### A6. Bloquear VLAN 20 (RH) → ERP, e VLAN 10 ↔ VLAN 20 sem comunicação

```
! No ROTEADOR-EMPRESA

! ACL 100 - aplicada na VLAN 20: bloqueia o ERP especificamente e bloqueia toda comunicação com a VLAN 10
access-list 100 deny ip 192.168.20.0 0.0.0.255 host 192.168.30.10
access-list 100 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 100 permit ip any any

! ACL 110 - aplicada na VLAN 10: bloqueia só a comunicação com a VLAN 20 (ERP continua liberado pro Administrativo)
access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip any any

interface GigabitEthernet0/1.20
 ip access-group 100 in

interface GigabitEthernet0/1.10
 ip access-group 110 in
```

**Teste**: PC de RH → `ping 192.168.30.10` (deve falhar) e `ping 192.168.30.20` (deve funcionar, só o ERP é bloqueado). PC5 (Administrativo) ↔ PC de RH → `ping` deve falhar nos dois sentidos.

---

## PARTE B — EMPRESA 2 (nova)

Empresa 2 ainda não existe na sua topologia — é um roteador (Router0), switch (Switch0), PC0, Server2 e Server3, ligados ao PROVEDOR.

### B1. PROVEDOR — nova interface pro link com a Empresa 2

As 3 portas Gigabit onboard do PROVEDOR já estão todas ocupadas (DNS, WEB, ROTEADOR-EMPRESA). Não precisa adicionar módulo novo — já existe um módulo no slot 3 com 4 portas `FastEthernet0/3/0` a `0/3/3` livres.

**Detalhe importante**: essas portas são de um mini-switch embutido (Camada 2) — têm modo Access/Trunk e VLAN como uma porta de switch comum, e **não aceitam IP direto**. O IP do link tem que ficar na interface virtual `Vlan1` (o gateway desse mini-switch):

```
configure terminal
interface FastEthernet0/3/0
 switchport mode access
 switchport access vlan 1
 no shutdown

interface Vlan1
 ip address 200.200.201.1 255.255.255.252
 no shutdown
```

Confirme com `show ip interface brief` — a `Vlan1` deve aparecer com o IP `200.200.201.1` e status `up/up`.

(se no seu Packet Tracer a porta livre tiver outro nome, ajuste — mas o princípio é o mesmo: `switchport` na porta física + IP na `Vlan1`)

### B2. Router0 — router-on-a-stick com 2 VLANs que se comunicam

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
> Repare: **nenhuma** `ip access-group` nas sub-interfaces — é isso que faz as VLANs 40 (Administrativo) e 50 (Servidores) se comunicarem livremente, diferente da Empresa 1.

### B3. Switch0

```
enable
configure terminal
hostname Switch0
vlan 40
 name ADMINISTRATIVO
vlan 50
 name SERVIDORES

interface GigabitEthernet0/1
 switchport mode trunk

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 40
description PC0

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 50
description Server2

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 50
description Server3
```

### B4. Servidores da Empresa 2

**Server2** (DNS)
- IP: `192.168.50.10 / 255.255.255.0`, gateway `192.168.50.1`
- Services → DNS → ligar
- Registros A: `www.empresa2.local` → `192.168.50.20` e `erp.empresa1.local` → `200.200.200.2` (pro PC0 acessar o ERP da Empresa 1 pelo nome)

**Server3** (HTTP)
- IP: `192.168.50.20 / 255.255.255.0`, gateway `192.168.50.1`
- Services → HTTP → ligar, editar `index.html` (ex: "Site - Empresa 2")

### B5. PC0

- IP Configuration → DHCP → recebe endereço em 192.168.40.0/24

---

## PARTE C — NAT para a Empresa 2 acessar o ERP da Empresa 1

```
! No ROTEADOR-EMPRESA (não no Router0!)
ip nat inside source static tcp 192.168.30.10 80 200.200.200.2 80
```

Isso expõe a porta 80 do ERP (192.168.30.10) no IP público do roteador da Empresa 1 (200.200.200.2). Como o registro DNS `erp.empresa1.local → 200.200.200.2` já foi criado no Server2 (passo B4), o PC0 acessa direto pelo nome.

**Teste final**: no PC0 (Empresa 2), abrir o navegador em `http://erp.empresa1.local` (ou `http://200.200.200.2`) → deve abrir a página do ERP da Empresa 1. No ROTEADOR-EMPRESA, `show ip nat translations` deve mostrar a entrada estática `192.168.30.10:80 ↔ 200.200.200.2:80`.

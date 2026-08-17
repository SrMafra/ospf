# NAT Seletivo por VLAN — Cisco Packet Tracer

Laboratório: 1 switch com 2 VLANs atrás de um roteador com NAT. A VLAN 10 sai para uma internet "de verdade" (IP público via DHCP + site navegável por DNS/HTTP). A VLAN 20 fica bloqueada.

## 1. Topologia

```
[Web Server]     [DNS Server]
 203.0.113.10     203.0.113.53
        \            /
         \          /
        [R-ISP] Gi0/1 --- 203.0.113.0/24
           |
           | Gi0/0  (pool DHCP 200.200.200.0/29 -> simula o provedor)
           |
        [R-GW] Gi0/0  (ip address dhcp, ip nat outside)
           |
           | Gi0/1  (trunk 802.1Q)
           |
         [SW1] Fa0/24 trunk
          /          \
    Fa0/1-2         Fa0/3-4
   VLAN 10          VLAN 20
192.168.10.0/24   192.168.20.0/24
  COM internet      SEM internet
   PC1, PC2          PC3, PC4
```

## 2. Equipamentos

| Equipamento | Modelo sugerido | Função |
|---|---|---|
| R-ISP | Router 1941/2911 | Simula o provedor: entrega IP público via DHCP e hospeda a "internet" (LAN com DNS + Web Server) |
| DNS Server | Server-PT | Resolve `www.internet.com` |
| Web Server | Server-PT | Site que os PCs vão acessar pelo navegador |
| R-GW | Router 1941/2911 (2 Gigabit) | Gateway da rede interna, recebe IP via DHCP do R-ISP, faz NAT |
| SW1 | Switch 2960 | VLAN 10 e VLAN 20 |
| PC1, PC2 | PC genérico | VLAN 10 — com internet |
| PC3, PC4 | PC genérico | VLAN 20 — sem internet |

## 3. Cabeamento

- R-ISP `Gi0/0` ↔ R-GW `Gi0/0`
- R-ISP `Gi0/1` ↔ Switch/hub da "internet" ↔ DNS Server e Web Server (ou ligue os dois servidores direto em portas separadas do R-ISP se ele tiver, senão use um switch simples nesse segmento)
- R-GW `Gi0/1` ↔ SW1 `Fa0/24` (**um único cabo**, trunk)
- SW1 `Fa0/1`, `Fa0/2` ↔ PC1, PC2 (VLAN 10)
- SW1 `Fa0/3`, `Fa0/4` ↔ PC3, PC4 (VLAN 20)

## 4. Plano de endereçamento

| Rede / host | Endereço | Observação |
|---|---|---|
| R-ISP ↔ R-GW (WAN) | `200.200.200.0/29` | R-ISP = `.1` (fixo). R-GW recebe IP **via DHCP** desse pool — simula o provedor entregando IP público |
| Internet (LAN do R-ISP) | `203.0.113.0/24` | R-ISP = `.1` |
| DNS Server | `203.0.113.53/24` | gw `203.0.113.1` |
| Web Server | `203.0.113.10/24` | gw `203.0.113.1`, registrado no DNS como `www.internet.com` |
| VLAN 10 | `192.168.10.0/24` | gw `192.168.10.1` (Gi0/1.10) — **com internet** |
| VLAN 20 | `192.168.20.0/24` | gw `192.168.20.1` (Gi0/1.20) — **sem internet** |
| PC1 / PC2 | `192.168.10.10` / `.11` | máscara `255.255.255.0`, gw `192.168.10.1`, DNS `203.0.113.53` |
| PC3 / PC4 | `192.168.20.10` / `.11` | máscara `255.255.255.0`, gw `192.168.20.1`, DNS `203.0.113.53` |

## 5. Switch SW1 — VLANs e trunk

```
enable
configure terminal
hostname SW1
vlan 10
 name VLAN10_INTERNET
exit
vlan 20
 name VLAN20_SEM_INTERNET
exit
interface range fastEthernet0/1-2
 switchport mode access
 switchport access vlan 10
exit
interface range fastEthernet0/3-4
 switchport mode access
 switchport access vlan 20
exit
interface fastEthernet0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit
end
write memory
```

## 6. Router R-ISP — simulando o provedor

Entrega IP público dinamicamente (como um provedor real faz) e hospeda a "internet" de teste.

```
enable
configure terminal
hostname R-ISP
interface gigabitEthernet0/0
 ip address 200.200.200.1 255.255.255.248
 no shutdown
exit
interface gigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.0
 no shutdown
exit
ip dhcp excluded-address 200.200.200.1
ip dhcp pool ISP_WAN
 network 200.200.200.0 255.255.255.248
 default-router 200.200.200.1
 dns-server 203.0.113.53
exit
end
write memory
```

## 7. DNS Server e Web Server (configuração via GUI, não CLI)

**Web Server:**
1. Clique no servidor → aba **Desktop → IP Configuration** → IP `203.0.113.10`, máscara `255.255.255.0`, gateway `203.0.113.1`.
2. Aba **Services → HTTP** → deixe habilitado. Edite o `index.html` para algo como `Você está na internet! (VLAN 10)`, só para deixar o teste visível.

**DNS Server:**
1. IP `203.0.113.53`, máscara `255.255.255.0`, gateway `203.0.113.1`.
2. Aba **Services → DNS** → habilite o serviço.
3. Adicione um registro: Type `A Record`, Name `www.internet.com`, Address `203.0.113.10` → **Add**.

## 8. Router R-GW — WAN dinâmica + sub-interfaces

```
enable
configure terminal
hostname R-GW
! WAN recebe IP publico via DHCP, como um roteador residencial de verdade
interface gigabitEthernet0/0
 ip address dhcp
 ip nat outside
 no shutdown
exit
interface gigabitEthernet0/1
 no shutdown
exit
interface gigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
exit
interface gigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
exit
end
write memory
```

Confira com `show ip route` se uma rota padrão (`0.0.0.0/0`) apareceu automaticamente via DHCP. Se não aparecer, descubra o IP recebido em `show ip interface brief` e complete manualmente:

```
configure terminal
ip route 0.0.0.0 0.0.0.0 200.200.200.1
end
```

## 9. NAT seletivo — libera VLAN 10, bloqueia VLAN 20

```
configure terminal
! só a rede da VLAN 10 é traduzida
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface gigabitEthernet0/0 overload
! bloqueio explícito da VLAN 20 (permite só tráfego local)
ip access-list extended BLOQUEIA_VLAN20
 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit icmp 192.168.20.0 0.0.0.255 host 192.168.20.1
 deny ip 192.168.20.0 0.0.0.255 any
 permit ip any any
exit
interface gigabitEthernet0/1.20
 ip access-group BLOQUEIA_VLAN20 in
exit
end
write memory
```

> Sem a `access-list 1` incluir a VLAN 20, os pacotes dela nem seriam traduzidos — mas ainda tentariam sair com IP privado, e o R-ISP não saberia rotear a resposta de volta. A ACL `BLOQUEIA_VLAN20` deixa o bloqueio explícito e ainda permite que a VLAN 20 continue enxergando a VLAN 10 e o próprio gateway.

## 10. PCs — IP estático + DNS

Em cada PC: **Desktop → IP Configuration → Static**.

| PC | IP | Máscara | Gateway | DNS Server |
|---|---|---|---|---|
| PC1 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 | 203.0.113.53 |
| PC2 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 | 203.0.113.53 |
| PC3 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 | 203.0.113.53 |
| PC4 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 | 203.0.113.53 |

## 11. Verificação

| Onde | Ação | Resultado esperado |
|---|---|---|
| R-GW | `show ip interface brief` | Gi0/0 com IP recebido do pool `200.200.200.0/29`, Gi0/1.10 e Gi0/1.20 up/up |
| R-GW | `show ip nat translations` | Só entradas com origem `192.168.10.x` |
| PC1/PC2 | Desktop → Web Browser → `http://www.internet.com` | Página carrega (site do Web Server) |
| PC1/PC2 | `ping www.internet.com` | Reply |
| PC3/PC4 | Desktop → Web Browser → `http://www.internet.com` | Falha (não resolve / não carrega) |
| PC3/PC4 | `ping 192.168.20.1` | Reply — o gateway local continua acessível |

**Depuração:** se a VLAN 10 também não sair, confira primeiro `show ip nat translations` no R-GW — geralmente é a `access-list 1` ou o `ip nat inside`/`outside` faltando em alguma interface.

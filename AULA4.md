# NAT Seletivo por VLAN — Cisco Packet Tracer

Laboratório: 1 switch com 2 VLANs atrás de um roteador com NAT. A VLAN 10 sai para uma internet "de verdade" (IP público via DHCP + site navegável por DNS/HTTP). A VLAN 20 fica bloqueada.

## 1. Topologia

Sem switch extra: o R-ISP é um roteador com 3 portas Gigabit (modelo 2911), uma para cada servidor e uma para o R-GW.

```
[Web Server]                    [DNS Server]
203.0.113.2                     203.0.113.6
     |  ponto-a-ponto                |  ponto-a-ponto
     |  203.0.113.0/30                |  203.0.113.4/30
     |                                |
   Gi0/1 -------------[R-ISP]------------- Gi0/2
                          |
                          | Gi0/0  (pool DHCP 200.200.200.0/29 -> simula o provedor)
                          |
                       [R-GW] Gi0/0  (ip address dhcp, ip nat outside)
                          |
                          | Gi0/1  (trunk 802.1Q, 1 cabo só)
                          |
                        [SW1] Gi0/1 trunk
                         /          \
                  Fa0/1-12         Fa0/13-24
                  VLAN 10          VLAN 20
               192.168.10.0/24   192.168.20.0/24
                 COM internet      SEM internet
              PC1 (Fa0/1), PC2 (Fa0/2)   PC3 (Fa0/13), PC4 (Fa0/14)
```

## 2. Equipamentos

| Equipamento | Modelo sugerido | Função |
|---|---|---|
| R-ISP | Router **2911** (3 portas Gigabit onboard) | Simula o provedor: entrega IP público via DHCP e conecta os dois servidores direto, sem switch |
| DNS Server | Server-PT | Resolve `www.internet.com` |
| Web Server | Server-PT | Site que os PCs vão acessar pelo navegador |
| R-GW | Router 1941/2911 (2 Gigabit) | Gateway da rede interna, recebe IP via DHCP do R-ISP, faz NAT |
| SW1 | Switch 2960-24TT (24x FastEthernet + 2x GigabitEthernet) | VLAN 10 (Fa0/1-12) e VLAN 20 (Fa0/13-24), trunk em Gi0/1 |
| PC1, PC2 | PC genérico | VLAN 10 — com internet (em Fa0/1 e Fa0/2) |
| PC3, PC4 | PC genérico | VLAN 20 — sem internet (em Fa0/13 e Fa0/14) |

## 3. Cabeamento

8 cabos ao todo, um único switch (o SW1 das VLANs) — nenhum switch extra do lado da internet. O trunk usa a porta Gigabit do switch, não uma FastEthernet.

| # | De | Porta | Para | Porta |
|---|---|---|---|---|
| 1 | Web Server | FastEthernet0 | R-ISP | Gi0/1 |
| 2 | DNS Server | FastEthernet0 | R-ISP | Gi0/2 |
| 3 | R-ISP | Gi0/0 | R-GW | Gi0/0 |
| 4 | R-GW | Gi0/1 | SW1 | **Gi0/1** |
| 5 | SW1 | Fa0/1 | PC1 | FastEthernet0 |
| 6 | SW1 | Fa0/2 | PC2 | FastEthernet0 |
| 7 | SW1 | Fa0/13 | PC3 | FastEthernet0 |
| 8 | SW1 | Fa0/14 | PC4 | FastEthernet0 |

As demais portas (Fa0/3 a Fa0/12 = VLAN 10, Fa0/15 a Fa0/24 = VLAN 20) ficam configuradas e livres, prontas pra plugar mais PCs depois se quiser.

> **Confira antes de configurar IP em qualquer coisa.** É fácil trocar sem querer a porta de um cabo (ex: ligar o R-GW no `Gi0/2` do R-ISP em vez do `Gi0/0`) e passar horas depurando IP/DHCP quando o problema era só o cabo no lugar errado. Depois de cabear, em cada roteador rode:
> ```bash
> show cdp neighbors
> ```
> e confira se o vizinho certo aparece na porta certa (ex: no R-GW, `Gi0/0` deve mostrar `R-ISP`). Só depois disso comece a digitar `ip address`.

## 4. Plano de endereçamento

| Rede / host | Endereço | Observação |
|---|---|---|
| R-ISP ↔ R-GW (WAN) | `200.200.200.0/29` | R-ISP `Gi0/0` = `.1` (fixo). R-GW recebe IP **via DHCP** desse pool — simula o provedor entregando IP público |
| R-ISP ↔ Web Server | `203.0.113.0/30` | R-ISP `Gi0/1` = `.1`, Web Server = `.2` |
| R-ISP ↔ DNS Server | `203.0.113.4/30` | R-ISP `Gi0/2` = `.5`, DNS Server = `.6` |
| VLAN 10 | `192.168.10.0/24` | gw `192.168.10.1` (Gi0/1.10) — **com internet** |
| VLAN 20 | `192.168.20.0/24` | gw `192.168.20.1` (Gi0/1.20) — **sem internet** |
| PC1 / PC2 | `192.168.10.10` / `.11` | máscara `255.255.255.0`, gw `192.168.10.1`, DNS `203.0.113.6` |
| PC3 / PC4 | `192.168.20.10` / `.11` | máscara `255.255.255.0`, gw `192.168.20.1`, DNS `203.0.113.6` |

## 5. Switch SW1 — VLANs e trunk

Divisão: **Fa0/1 a Fa0/12 = VLAN 10**, **Fa0/13 a Fa0/24 = VLAN 20**, trunk na porta Gigabit (`Gi0/1`) pro roteador. Usa `interface range` com espaço antes e depois do hífen — sem o espaço, o Packet Tracer às vezes só pega a primeira porta do intervalo, que foi o que bagunçou a config anterior.

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

interface range fastEthernet0/1 - 12
 switchport mode access
 switchport access vlan 10
exit

interface range fastEthernet0/13 - 24
 switchport mode access
 switchport access vlan 20
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

end
write memory
```

**Confira imediatamente depois**, antes de seguir pro resto do lab:

```bash
show vlan brief
```

Resultado esperado:

| VLAN | Nome | Portas |
|---|---|---|
| 1 | default | nenhuma (todas as 24 portas foram atribuídas) |
| 10 | VLAN10_INTERNET | Fa0/1, Fa0/2, Fa0/3, Fa0/4, Fa0/5, Fa0/6, Fa0/7, Fa0/8, Fa0/9, Fa0/10, Fa0/11, Fa0/12 |
| 20 | VLAN20_SEM_INTERNET | Fa0/13, Fa0/14, Fa0/15, Fa0/16, Fa0/17, Fa0/18, Fa0/19, Fa0/20, Fa0/21, Fa0/22, Fa0/23, Fa0/24 |

`Gi0/1` **não aparece em nenhuma VLAN** nessa lista — normal, porta trunk carrega as duas. Se alguma porta individual aparecer fora do lugar (ex: Fa0/7 ainda em VLAN 1), rode só o bloco dela sozinha, sem range:
```
configure terminal
interface fastEthernet0/7
 switchport mode access
 switchport access vlan 10
exit
end
```

## 6. Router R-ISP — simulando o provedor

Um roteador **2911** com 3 portas Gigabit: uma para o Web Server, uma para o DNS Server, uma para o R-GW. Entrega IP público dinamicamente ao R-GW (como um provedor real faz) e roteia direto para os dois servidores, sem switch no meio.

> **Se `interface gigabitEthernet0/2` der `%Invalid interface type and number`:** esse roteador não tem uma terceira porta de fábrica. Rode `show ip interface brief` pra ver o que existe de verdade. Se só tiver `Gi0/0` e `Gi0/1`, adicione uma porta: clique no roteador → aba **Physical** → desligue no botão de força → arraste um módulo com porta Ethernet (ex: `NM-1FE-TX` ou `PT-ROUTER-NM-1FGE`) pra um slot vazio → ligue de novo. Confira o nome exato da interface nova com `show ip interface brief` antes de configurar — pode não ser `GigabitEthernet0/2`.

```
enable
configure terminal
hostname R-ISP
! link com o R-GW (WAN do cliente)
interface gigabitEthernet0/0
 ip address 200.200.200.1 255.255.255.248
 no shutdown
exit
! link ponto-a-ponto com o Web Server
interface gigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.252
 no shutdown
exit
! link ponto-a-ponto com o DNS Server
interface gigabitEthernet0/2
 ip address 203.0.113.5 255.255.255.252
 no shutdown
exit
ip dhcp excluded-address 200.200.200.1
ip dhcp pool ISP_WAN
 network 200.200.200.0 255.255.255.248
 default-router 200.200.200.1
 dns-server 203.0.113.6
exit
end
write memory
```

## 7. DNS Server e Web Server (configuração via GUI, não CLI)

**Web Server:**
1. Clique no servidor → aba **Desktop → IP Configuration** → IP `203.0.113.2`, máscara `255.255.255.252`, gateway `203.0.113.1`.
2. Aba **Services → HTTP** → deixe habilitado. Edite o `index.html` para algo como `Você está na internet! (VLAN 10)`, só para deixar o teste visível.

**DNS Server:**
1. IP `203.0.113.6`, máscara `255.255.255.252`, gateway `203.0.113.5`.
2. Aba **Services → DNS** → habilite o serviço.
3. Adicione um registro: Type `A Record`, Name `www.internet.com`, Address `203.0.113.2` → **Add**.

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

Confira com `show ip interface brief` se o `Gi0/0` recebeu um IP do pool (algo entre `200.200.200.2` e `.6`). Se continuar `unassigned`:

1. Confirme que o pool existe no R-ISP: `show ip dhcp pool` (se der vazio ou erro, o pool não foi criado — refaça o passo 6).
2. Confirme o cabo certo com `show cdp neighbors` — deve aparecer `R-ISP` na porta `Gi0/0`.
3. Force o cliente DHCP a pedir de novo (um simples `shutdown`/`no shutdown` às vezes não é suficiente):
```
enable
configure terminal
interface gigabitEthernet0/0
 no ip address dhcp
 ip address dhcp
exit
end
show ip interface brief
```

Depois de receber o IP, confira com `show ip route` se uma rota padrão (`0.0.0.0/0`) apareceu automaticamente via DHCP. Se não aparecer, complete manualmente com o IP do R-ISP:

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

| PC | Porta no SW1 | IP | Máscara | Gateway | DNS Server |
|---|---|---|---|---|---|
| PC1 | Fa0/1 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 | 203.0.113.6 |
| PC2 | Fa0/2 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 | 203.0.113.6 |
| PC3 | Fa0/13 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 | 203.0.113.6 |
| PC4 | Fa0/14 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 | 203.0.113.6 |

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

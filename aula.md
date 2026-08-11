# Passo a Passo — Rede Cisco: Filiais Rondônia, São Paulo e Minas Gerais

VLANs + OSPF | Roteiro para apresentação em aula

## 1. Topologia

```
 Filial RONDÔNIA (RO)          Filial SÃO PAULO (SP)          Filial MINAS GERAIS (MG)
+--------------------+        +--------------------+        +--------------------+
| VLAN 10 - Vendas   |        | VLAN 30 - Vendas   |        | VLAN 50 - Vendas   |
| 192.168.10.0/24    |        | 192.168.30.0/24    |        | 192.168.50.0/24    |
|                    |        |                    |        |                    |
| VLAN 20 - Adm      |        | VLAN 40 - Adm      |        | VLAN 60 - Adm      |
| 192.168.20.0/24    |        | 192.168.40.0/24    |        | 192.168.60.0/24    |
+---------+----------+        +---------+----------+        +---------+----------+
          | Gi0/1 (trunk)                | Gi0/1 (trunk)                | Gi0/1 (trunk)
       [SW-RO]                        [SW-SP]                        [SW-MG]
          | Gi0/0                        | Gi0/0                        | Gi0/0
       [R-RO] Se0/3/0 --- Se0/3/0 [R-SP] Se0/3/1 --- Se0/3/0 [R-MG]
              192.168.100.1/30  192.168.100.2/30 192.168.101.1/30 192.168.101.2/30
```

- **Roteamento:** Router-on-a-stick — uma porta física do roteador dividida em
  sub-interfaces 802.1Q, uma por VLAN.
- **Protocolo de roteamento:** OSPF, área única (Area 0), Process-ID 1.
- **Topologia entre filiais:** cadeia (RO ↔ SP ↔ MG) — SP fica no meio, RO fala com
  MG através de SP (rota multi-hop calculada pelo OSPF).
- **Links WAN:** Serial ponto-a-ponto RO-SP (192.168.100.0/30) e SP-MG (192.168.101.0/30).

## 2. Equipamentos usados

| Papel | Modelo | Detalhe |
|-------|--------|---------|
| Roteador (3x) | **Cisco 2901** | + módulo **HWIC-2T** encaixado no **slot 3** → gera as interfaces `Serial0/3/0` e `Serial0/3/1` |
| Switch (3x) | **Cisco 2960-24TT-L** | 24 portas FastEthernet (Fa0/1–0/24) + 2 portas GigabitEthernet (Gi0/1, Gi0/2) |

> ⚠️ O roteador de SP usa as **duas portas** do próprio módulo HWIC-2T:
> `Serial0/3/0` (já usada, liga em RO) e `Serial0/3/1` (livre, vai ligar em MG).
> Não precisa de módulo novo no R-SP — só no R-MG.
>
> O slot do módulo HWIC-2T pode variar conforme onde você encaixar no Packet
> Tracer. Depois de instalar, sempre confira o nome real da interface com
> `do show ip interface brief` antes de configurar a serial.

## 3. Plano de Endereçamento IP

| Local | VLAN | Nome        | Rede             | Gateway (Router)  |
|-------|------|-------------|------------------|--------------------|
| RO    | 10   | Vendas-RO   | 192.168.10.0/24  | 192.168.10.1       |
| RO    | 20   | Adm-RO      | 192.168.20.0/24  | 192.168.20.1       |
| SP    | 30   | Vendas-SP   | 192.168.30.0/24  | 192.168.30.1       |
| SP    | 40   | Adm-SP      | 192.168.40.0/24  | 192.168.40.1       |
| MG    | 50   | Vendas-MG   | 192.168.50.0/24  | 192.168.50.1       |
| MG    | 60   | Adm-MG      | 192.168.60.0/24  | 192.168.60.1       |
| WAN   | —    | Link RO-SP  | 192.168.100.0/30 | .1 (RO) / .2 (SP)  |
| WAN   | —    | Link SP-MG  | 192.168.101.0/30 | .1 (SP) / .2 (MG)  |

---

# PASSO A PASSO

## Passo 1 — Montar a topologia física no Packet Tracer

1. Arraste 2 roteadores **2901**, 2 switches **2960-24TT-L** e pelo menos 2 PCs por
   filial (1 na VLAN de Vendas, 1 na VLAN de Adm).
2. Cabos:
   - Roteador `Gi0/0` **↔** Switch `Gi0/1` (será a porta trunk) — cabo Copper Straight-Through
   - PCs **↔** portas `Fa0/1` a `Fa0/20` do switch (acesso)
   - Roteador RO `Se0/3/0` **↔** Roteador SP `Se0/3/0` — cabo **Serial DCE**

## Passo 2 — Instalar o módulo HWIC-2T nos dois roteadores

1. Clique no roteador → aba **Physical**
2. Desligue a energia (botão liga/desliga na lateral)
3. Arraste o módulo **HWIC-2T** para um slot vazio
4. Ligue novamente
5. Confirme o nome da interface serial criada:
   ```
   enable
   show ip interface brief
   ```
   (no nosso teste em aula deu `Serial0/3/0` — se no seu der diferente, use esse
   nome em todos os comandos abaixo)

## Passo 3 — Configurar o Switch SW-RO (Filial Rondônia)

```
enable
configure terminal
hostname SW-RO
vlan 10
 name Vendas-RO
vlan 20
 name Adm-RO
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 10
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 20
end
write memory
```

## Passo 4 — Configurar o Roteador R-RO (Filial Rondônia)

```
enable
configure terminal
hostname R-RO
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.100.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.3 area 0
end
write memory
```

> `clock rate 64000` só entra no lado **DCE** do cabo serial. Confirme com
> `show controllers Serial0/3/0`. Se o R-RO for o DTE, remova essa linha daqui e
> coloque no R-SP.

## Passo 5 — Configurar o Switch SW-SP (Filial São Paulo)

```
enable
configure terminal
hostname SW-SP
vlan 30
 name Vendas-SP
vlan 40
 name Adm-SP
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 30,40
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 30
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 40
end
write memory
```

## Passo 6 — Configurar o Roteador R-SP (Filial São Paulo)

```
enable
configure terminal
hostname R-SP
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.100.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 2.2.2.2
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.40.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.3 area 0
end
write memory
```

## Passo 7 — Configurar os PCs

| Filial | VLAN | IP              | Máscara         | Gateway         |
|--------|------|------------------|------------------|------------------|
| RO     | 10   | 192.168.10.10    | 255.255.255.0    | 192.168.10.1     |
| RO     | 20   | 192.168.20.10    | 255.255.255.0    | 192.168.20.1     |
| SP     | 30   | 192.168.30.10    | 255.255.255.0    | 192.168.30.1     |
| SP     | 40   | 192.168.40.10    | 255.255.255.0    | 192.168.40.1     |

No PC: aba **Desktop → IP Configuration** → preencher IP, máscara e gateway acima.

## Passo 8 — Verificação final

Em cada roteador:

```
show vlan brief
show ip interface brief
show ip ospf neighbor
show ip route ospf
```

**O que checar:**
- `Gi0/0.10`, `Gi0/0.20` (ou .30/.40) → `up / up`
- `Serial0/3/0` → `up / up`
- `show ip ospf neighbor` → vizinho aparece com estado **FULL**
- `show ip route ospf` → aparecem rotas `O` para as redes da outra filial

Teste de ping fim a fim (de um PC):

```
ping 192.168.30.10   ! de um PC da VLAN 10 (RO) até a VLAN 30 (SP)
ping 192.168.40.10   ! de um PC da VLAN 20 (RO) até a VLAN 40 (SP)
```

## Passo 9 — Troubleshooting rápido (erros comuns vistos nos testes)

| Sintoma | Causa | Solução |
|---|---|---|
| `% Invalid input detected` ao digitar `interface ...` | Você não está em `(config)#`, ainda está em `Router>` ou `Router#` | Rode `enable` → `configure terminal` antes |
| `%Invalid interface type and number` na serial | Módulo HWIC-2T não instalado ou em outro slot | Rode `show ip interface brief` pra achar o nome real |
| Sub-interface com **Status up / Protocol down** | Porta física `Gi0/0` sem cabo, ou switch sem trunk configurado | Conectar cabo e configurar `switchport mode trunk` no switch |
| Serial com **Status down / Protocol down** | Cabo serial não conectado, ou falta `clock rate` no lado DCE | Conectar cabo DCE e rodar `show controllers` para achar o lado DCE |
| `show ip ospf neighbor` vazio | Redes do `network` no OSPF não batem com a rede real, ou link ainda down | Conferir wildcard mask e se a interface está `up/up` |

---

# PARTE 2 — Adicionando a Filial Minas Gerais (MG)

Topologia em **cadeia**: `RO ↔ SP ↔ MG`. RO e MG não têm link direto — o tráfego
entre eles passa por dentro de SP, e o OSPF calcula essa rota sozinho (2 saltos).

## Passo 10 — Montar a topologia física da filial MG

1. Adicione 1 roteador **2901** (`R-MG`), 1 switch **2960-24TT-L** (`SW-MG`) e 2 PCs
   (um pra VLAN 50, outro pra VLAN 60).
2. Cabos:
   - `R-MG Gi0/0` **↔** `SW-MG Gi0/1` — Copper Straight-Through
   - PCs **↔** `SW-MG Fa0/1-20` — Copper Straight-Through
   - `R-SP Serial0/3/1` **↔** `R-MG Serial0/3/0` — **Serial DCE**

> Repare: do lado de SP você usa a porta **Se0/3/1** (a que sobrou livre no
> módulo), não a Se0/3/0 (que já está ocupada com o link pra RO).

## Passo 11 — Instalar o módulo HWIC-2T no roteador R-MG

Mesmo processo do Passo 2: desligar o roteador na aba Physical, arrastar o
**HWIC-2T** pra um slot vazio, ligar de novo, e conferir com
`show ip interface brief` qual nome a interface serial recebeu (nos outros dois
roteadores deu `Serial0/3/0`).

## Passo 12 — Configurar o Switch SW-MG

```
enable
configure terminal
hostname SW-MG
vlan 50
 name Vendas-MG
vlan 60
 name Adm-MG
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 50,60
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 50
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 60
end
write memory
```

## Passo 13 — Configurar o Roteador R-MG

```
enable
configure terminal
hostname R-MG
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0
interface GigabitEthernet0/0.60
 encapsulation dot1Q 60
 ip address 192.168.60.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.101.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 3.3.3.3
 network 192.168.50.0 0.0.0.255 area 0
 network 192.168.60.0 0.0.0.255 area 0
 network 192.168.101.0 0.0.0.3 area 0
end
write memory
```

> Se o cabo serial colocar o R-MG como lado **DCE**, adicione `clock rate 64000`
> embaixo do `ip address` da `Serial0/3/0` aqui. Confirme com
> `show controllers Serial0/3/0`.

## Passo 14 — Atualizar o Roteador R-SP (novo link + nova rede no OSPF)

Volte no R-SP e adicione a nova interface e o novo `network` no OSPF (não mexe no
que já existe, só acrescenta):

```
enable
configure terminal
interface Serial0/3/1
 ip address 192.168.101.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 network 192.168.101.0 0.0.0.3 area 0
end
write memory
```

> Aqui coloquei `clock rate` no lado de SP (fica o DCE deste link) — se o Packet
> Tracer decidir diferente, confirme com `show controllers Serial0/3/1` e mova o
> comando pro lado certo.

## Passo 15 — Verificação final com as 3 filiais

No R-RO, R-SP e R-MG:

```
show ip interface brief
show ip ospf neighbor
show ip route ospf
```

**O que checar:**
- No **R-SP**, `show ip ospf neighbor` deve mostrar **dois** vizinhos agora: RO
  (`1.1.1.1`) e MG (`3.3.3.3`), ambos em **FULL**.
- No **R-RO**, `show ip route ospf` deve mostrar rotas para as redes de MG
  (`192.168.50.0/24`, `192.168.60.0/24`) **via SP** — prova de que o OSPF calculou
  o caminho de 2 saltos sozinho, sem você precisar configurar rota nenhuma "manual".
- Mesma coisa no **R-MG**, enxergando as redes de RO via SP.

Teste de ping fim a fim (RO → MG, passando por dentro de SP):

```
ping 192.168.50.10   ! de um PC da VLAN 10 (RO) até a VLAN 50 (MG)
```

Se o ping funcionar mesmo sem RO e MG terem link direto, é a prova visual de que
o roteamento dinâmico (OSPF) está funcionando — ele "descobriu" o caminho sozinho.

---

# SEÇÃO DE COMANDOS DE TESTE (Checklist Completo)

Use esta seção como roteiro único de testes na hora de apresentar — rode na
ordem, em cada roteador/switch indicado.

## A. Testes nos Switches (SW-RO, SW-SP, SW-MG)

```
show vlan brief
show interfaces trunk
show interfaces status
```

- `show vlan brief` → confirma que as VLANs existem e quais portas pertencem a cada uma
- `show interfaces trunk` → confirma que a porta pro roteador está em modo **trunk** e passando as VLANs certas
- `show interfaces status` → mostra o status físico (connected/notconnect) de cada porta

## B. Testes nos Roteadores (R-RO, R-SP, R-MG)

### B.1 — Interfaces (sempre primeiro)

```
show ip interface brief
```

Checklist: `Gi0/0`, sub-interfaces (`.10/.20`, `.30/.40` ou `.50/.60`) e a(s)
interface(s) `Serial` devem estar **up / up**. Qualquer coisa diferente disso,
resolva antes de seguir para os testes de OSPF.

### B.2 — OSPF

```
show ip protocols
show ip ospf neighbor
show ip ospf database
show ip route ospf
```

| Comando | O que confirmar |
|---|---|
| `show ip protocols` | O processo OSPF 1 está ativo e as redes certas estão sendo anunciadas |
| `show ip ospf neighbor` | Vizinhos aparecem com estado **FULL** (no R-SP devem aparecer **2** vizinhos: RO e MG) |
| `show ip ospf database` | Mostra o "mapa" da rede que o OSPF construiu (LSAs) |
| `show ip route ospf` | Lista só as rotas aprendidas via OSPF (marcadas com `O` na tabela de rotas) |

### B.3 — Tabela de roteamento completa

```
show ip route
```

Use esse quando quiser mostrar a tabela **inteira** (rotas diretas `C`, conectadas
`L` e aprendidas `O`), não só as do OSPF.

## C. Testes de conectividade (a partir dos PCs)

Abra o **Desktop → Command Prompt** de cada PC.

### C.1 — Ping dentro da própria filial (mesma VLAN)

```
ping 192.168.10.1     ! PC da VLAN 10 pingando o próprio gateway (R-RO)
```

### C.2 — Ping entre filiais vizinhas (1 salto)

```
ping 192.168.30.10    ! de um PC da VLAN 10 (RO) até a VLAN 30 (SP)
ping 192.168.50.10    ! de um PC da VLAN 30 (SP) até a VLAN 50 (MG)
```

### C.3 — Ping entre filiais nas pontas (2 saltos, passando por SP)

```
ping 192.168.50.10    ! de um PC da VLAN 10 (RO) até a VLAN 50 (MG)
ping 192.168.60.10    ! de um PC da VLAN 20 (RO) até a VLAN 60 (MG)
```

### C.4 — Traceroute (mostra o caminho salto a salto — ótimo pra explicar na aula)

```
tracert 192.168.50.10
```

No PC da VLAN 10 (RO), o `tracert` até a VLAN 50 (MG) deve mostrar **2 saltos**:
primeiro o `192.168.100.2` (R-SP), depois o `192.168.50.10` (destino) — provando
visualmente que o tráfego passou por dentro de SP.

## D. Tabela-resumo do que cada teste prova

| Teste | O que prova |
|---|---|
| `show vlan brief` | VLANs criadas e portas associadas corretamente |
| `show ip interface brief` (up/up) | Camada física + lógica (IP) funcionando |
| `show ip ospf neighbor` (FULL) | Adjacência OSPF formada entre os roteadores |
| `show ip route ospf` | Roteador aprendeu dinamicamente as redes remotas |
| `ping` entre VLANs de filiais diferentes | Roteamento fim a fim funcionando |
| `tracert` | Visualiza o caminho multi-hop (RO → SP → MG) calculado pelo OSPF |

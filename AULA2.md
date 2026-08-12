# PARTE 3 — OSPF Multi-Área + 2ª Cidade em cada Estado

Continuação de [aula.md](aula.md). Pré-requisito: Partes 1 e 2 já montadas e
funcionando (RO ↔ SP ↔ MG, área única, OSPF FULL entre os três).

## 1. Novo desenho lógico

**SP vira o backbone (Area 0).** RO e MG viram áreas próprias, e cada estado
ganha uma segunda cidade dentro da **mesma área** do seu estado.

```
                         ÁREA 0 (backbone)
                    R-SP ─────────────┐
                   /    \              \
                  /      \              \
        Se0/3/0  /        \ Se0/3/1      \ Se0/1/0 (2º módulo)
                 /          \              \
            R-RO             R-MG          R-SP2
         (ABR 1|0)         (ABR 2|0)      (Area 0)
              |                 |              |
          ÁREA 1            ÁREA 2         ÁREA 0
              |                 |              |
        Se0/3/1 (novo)     Se0/3/1 (novo)      |
              |                 |          VLAN 31/41
            R-RO2             R-MG2         Vendas/Adm SP2
         (Area 1)           (Area 2)
              |                 |
        VLAN 11/21         VLAN 51/61
      Vendas/Adm RO2      Vendas/Adm MG2
```

- **R-RO** e **R-MG** são **ABRs**: uma perna em Area 0 (link serial para SP),
  outra perna em Area 1/2 (LAN do estado + link serial para a 2ª cidade).
- **R-SP** fica inteiro em Area 0 (é o backbone).
- **R-RO2, R-SP2, R-MG2** são roteadores internos de área — nenhum deles tem pé
  em duas áreas, então não são ABR.

## 2. Plano de Endereçamento IP (completo, Partes 1+2+3)

| Estado | Cidade | VLAN | Nome | Rede | Gateway | Área OSPF |
|---|---|---|---|---|---|---|
| RO | Capital (RO)   | 10 | Vendas-RO  | 192.168.10.0/24 | 192.168.10.1 | 1 |
| RO | Capital (RO)   | 20 | Adm-RO     | 192.168.20.0/24 | 192.168.20.1 | 1 |
| RO | Cidade 2 (RO2) | 11 | Vendas-RO2 | 192.168.11.0/24 | 192.168.11.1 | 1 |
| RO | Cidade 2 (RO2) | 21 | Adm-RO2    | 192.168.21.0/24 | 192.168.21.1 | 1 |
| SP | Capital (SP)   | 30 | Vendas-SP  | 192.168.30.0/24 | 192.168.30.1 | 0 |
| SP | Capital (SP)   | 40 | Adm-SP     | 192.168.40.0/24 | 192.168.40.1 | 0 |
| SP | Cidade 2 (SP2) | 31 | Vendas-SP2 | 192.168.31.0/24 | 192.168.31.1 | 0 |
| SP | Cidade 2 (SP2) | 41 | Adm-SP2    | 192.168.41.0/24 | 192.168.41.1 | 0 |
| MG | Capital (MG)   | 50 | Vendas-MG  | 192.168.50.0/24 | 192.168.50.1 | 2 |
| MG | Capital (MG)   | 60 | Adm-MG     | 192.168.60.0/24 | 192.168.60.1 | 2 |
| MG | Cidade 2 (MG2) | 51 | Vendas-MG2 | 192.168.51.0/24 | 192.168.51.1 | 2 |
| MG | Cidade 2 (MG2) | 61 | Adm-MG2    | 192.168.61.0/24 | 192.168.61.1 | 2 |

| Link WAN | Rede | Área OSPF | DCE (clock rate) |
|---|---|---|---|
| RO ↔ SP   | 192.168.100.0/30 | 0 | R-RO |
| SP ↔ MG   | 192.168.101.0/30 | 0 | R-SP |
| RO ↔ RO2  | 192.168.110.0/30 | 1 | R-RO |
| SP ↔ SP2  | 192.168.120.0/30 | 0 | R-SP |
| MG ↔ MG2  | 192.168.130.0/30 | 2 | R-MG |

Router-IDs (padrão `<área-estado>.<área-estado>.<sequência-cidade>.<sequência-cidade>`,
só pra ficar fácil de identificar em `show ip ospf neighbor`):

| Roteador | Router-ID |
|---|---|
| R-RO  | 1.1.1.1 |
| R-RO2 | 1.1.1.2 |
| R-SP  | 2.2.2.2 |
| R-SP2 | 2.2.2.3 |
| R-MG  | 3.3.3.3 |
| R-MG2 | 3.3.3.4 |

## 3. Equipamento adicional

| Papel | Modelo | Detalhe |
|---|---|---|
| Roteador (3x novos: R-RO2, R-SP2, R-MG2) | **Cisco 2901** | + **HWIC-2T** no slot 3, mesmo processo do Passo 2/11 |
| Switch (3x novos: SW-RO2, SW-SP2, SW-MG2) | **Cisco 2960-24TT-L** | igual às filiais já existentes |
| **2º módulo HWIC-2T em R-SP** | — | R-SP já usa as duas portas seriais do primeiro módulo (RO e MG). Pra falar com SP2 precisa de um **segundo HWIC-2T em outro slot**. As interfaces novas costumam nascer como `Serial0/1/0` / `Serial0/1/1` — confirme sempre com `show ip interface brief` |

> R-RO e R-MG **não precisam de módulo novo**: cada um já tem uma porta serial
> livre no HWIC-2T que instalaram na Parte 2 (`Serial0/3/1`), que sobra livre
> pra ligar na respectiva cidade 2.

---

# PASSO A PASSO

## Passo 16 — Retrofit: mover RO e MG para áreas próprias

Antes de adicionar as cidades novas, ajuste o OSPF que já existe. **Só troca o
número da área nos `network` já configurados — não mexe em mais nada.**

No **R-RO**:

```
enable
configure terminal
router ospf 1
 no network 192.168.10.0 0.0.0.255 area 0
 no network 192.168.20.0 0.0.0.255 area 0
 network 192.168.10.0 0.0.0.255 area 1
 network 192.168.20.0 0.0.0.255 area 1
end
write memory
```

No **R-MG**:

```
enable
configure terminal
router ospf 1
 no network 192.168.50.0 0.0.0.255 area 0
 no network 192.168.60.0 0.0.0.255 area 0
 network 192.168.50.0 0.0.0.255 area 2
 network 192.168.60.0 0.0.0.255 area 2
end
write memory
```

> Repare que o link serial `192.168.100.0/30` (em R-RO) e `192.168.101.0/30` (em
> R-MG) **continuam em area 0** — são eles que ligam a área do estado ao
> backbone. R-SP não muda em nada neste passo.

Verificação imediata:

```
show ip ospf neighbor
show ip route ospf
```

As rotas que antes apareciam como `O` agora aparecem como **`O IA`**
(inter-area) em R-RO, R-SP e R-MG — essa é a prova visual de que a rede virou
multi-área.

## Passo 17 — Montar a topologia física das 3 cidades novas

1. Adicione 3 roteadores **2901** (`R-RO2`, `R-SP2`, `R-MG2`), 3 switches
   **2960-24TT-L** e 2 PCs por cidade nova (1 Vendas, 1 Adm).
2. Cabos:
   - `R-RO2 Gi0/0` ↔ `SW-RO2 Gi0/1`, `R-SP2 Gi0/0` ↔ `SW-SP2 Gi0/1`,
     `R-MG2 Gi0/0` ↔ `SW-MG2 Gi0/1` — Copper Straight-Through
   - PCs ↔ `Fa0/1-20` dos switches novos — Copper Straight-Through
   - `R-RO Se0/3/1` ↔ `R-RO2 Se0/3/0` — **Serial DCE** (lado R-RO)
   - `R-SP <2ª serial do 2º módulo>` ↔ `R-SP2 Se0/3/0` — **Serial DCE** (lado R-SP)
   - `R-MG Se0/3/1` ↔ `R-MG2 Se0/3/0` — **Serial DCE** (lado R-MG)

## Passo 18 — Instalar módulos HWIC-2T

1. Em `R-RO2`, `R-SP2` e `R-MG2`: mesmo processo do Passo 2 (desligar na aba
   Physical, arrastar HWIC-2T pra um slot vazio, ligar de novo).
2. Em `R-SP`: desligue, arraste um **2º HWIC-2T** para outro slot vazio (sem
   mexer no módulo já instalado), ligue de novo.
3. Em todos, confira o nome real das interfaces seriais criadas:
   ```
   enable
   show ip interface brief
   ```
   (se o 2º módulo do R-SP não gerar `Serial0/1/0`, use o nome que aparecer no
   lugar de todos os comandos abaixo referentes a esse link)

## Passo 19 — Configurar o Switch SW-RO2

```
enable
configure terminal
hostname SW-RO2
vlan 11
 name Vendas-RO2
vlan 21
 name Adm-RO2
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 11,21
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 11
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 21
end
write memory
```

## Passo 20 — Configurar o Roteador R-RO2

```
enable
configure terminal
hostname R-RO2
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.11
 encapsulation dot1Q 11
 ip address 192.168.11.1 255.255.255.0
interface GigabitEthernet0/0.21
 encapsulation dot1Q 21
 ip address 192.168.21.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.110.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 1.1.1.2
 network 192.168.11.0 0.0.0.255 area 1
 network 192.168.21.0 0.0.0.255 area 1
 network 192.168.110.0 0.0.0.3 area 1
end
write memory
```

## Passo 21 — Atualizar o Roteador R-RO (novo link para RO2)

```
enable
configure terminal
interface Serial0/3/1
 ip address 192.168.110.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 network 192.168.110.0 0.0.0.3 area 1
end
write memory
```

## Passo 22 — Configurar o Switch SW-MG2

```
enable
configure terminal
hostname SW-MG2
vlan 51
 name Vendas-MG2
vlan 61
 name Adm-MG2
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 51,61
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 51
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 61
end
write memory
```

## Passo 23 — Configurar o Roteador R-MG2

```
enable
configure terminal
hostname R-MG2
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.51
 encapsulation dot1Q 51
 ip address 192.168.51.1 255.255.255.0
interface GigabitEthernet0/0.61
 encapsulation dot1Q 61
 ip address 192.168.61.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.130.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 3.3.3.4
 network 192.168.51.0 0.0.0.255 area 2
 network 192.168.61.0 0.0.0.255 area 2
 network 192.168.130.0 0.0.0.3 area 2
end
write memory
```

## Passo 24 — Atualizar o Roteador R-MG (novo link para MG2)

```
enable
configure terminal
interface Serial0/3/1
 ip address 192.168.130.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 network 192.168.130.0 0.0.0.3 area 2
end
write memory
```

## Passo 25 — Configurar o Switch SW-SP2

```
enable
configure terminal
hostname SW-SP2
vlan 31
 name Vendas-SP2
vlan 41
 name Adm-SP2
exit
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 31,41
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 31
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 41
end
write memory
```

## Passo 26 — Configurar o Roteador R-SP2

```
enable
configure terminal
hostname R-SP2
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.31
 encapsulation dot1Q 31
 ip address 192.168.31.1 255.255.255.0
interface GigabitEthernet0/0.41
 encapsulation dot1Q 41
 ip address 192.168.41.1 255.255.255.0
interface Serial0/3/0
 ip address 192.168.120.2 255.255.255.252
 no shutdown
router ospf 1
 router-id 2.2.2.3
 network 192.168.31.0 0.0.0.255 area 0
 network 192.168.41.0 0.0.0.255 area 0
 network 192.168.120.0 0.0.0.3 area 0
end
write memory
```

## Passo 27 — Atualizar o Roteador R-SP (novo link para SP2)

Use o nome real da interface do 2º módulo (confirmado no Passo 18):

```
enable
configure terminal
interface Serial0/1/0
 ip address 192.168.120.1 255.255.255.252
 clock rate 64000
 no shutdown
router ospf 1
 network 192.168.120.0 0.0.0.3 area 0
end
write memory
```

## Passo 28 — Configurar os PCs das cidades novas

| Cidade | VLAN | IP | Máscara | Gateway |
|---|---|---|---|---|
| RO2 | 11 | 192.168.11.10 | 255.255.255.0 | 192.168.11.1 |
| RO2 | 21 | 192.168.21.10 | 255.255.255.0 | 192.168.21.1 |
| SP2 | 31 | 192.168.31.10 | 255.255.255.0 | 192.168.31.1 |
| SP2 | 41 | 192.168.41.10 | 255.255.255.0 | 192.168.41.1 |
| MG2 | 51 | 192.168.51.10 | 255.255.255.0 | 192.168.51.1 |
| MG2 | 61 | 192.168.61.10 | 255.255.255.0 | 192.168.61.1 |

## Passo 29 — Verificação final multi-área

Em **todos os 6 roteadores**:

```
show ip interface brief
show ip ospf neighbor
show ip route ospf
show ip ospf border-routers
```

**O que checar:**

- `show ip ospf neighbor`:
  - R-SP deve ter **3** vizinhos FULL: R-RO, R-MG, R-SP2 (todos em area 0)
  - R-RO deve ter **2** vizinhos: R-SP (area 0) e R-RO2 (area 1)
  - R-MG deve ter **2** vizinhos: R-SP (area 0) e R-MG2 (area 2)
- `show ip route ospf`:
  - Rotas dentro da mesma área aparecem como `O`
  - Rotas cruzando área (ex.: R-RO2 enxergando a rede de MG2) aparecem como
    **`O IA`** — prova de que passaram por um ABR
- `show ip ospf border-routers` (rodar em qualquer roteador): lista R-RO e R-MG
  como **ABR** — confirma o desenho da Area 0/1/2

Teste de ping/traceroute ponta a ponta (o mais completo do laboratório — RO2 até
MG2, atravessando 2 ABRs e o backbone inteiro):

```
ping 192.168.51.10        ! de um PC da VLAN 11 (RO2) até a VLAN 51 (MG2)
tracert 192.168.51.10
```

O `tracert` deve mostrar 4 saltos: `R-RO` → `R-SP` → `R-MG` → `192.168.51.10`,
provando visualmente que o tráfego atravessou Area 1 → Area 0 → Area 2.

## Passo 30 — Troubleshooting específico de multi-área

| Sintoma | Causa | Solução |
|---|---|---|
| Vizinho OSPF não forma entre R-RO e R-RO2 | Área errada em algum lado do link `192.168.110.0/30` | As duas pontas do mesmo link **têm que estar na mesma área** — confirme com `show ip protocols` em ambos |
| `show ip route ospf` não mostra `O IA` em lugar nenhum | Esqueceu o Passo 16 (RO/MG ainda em area 0) | Rodar `show ip ospf interface brief` e conferir a coluna Area de cada interface |
| `show ip ospf border-routers` vazio | Nenhum roteador tem interface em duas áreas diferentes | Conferir se R-RO e R-MG realmente têm uma rede em area 0 (link pra SP) e outra em area 1/2 (LAN) |
| `%OSPF: Interface serialX/Y/Z has same area as neighbor's other interface` (ou vizinho preso em EXSTART) | MTU ou área diferente entre as pontas do link novo | Conferir `network ... area N` idêntico nas duas pontas do mesmo link ponto-a-ponto |
| Nova interface serial do 2º HWIC-2T do R-SP não aparece como `Serial0/1/0` | Simulador numerou diferente conforme o slot escolhido | Sempre confirmar com `show ip interface brief` antes de configurar, e ajustar o nome em todos os comandos do Passo 27 |

---

# Tabela-resumo (o que cada novo teste prova)

| Teste | O que prova |
|---|---|
| `network ... area 1/2` em R-RO/R-MG | Área do estado separada do backbone |
| `show ip route ospf` com `O IA` | Rota aprendida através de um ABR (inter-area) |
| `show ip ospf border-routers` | Identifica quem são os ABRs da rede (R-RO e R-MG) |
| `show ip ospf neighbor` (3 vizinhos em R-SP) | Backbone concentrando RO, MG e SP2 |
| `tracert` RO2 → MG2 (4 saltos) | Caminho completo Area 1 → Area 0 → Area 2 calculado pelo OSPF sozinho |

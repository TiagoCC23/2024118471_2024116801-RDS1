# Testes STP — Projeto de Redes (RDS1)

Ordem sugerida para a defesa: **0 → 1 → 2 → 3 → 4 → 5 → 6**.

## Mapa rápido

| Switch | Po1 (e0/0-1) | Po2 (e0/2-3) | e1/0 | e1/1 |
|---|---|---|---|---|
| A1 | → CD1 | → CD2 | PC1 (VLAN 10) | PC4 (VLAN 10) |
| A2 | → CD1 | → CD2 | PC2 (VLAN 20) | PC5 (VLAN 20) |
| A3 | → CD1 | → CD2 | PC3 (VLAN 30) | Management (VLAN 99) |

Roots: **CD1** = VLAN 10 e 30 | **CD2** = VLAN 20 e 99.

---

## 0. Antes de começar (guardar tudo)

Em **todos** os switches (A1, A2, A3, CD1, CD2):

```
write memory
```

**Esperado:**
```
Building configuration...
[OK]
```

---

## 1. CD1 — modo e root

```
show spanning-tree summary
```
**Esperado:**
- `Switch is in rapid-pvst mode`
- `Root bridge for: VLAN0010, VLAN0030`

```
show spanning-tree vlan 10
```
**Esperado:**
- `Spanning tree enabled protocol rstp`
- `This bridge is the root`
- Bridge ID Priority **24586** (24576 + 10)
- Todas as portas `Desg FWD`

```
show spanning-tree vlan 20
```
**Esperado:**
- **Não** aparece "This bridge is the root"
- Root ID Priority **24596** (24576 + 20), com o MAC do CD2
- Uma porta `Root FWD` (o port-channel para o CD2)
- Bridge ID Priority **28692** (28672 + 20, root secondary)

```
show etherchannel summary
```
**Esperado:** port-channels com flag `(SU)`, protocolo `LACP`, portas com `(P)`.

---

## 2. CD2 — modo e root

```
show spanning-tree summary
```
**Esperado:**
- `Switch is in rapid-pvst mode`
- `Root bridge for: VLAN0020, VLAN0099`

```
show spanning-tree vlan 20
```
**Esperado:** `This bridge is the root`, Priority **24596**, portas `Desg FWD`.

```
show spanning-tree vlan 10
```
**Esperado:**
- **Não** é root
- Root ID Priority **24586** (CD1)
- Bridge ID Priority **28682** (28672 + 10, root secondary)
- Uma porta `Root FWD` (para o CD1)

---

## 3. A1 — acesso, PortFast e BPDU Guard

```
show spanning-tree summary
```
**Esperado:** `Switch is in rapid-pvst mode`.

```
show spanning-tree vlan 10
```
**Esperado:**
- Root ID = CD1 (Priority 24586)
- `Po1  Root FWD` (caminho para o CD1)
- `Po2  Altn BLK` (caminho de reserva para o CD2, bloqueado)
- As portas `Et1/0` e `Et1/1` em `Desg FWD ... P2p Edge`

```
show spanning-tree vlan 20
```
**Esperado:** Root ID = CD2 (Priority 24596), com `Po2  Root FWD` e `Po1  Altn BLK`. É isto que prova o **balanceamento de carga**.

```
show spanning-tree interface e1/0 portfast
```
**Esperado:** `VLAN0010  enabled`

```
show spanning-tree interface e1/0 detail
```
**Esperado (linhas relevantes):**
```
The port is in the portfast mode
Bpdu guard is enabled
```

```
show interfaces trunk
```
**Esperado:** Po1 e Po2 em `802.1q`, `trunking`, com as VLANs 10, 20, 30 e 99 permitidas e ativas.

```
show vlan brief
```
**Esperado:** VLANs 10 Guest, 20 Student, 30 Staff, 99 Management criadas; `Et1/0` e `Et1/1` na **VLAN 10**.

```
show interfaces status err-disabled
```
**Esperado:** vazio (nenhuma porta desligada por BPDU Guard).

> **A2:** repete com VLAN 20 nas e1/0-1. **A3:** e1/0 na VLAN 30 e e1/1 na VLAN 99.

---

## 4. Testes de ping

### 4.1 Configurar IPs nos PCs (dentro de cada container)

Se ainda não tiverem IP. Endereços do VLSM (10.50.0.0/20):

| VLAN | Rede | Máscara |
|---|---|---|
| 20 | 10.50.0.0/25 | /25 |
| 10 | 10.50.0.128/26 | /26 |
| 30 | 10.50.0.192/27 | /27 |
| 99 | 10.50.0.224/28 | /28 |

```
# PC1
ip addr add 10.50.0.130/26 dev eth0
ip link set eth0 up
# PC4
ip addr add 10.50.0.131/26 dev eth0
ip link set eth0 up
# PC2
ip addr add 10.50.0.2/25 dev eth0
ip link set eth0 up
# PC5
ip addr add 10.50.0.3/25 dev eth0
ip link set eth0 up
# PC3
ip addr add 10.50.0.194/27 dev eth0
ip link set eth0 up
# Server (Debian, como root: sudo -s)
ip addr add 10.50.0.195/27 dev ens4
ip link set ens4 up
# Management
ip addr add 10.50.0.226/28 dev eth0
ip link set eth0 up
```

Se já tiverem IPs atribuídos, usa esses e ignora este passo. Confirma com `ip a`.

### 4.2 Pings dentro da mesma VLAN (devem funcionar)

```
# No PC1
ping -c 4 10.50.0.131
```
**Esperado:**
```
4 packets transmitted, 4 received, 0% packet loss
```

```
# No PC2
ping -c 4 10.50.0.3
```
**Esperado:** `0% packet loss` (VLAN 20, PC2 ↔ PC5).

```
# No PC3
ping -c 4 10.50.0.195
```
**Esperado:** `0% packet loss` (VLAN 30, PC3 ↔ Server). Só funciona se as portas do servidor nos CD1/CD2 estiverem em acesso VLAN 30.

A primeira resposta pode demorar mais (resolução ARP); é normal.

### 4.3 Pings entre VLANs diferentes (devem falhar nesta fase)

```
# No PC1
ping -c 4 10.50.0.2
```
**Esperado:**
```
4 packets transmitted, 0 received, 100% packet loss
```
(ou `Destination Host Unreachable`).

Isto é o **comportamento correto**: as VLANs são domínios de broadcast separados e ainda não há routing inter-VLAN (SVIs nos CDs). Se o professor perguntar, é este o motivo.

---

## 5. Demonstração de falha (redundância)

Deixa um ping contínuo no PC1 (`ping 10.50.0.131`) e, no **A1**:

```
configure terminal
interface port-channel 1
 shutdown
end
show spanning-tree vlan 10
```
**Esperado:**
- O `Po2` passa de `Altn BLK` a `Root FWD` (convergência em poucos segundos com Rapid-PVST)
- O ping tem no máximo 1 ou 2 perdas e **recupera sozinho**
- Root ID continua a ser o CD1 (chega por CD2 → CD1 através do link entre os dois)

Repor:
```
configure terminal
interface port-channel 1
 no shutdown
end
show spanning-tree vlan 10
```
**Esperado:** `Po1` volta a `Root FWD` e `Po2` a `Altn BLK`.

---

## 6. (Opcional) Demonstrar o BPDU Guard

Só se o professor pedir. Liga temporariamente um switch a uma porta de acesso ou, mais simples, mostra a configuração:

```
show running-config interface e1/0
```
**Esperado:**
```
interface Ethernet1/0
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
```

Se uma porta for desligada por BPDU Guard, aparece:
```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Et1/0 with BPDU Guard enabled. Disabling port.
```
e `show interfaces status err-disabled` mostra a porta como `err-disabled`. Recupera-se com `shutdown` seguido de `no shutdown`.

---

## Cola de respostas (30 segundos)

- **O que faz o STP?** Evita ciclos na camada 2 bloqueando caminhos redundantes.
- **Porquê Rapid-PVST+?** Uma instância por VLAN e convergência em segundos, o que permite ter roots diferentes por VLAN.
- **Porque o CD1 e o CD2 têm roots diferentes?** Balanceamento de carga: cada um é root de metade das VLANs e secundário da outra metade.
- **O que é uma porta Altn BLK?** O caminho de reserva, bloqueado, que ativa se o principal falhar.
- **PortFast?** A porta de acesso vai logo para forwarding.
- **BPDU Guard?** Desliga a porta se receber BPDUs, protege contra switches ligados por engano.
- **Porque não pingas entre VLANs?** Falta routing inter-VLAN, que é outra fase.

> Os valores exatos (MAC, custos, nomes dos Po) podem diferir no teu GNS3. O importante é: **rapid-pvst**, **This bridge is the root** nos sítios certos, e **Root FWD / Altn BLK** nos switches de acesso.

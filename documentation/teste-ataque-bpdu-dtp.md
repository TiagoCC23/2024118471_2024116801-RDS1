# Teste de ataques STP e DTP no GNS3

Este guia descreve como demonstrar, a partir do PC3, a tentativa de assumir a Root Bridge por STP e a tentativa de negociar um trunk por DTP. A ligação do PC3 é `A3 Ethernet1/0`.

## Preparação

O PC3 do projeto usa uma imagem de utilitários que pode não incluir Yersinia. Usa uma VM Kali ligada temporariamente à porta `A3 Ethernet1/0` ou instala Yersinia no PC3, se o contentor tiver acesso aos repositórios:

```bash
sudo apt update
sudo apt install yersinia
ip -br link
```

Identifica a interface ligada ao A3 (normalmente `eth0`). Executa os comandos seguintes nessa interface, com privilégios de administrador. Yersinia implementa ataques STP e DTP; consulta o [manual do Yersinia](https://github.com/tomac/yersinia/blob/master/yersinia.8) e a [página da ferramenta no Kali](https://www.kali.org/tools/yersinia/).

## 1. Confirmar a configuração em A1, A2 e A3

Nas consolas de cada switch, verifica as portas de acesso `Ethernet1/0` e `Ethernet1/1`:

```text
show running-config interface Ethernet1/0
show running-config interface Ethernet1/1
show interfaces Ethernet1/0 switchport
show interfaces Ethernet1/1 switchport
show spanning-tree interface Ethernet1/0 detail
show spanning-tree interface Ethernet1/1 detail
```

Em cada porta, procura `switchport mode access`, `switchport nonegotiate`, `spanning-tree bpduguard enable` e `spanning-tree guard root`. A saída de `show interfaces ... switchport` deve indicar modo administrativo access e negociação DTP desligada. A porta deve estar ligada e não estar `err-disabled` nem `root-inconsistent` antes dos testes.

## 2. Tentar negociar um trunk por DTP

Executa no PC3/Kali:

```bash
sudo yersinia dtp -attack 1 -interface eth0
```

O ataque 1 do Yersinia tenta estabelecer trunk. Deixa correr alguns segundos e interrompe com `Ctrl+C`. No A3, verifica:

```text
show dtp interface Ethernet1/0
show interfaces Ethernet1/0 switchport
show interfaces trunk
```

Resultado esperado: negociação DTP desligada, interface operacional em access (VLAN 30) e sem `Ethernet1/0` na lista de trunks.

## 3. Injetar BPDUs para testar BPDU Guard

Executa no PC3/Kali:

```bash
sudo yersinia stp -attack 4 -interface eth0
```

O ataque 4 do Yersinia é “Claiming Root Role”. Com BPDU Guard ativo, o esperado é a porta ser desativada ao receber a BPDU, antes de uma nova Root Bridge ser eleita. Interrompe o Yersinia com `Ctrl+C` e confirma no A3:

```text
show logging | include BPDU|ERR_DISABLE
show interfaces status
show spanning-tree vlan 30 root
```

O log deve indicar BPDU Guard e `Ethernet1/0` deve aparecer `err-disabled`. A Root Bridge legítima deve manter-se. Recupera a porta com os comandos da secção 5 antes do teste seguinte.

## 4. Testar Root Guard isoladamente

BPDU Guard desativa a porta ao receber qualquer BPDU, por isso não permite observar a reação própria do Root Guard. Faz este teste apenas na porta de laboratório `A3 Ethernet1/0`, ligada ao PC3:

1. Na consola A3, desativa BPDU Guard temporariamente, mantendo Root Guard ativo:

   ```text
   configure terminal
   interface Ethernet1/0
   no spanning-tree bpduguard enable
   end
   ```

2. No PC3/Kali, volta a executar o ataque STP:

   ```bash
   sudo yersinia stp -attack 4 -interface eth0
   ```

3. Na consola A3, confirma a proteção Root Guard:

   ```text
   show spanning-tree inconsistentports
   show spanning-tree vlan 30 interface Ethernet1/0 detail
   show spanning-tree vlan 30 root
   ```

   Resultado esperado durante o ataque: `Ethernet1/0` aparece `root-inconsistent`/inconsistent e a Root Bridge legítima não muda. Interrompe o Yersinia com `Ctrl+C`.

4. Repõe BPDU Guard imediatamente e confirma a configuração:

   ```text
   configure terminal
   interface Ethernet1/0
   spanning-tree bpduguard enable
   end
   show running-config interface Ethernet1/0
   ```

   Se a porta ficar `err-disabled`, recupera-a como descrito abaixo. Confirma que voltou a `connected` e que não permanece `root-inconsistent`.

## 5. Recuperar a porta após o teste

Depois de interromper o ataque, repõe a porta no A3:

```text
configure terminal
interface Ethernet1/0
shutdown
no shutdown
end
```

Confirma que voltou a `connected` e que a proteção permanece configurada:

```text
show interfaces status
show running-config interface Ethernet1/0
```

Executa estes testes apenas na porta de acesso do PC3, nunca num uplink. Depois de todos os testes, repete as verificações da secção 1 em A1, A2 e A3 para confirmar que as configurações foram repostas.

## Resultado do teste já realizado

O teste efetuado neste projeto usou tramas geradas inline no contentor do PC3; não usou Yersinia. Três BPDUs fizeram o A3 registar `%SPANTREE-2-BLOCK_BPDUGUARD` e colocar `Ethernet1/0` em `err-disabled`. Cinco tramas DTP foram recebidas e descartadas; a porta permaneceu access na VLAN 30, com a negociação DTP desligada e sem trunk. A Root Bridge da VLAN 30 manteve-se `aabb.cc00.0100`. A porta foi reposta e terminou ligada.

O teste isolado de Root Guard feito anteriormente não foi conclusivo: a BPDU PVST+ usada foi bloqueada por inconsistência de tipo de porta, sem gerar `root-inconsistent`. O procedimento da secção 4 descreve a tentativa com Yersinia; confirma a saída real do switch durante a apresentação.

# Troubleshooting — Registro de Incidentes

Este documento reúne todos os problemas reais enfrentados durante o desenvolvimento do projeto ProDesk, em ordem cronológica, cobrindo as duas fases: o design original da rede (unidade curricular *Ambientes Computacionais e Conectividade*) e o hardening de segurança realizado posteriormente.

Cada incidente segue a estrutura: **sintoma → diagnóstico → correção**, priorizando o raciocínio até a causa raiz, não apenas o comando final.

---

## Fase 1 — Design original da rede

### 1.1 Vizinhança OSPF — dúvida entre Router ID e endereço do enlace

**Sintoma:** ao analisar `show ip ospf neighbor`, o endereço do vizinho aparecia como `10.0.0.2` (endereço da interface no link `/30`), enquanto o Router ID exibido era algo como `192.168.40.1` — gerando dúvida se isso indicava erro de configuração.

**Diagnóstico:** Router ID e endereço do enlace **não precisam ser iguais**. O Router ID é um identificador lógico do processo OSPF (por padrão, o maior IP configurado no roteador no momento em que o processo OSPF sobe, a menos que se defina manualmente), enquanto o endereço do vizinho reportado é o IP da interface usada para formar a adjacência.

**Resultado:** não havia erro — apenas necessidade de entender a diferença conceitual entre os dois campos.

---

### 1.2 Subinterfaces com `protocol down`

**Sintoma:** subinterfaces dos roteadores (`GigabitEthernet0/1.X`) apresentavam `Status: up`, mas `Protocol: down`.

**Diagnóstico:** investigação de compatibilidade entre a configuração do roteador (`encapsulation dot1Q`) e a porta correspondente no switch. Em uma topologia router-on-a-stick, a interface física do roteador e a porta trunk do switch conectada a ela precisam estar configuradas de forma compatível — verificação de encapsulamento, VLANs permitidas, estado físico da porta e configuração do switch do outro lado.

**Correção:** ajuste de encapsulamento e configuração trunk até a subinterface subir com `Protocol: up`.

---

### 1.3 Conflito de endereço IP na VLAN 99

**Sintoma:** durante reorganização da infraestrutura, foi identificado que um endereço na VLAN 99 já estava em uso por outro dispositivo.

**Diagnóstico:** violação da regra fundamental de que cada interface L3 numa mesma rede deve ter endereço único. O conflito envolvia a faixa `192.168.99.0/26` (gateway `.1`, servidor `.10`).

**Correção:** reatribuição de endereço único a cada dispositivo, eliminando a sobreposição.

---

### 1.4 Dúvida arquitetural — switches de distribuição devem conhecer VLANs de outro prédio?

**Sintoma:** durante o planejamento, surgiu a dúvida se os switches de distribuição de cada prédio deveriam ter conhecimento das VLANs do prédio remoto.

**Diagnóstico/decisão:** um switch não precisa carregar indiscriminadamente todas as VLANs da empresa. Cada prédio transporta localmente apenas suas próprias VLANs; a comunicação entre prédios acontece via roteadores, que fazem o roteamento entre as diferentes redes:

```text
VLAN local → Roteador → Link entre prédios → Roteador → VLAN do outro prédio
```

**Resultado:** não é necessário estender uma camada 2 única atravessando os dois prédios — decisão que reforçou a arquitetura roteada entre unidades.

---

### 1.5 Tentativa de switches Layer 3 na distribuição (abordagem descartada)

**Sintoma:** ao testar mover o roteamento inter-VLAN para os switches de distribuição (via `ip routing`, SVIs e interfaces VLAN), surgiram múltiplos problemas: interfaces em `protocol down`, conflitos de endereçamento, dificuldade de comunicação estável com os roteadores e com o OSPF.

**Diagnóstico:** a complexidade adicional introduzida (múltiplos pontos de roteamento, SVIs, redistribuição) não se estabilizou no cenário implementado no Packet Tracer dentro do tempo disponível.

**Correção/decisão final:** reversão para a arquitetura com switches L2 puros na distribuição e roteamento centralizado nos roteadores (router-on-a-stick), priorizando uma solução simples e previsível sobre uma tecnicamente mais sofisticada, mas instável no ambiente de implementação.

---

### 1.6 Native VLAN Mismatch (ocorrência original, fase de design)

**Sintoma:** mensagens de log como:
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered...
```
identificando VLANs nativas diferentes nas duas pontas de um enlace (ex.: uma porta com nativa VLAN 1, a outra com nativa VLAN 10).

**Diagnóstico:** configuração incompatível de native VLAN entre as duas extremidades de uma porta trunk — geralmente decorrente de configuração feita durante os testes com a arquitetura L3 (seção 1.5).

**Correção:** alinhamento manual da native VLAN entre as pontas do enlace.

> **Nota:** este é um incidente diferente do mismatch de native VLAN ocorrido durante o hardening (seção 2.3) — mesma classe de erro, mas contextos e causas distintas, separados no tempo pela reformulação completa da arquitetura entre as duas fases.

---

### 1.7 STP/PVID — BPDU recebida em porta não-trunk

**Sintoma:** no switch de distribuição de Logística, mensagens:
```
%SPANTREE-2-RECV_PVID_ERR
%SPANTREE-2-BLOCK_PVID_LOCAL
```
com indicação de BPDU 802.1Q recebida em porta não configurada como trunk.

**Diagnóstico:** uma das pontas do enlace não estava configurada como trunk, mas recebia BPDUs taggeadas — inconsistência de papel entre as duas extremidades do link.

**Correção:** alinhamento do modo de porta (trunk/access) nas duas pontas do enlace, eliminando o bloqueio de proteção do STP.

---

## Fase 2 — Hardening de segurança

### 2.1 `crypto key generate rsa` rejeitado — hostname padrão

**Sintoma:**
```
% Please define a hostname other than Router.
```
ao tentar gerar par de chaves RSA para habilitar SSH, seguido de comandos subsequentes (`admin`, `1024`) sendo interpretados como comandos de configuração inválidos, pois o prompt havia retornado ao modo `(config)#` sem a chave ter sido gerada.

**Diagnóstico:** o IOS exige um hostname customizado (diferente do padrão `Router`) antes de gerar chaves RSA, já que o nome do dispositivo entra no cálculo do domínio da chave.

**Correção:**
```
hostname R-Producao
ip domain-name prodesk.local
crypto key generate rsa
```
seguido do tamanho do módulo (`1024`) quando solicitado interativamente.

---

### 2.2 `switchport port-security` rejeitado em porta dinâmica

**Sintoma:**
```
Command rejected: FastEthernet0/2 is a dynamic port.
```

**Diagnóstico:** o comando `switchport port-security` exige que a porta esteja em modo fixo (`access` ou `trunk`) — não funciona em portas no modo dinâmico padrão de fábrica (`dynamic desirable/auto`). Os comandos subsequentes (`maximum`, `violation`, `mac-address sticky`) foram aceitos sem erro aparente, mas ficaram sem efeito real, já que o comando raiz havia falhado.

**Correção:** fixar o modo da porta antes de aplicar port security:
```
switchport mode access
switchport port-security
switchport port-security maximum 1
switchport port-security violation restrict
switchport port-security mac-address sticky
```

---

### 2.3 Native VLAN Mismatch e bloqueio persistente no STP (fase de hardening)

**Sintoma:** após aplicar `switchport trunk native vlan 999` em todas as portas trunk dos switches de distribuição, `show interfaces trunk` continuava mostrando algumas portas sem a VLAN 999 na tabela de *forwarding state*, mesmo com configuração idêntica nas duas pontas. Em um caso mais severo, o STP chegou a bloquear ativamente o tráfego da VLAN 999 numa porta específica:
```
%SPANTREE-2-RECV_PVID_ERR: Received BPDU with inconsistent peer vlan id 1 on FastEthernet0/2 VLAN999.
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking FastEthernet0/2 on VLAN0999. Inconsistent local vlan.
```

**Diagnóstico:** comparando a configuração porta a porta via `show running-config`, identificou-se que apenas a primeira porta trunk de cada switch de distribuição (`Fa0/1`) tinha `switchport mode trunk` explícito, herdado da configuração original do projeto. As demais portas trunk estavam em modo **dinâmico** (negociado via DTP) — o que fazia o Spanning Tree tratar a mudança de native VLAN com mais cautela, mantendo estado de bloqueio residual mesmo após a porta remota já estar corrigida.

**Correção:**
1. Fixação do modo trunk explicitamente em todas as portas: `switchport mode trunk`.
2. Nas portas que permaneceram bloqueadas mesmo após a correção do modo, foi necessário forçar a reconvergência do STP com bounce manual da interface:
```
interface range FastEthernet0/X-Y
 shutdown
 no shutdown
```
O protocolo não recalculava automaticamente o estado a partir de um bloqueio já estabelecido — o bounce foi necessário para reiniciar o processo de convergência.

---

### 2.4 DHCP Snooping bloqueando requisições legítimas (`NONZERO_GIADDR`)

**Sintoma:** logo após habilitar `ip dhcp snooping`, hosts pararam de conseguir renovar IP (`ipconfig /renew` retornando "DHCP request failed").

**Diagnóstico:** o log do switch de distribuição revelou a causa exata:
```
%DHCP_SNOOPING-5-DHCP_SNOOPING_NONZERO_GIADDR: DHCP_SNOOPING drop message 
with non-zero giaddr or option82 value on untrusted port
```
Por padrão, o DHCP Snooping do IOS insere a opção 82 (relay agent information) nos pacotes e rejeita, em portas não plenamente confiáveis para esse fim, qualquer pacote que já chegue com `giaddr` (gateway IP address) diferente de zero — exatamente o que ocorre neste cenário, já que o **roteador** insere esse campo ao fazer relay via `ip helper-address`. O switch estava interpretando o relay legítimo do roteador como potencialmente malicioso.

**Correção:** desabilitação da inserção de opção 82 em todos os switches com snooping ativo, mantendo apenas as portas de uplink como *trusted* — suficiente para o modelo de ameaça deste cenário (relay feito por dispositivo de Camada 3 confiável):
```
no ip dhcp snooping information option
```

---

## O que esses incidentes, em conjunto, ilustram

Uma rede pode ter todos os equipamentos fisicamente conectados e ainda assim não funcionar corretamente — cada camada precisa ser validada individualmente:

```text
1. Física → 2. VLAN → 3. Trunk → 4. IP → 5. Roteamento → 6. Serviços → 7. Segurança
```

Os incidentes da Fase 1 (design) e Fase 2 (hardening) têm naturezas diferentes — a primeira lida principalmente com conectividade básica e escolhas de arquitetura; a segunda, com controles de segurança que, mal aplicados, podem quebrar serviços legítimos (como visto no caso do DHCP Snooping). Em ambas, o padrão de diagnóstico foi o mesmo: isolar a camada afetada, comparar configuração porta a porta ou dispositivo a dispositivo, e usar as mensagens de log do IOS (CDP, STP, DHCP Snooping) como fonte primária de diagnóstico em vez de tentativa e erro.

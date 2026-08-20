# Histórico de Desenvolvimento — Projeto de Rede Corporativa ProDesk

> **Nota de contexto:** este documento foi escrito durante o desenvolvimento original do projeto (unidade curricular *Ambientes Computacionais e Conectividade*, UNISUL), antes da fase de hardening de segurança documentada no [README principal](../README.md). Ele registra o raciocínio de design, as decisões de arquitetura e o processo de troubleshooting da versão acadêmica original do projeto. Alguns detalhes de implementação aqui descritos (por exemplo, tentativas com switches Layer 3 na distribuição) foram testados e descartados — a arquitetura final é a documentada no README. Mantido na íntegra por seu valor como registro do processo de engenharia.

---

## 📌 Visão geral

Este projeto consiste no planejamento e implementação de uma infraestrutura de rede para uma empresa distribuída em **dois prédios**, denominados:

* **Produção**
* **Logística**

A proposta foi desenvolver uma rede funcional, organizada e escalável, utilizando conceitos de redes corporativas como:

* Arquitetura hierárquica de rede;
* VLANs para segmentação lógica;
* Switches Layer 2;
* Roteamento entre VLANs;
* Roteamento dinâmico com OSPF;
* DHCP centralizado;
* `ip helper-address`;
* Endereçamento IPv4 com sub-redes `/26`;
* Link ponto-a-ponto `/30`;
* Interligação dos prédios por fibra óptica;
* Módulos/interfaces SFP;
* Cabeamento estruturado;
* Testes de conectividade e validação dos serviços.

O projeto foi desenvolvido e testado no **Cisco Packet Tracer**, passando por diversas etapas de implementação e troubleshooting até chegar à arquitetura funcional apresentada na versão final.

---

## 1. Objetivo do projeto

O objetivo foi construir uma infraestrutura de rede capaz de:

* conectar os diferentes setores da empresa;
* separar logicamente os departamentos;
* permitir comunicação entre setores quando necessário;
* permitir comunicação entre os dois prédios;
* fornecer endereçamento IP automaticamente;
* centralizar serviços de rede;
* utilizar roteamento dinâmico;
* organizar fisicamente os equipamentos;
* permitir expansão futura da infraestrutura;
* demonstrar, através do Packet Tracer, o funcionamento dos principais serviços.

A rede deveria atender **195 dispositivos de usuários** distribuídos entre os setores.

---

## 2. Arquitetura geral

A topologia foi estruturada seguindo uma arquitetura hierárquica de três camadas:

```text
                 CORE
          ┌───────────────┐
          │ R1 ─────── R2 │
          └───────┬───────┘
                  │
        ┌─────────┴─────────┐
        │                   │
 DISTRIBUIÇÃO          DISTRIBUIÇÃO
   Prédio 1               Prédio 2
        │                   │
   ┌────┼────┐         ┌────┼────┐
   │    │    │         │    │    │
 ACESSO ACESSO ACESSO  ACESSO ACESSO
   │    │    │         │    │    │
 HOSTS HOSTS HOSTS     HOSTS HOSTS
```

### Camada de acesso

É responsável pela conexão direta dos dispositivos finais:

* computadores;
* hosts dos setores;
* demais dispositivos de usuário.

Cada setor possui seus próprios switches de acesso.

### Camada de distribuição

É responsável pela concentração das conexões dos switches de acesso de cada prédio.

Na arquitetura final utilizada no projeto, os switches de distribuição permaneceram como **Layer 2**.

Isso permitiu manter o roteamento centralizado nos roteadores, simplificando a arquitetura e mantendo clara a divisão de responsabilidades:

```text
Switches L2 → transporte das VLANs
        ↓
Roteador → roteamento entre redes/VLANs
        ↓
OSPF → roteamento entre os prédios
```

### Camada core

Foi representada pelos roteadores:

* **R1 — Produção**
* **R2 — Logística**

Os dois roteadores são responsáveis pela interligação dos prédios e pelo roteamento entre suas respectivas redes.

---

## 3. Estrutura física

### 3.1 Prédio de Produção

O prédio de Produção possui os seguintes setores:

| Setor     | Dispositivos | VLAN |
| --------- | -----------: | ---: |
| Compras   |           40 |   10 |
| Qualidade |           40 |   20 |
| P&D       |           40 |   30 |

Total: **120 dispositivos**

A estrutura física utiliza dois switches de acesso por setor.

O switch de distribuição fica em uma sala técnica do prédio e concentra as conexões dos switches de acesso.

O roteador **R1** fica conectado ao switch de distribuição.

Também existe uma VLAN específica para serviços:

* VLAN 99 — Serviços

Nessa VLAN fica o servidor utilizado para DHCP.

---

## 4. Prédio de Logística

O prédio de Logística possui:

| Setor           | Dispositivos | VLAN |
| --------------- | -----------: | ---: |
| Administrativo  |           35 |   40 |
| Desenvolvimento |           40 |   50 |

Total: **75 dispositivos**

Somando os dois prédios:

```text
Produção:   120
Logística:   75
----------------
Total:      195 dispositivos
```

Assim, o projeto atende à quantidade de dispositivos especificada.

O prédio possui seu próprio switch de distribuição e seus switches de acesso.

O roteador **R2** realiza a interligação do prédio com o restante da rede.

---

## 5. Topologia física x topologia lógica

Durante o desenvolvimento foram utilizados os dois conceitos.

### Topologia física

Representa **onde os equipamentos estão fisicamente posicionados**.

```text
Computador
    │
Switch de acesso
    │
Switch de distribuição
    │
Roteador
    │
Fibra
    │
Roteador
```

Também representa a distribuição por andares e salas técnicas.

### Topologia lógica

Representa **como os dispositivos e redes se comunicam logicamente**, independentemente da disposição física.

No projeto, isso envolve principalmente:

* VLANs;
* sub-redes;
* gateways;
* roteamento;
* OSPF;
* DHCP;
* comunicação entre prédios.

Portanto, o termo mais adequado para o diagrama que mostra VLANs, redes e comunicação é **topologia lógica**.

---

## 6. VLANs

Cada setor recebeu uma VLAN própria.

| VLAN | Setor           | Rede            |
| ---: | --------------- | --------------- |
|   10 | Compras         | 192.168.0.0/26  |
|   20 | Qualidade       | 192.168.10.0/26 |
|   30 | P&D             | 192.168.20.0/26 |
|   40 | Administrativo  | 192.168.30.0/26 |
|   50 | Desenvolvimento | 192.168.40.0/26 |
|   99 | Serviços        | 192.168.99.0/26 |

A utilização de VLANs permite que os setores sejam separados logicamente mesmo quando compartilham a mesma infraestrutura física de switches.

Isso proporciona:

* segmentação;
* organização;
* redução do domínio de broadcast;
* controle de tráfego;
* maior facilidade de administração;
* possibilidade de aplicação de políticas de segurança posteriormente.

---

## 7. Por que uma VLAN por setor?

A separação por VLAN foi adotada para evitar que todos os dispositivos da empresa permanecessem em uma única rede lógica.

Com VLANs:

```text
Compras         → VLAN 10
Qualidade       → VLAN 20
P&D             → VLAN 30
Administrativo  → VLAN 40
Desenvolvimento → VLAN 50
Serviços        → VLAN 99
```

Isso torna a infraestrutura mais organizada e facilita o controle do tráfego.

Importante: VLAN por si só não significa isolamento absoluto de segurança. O roteamento entre VLANs ainda pode permitir comunicação quando configurado. A VLAN fornece principalmente **segmentação lógica**.

---

## 8. Endereçamento IPv4

O plano de endereçamento utilizou principalmente redes `/26`.

```text
/26 = 255.255.255.192
64 endereços totais
  1 → endereço de rede
 62 → endereços de hosts
  1 → broadcast
```

---

## 9. Por que utilizar /26?

O projeto precisava atender vários dispositivos em cada setor. Uma rede `/30` possui apenas 4 endereços totais (2 hosts válidos) — inadequada para uma rede de usuários.

A utilização foi:

```text
/26 → redes dos setores
/30 → link ponto-a-ponto entre roteadores
```

---

## 10. Faixas de endereçamento

**VLAN 10 — Compras:** rede `192.168.0.0/26`, gateway `192.168.0.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 20 — Qualidade:** rede `192.168.10.0/26`, gateway `192.168.10.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 30 — P&D:** rede `192.168.20.0/26`, gateway `192.168.20.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 40 — Administrativo:** rede `192.168.30.0/26`, gateway `192.168.30.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 50 — Desenvolvimento:** rede `192.168.40.0/26`, gateway `192.168.40.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 99 — Serviços:** rede `192.168.99.0/26`, gateway `192.168.99.1`, servidor `192.168.99.10`, broadcast `.63`

O servidor possui endereço estático (`192.168.99.10/26`) e não recebe seu próprio endereço via DHCP.

---

## 11. Link entre R1 e R2

```text
Rede:    10.0.0.0/30
Máscara: 255.255.255.252
R1:      10.0.0.1
R2:      10.0.0.2
Broadcast: 10.0.0.3
```

Essa rede possui somente dois hosts válidos, exatamente o necessário para um link ponto-a-ponto.

---

## 12. Interligação por fibra óptica

Os roteadores foram preparados com módulo/interface compatível para conexão de fibra, aparecendo como:

```text
GigabitEthernet0/3/0
```

```text
R1 GigabitEthernet0/3/0 → 10.0.0.1/30
R2 GigabitEthernet0/3/0 → 10.0.0.2/30
```

Na implementação final, essa interface foi viabilizada com um módulo `HWIC-1GE-SFP` e transceiver `GLC-SH-SMD`, coerente com a especificação do estudo de caso (duas unidades separadas por 600 metros). O `/30` foi utilizado porque o enlace necessita apenas de dois endereços IP, um para cada extremidade.

---

## 13. Cabeamento estruturado

O projeto considerou cabeamento estruturado seguindo padrões TIA/EIA.

```text
TIA/EIA → padrão/norma de infraestrutura (não é o nome de um cabo)
Cat5e   → categoria do cabo de par trançado
```

---

## 14. DHCP

O servidor DHCP foi colocado na VLAN 99:

```text
IP:       192.168.99.10
Máscara:  255.255.255.192
Gateway:  192.168.99.1
```

A VLAN 99 foi utilizada como uma rede de serviços, separando o servidor das redes de usuários.

---

## 15. Por que o servidor DHCP está em uma VLAN própria?

A VLAN 99 foi utilizada como uma rede dedicada a serviços, separando logicamente usuários e infraestrutura. Além do DHCP, uma rede de serviços pode futuramente comportar DNS, servidores de arquivos, aplicações, monitoramento e outros recursos corporativos.

---

## 16. DHCP centralizado

Em vez de criar um servidor DHCP para cada setor ou prédio, foi utilizado **um único servidor DHCP centralizado**, com escopos correspondentes às diferentes redes:

```text
                 DHCP
             192.168.99.10
                   │
        ┌──────────┼──────────┬──────────┬──────────┐
        │          │          │          │          │
     VLAN 10    VLAN 20    VLAN 30    VLAN 40    VLAN 50
        │          │          │          │          │
     Hosts       Hosts       Hosts      Hosts      Hosts
```

Isso simplifica administração, manutenção, criação de escopos, controle de endereços e expansão da infraestrutura.

---

## 17. Escopos DHCP

Foi criado um escopo para cada rede de usuários, entregando aos clientes: endereço IP, máscara, gateway padrão e DNS.

---

## 18. O problema do DHCP entre VLANs

As solicitações iniciais do DHCP utilizam broadcast. Um roteador normalmente **não encaminha broadcasts de uma rede para outra**:

```text
PC VLAN 10 → DHCP broadcast → Gateway → ✗ → Servidor em VLAN 99
```

Sem configuração adicional, a solicitação não chegaria ao servidor.

---

## 19. `ip helper-address`

Solução aplicada nas interfaces/subinterfaces responsáveis pelas redes dos clientes:

```text
ip helper-address 192.168.99.10
```

Comportamento resultante:

```text
PC → DHCP Discover → Gateway → ip helper-address → Servidor DHCP 192.168.99.10 → DHCP Offer → PC
```

Assim, o servidor centralizado consegue atender clientes de outras VLANs e outro prédio, desde que o roteamento entre as redes esteja funcionando.

---

## 20. O DHCP atende 195 dispositivos?

Sim. O `/26` não significa que o servidor só possa fornecer IP para 62 dispositivos na empresa inteira — os 62 hosts são **por sub-rede/VLAN**.

```text
5 VLANs de usuários × 62 hosts = 310 endereços disponíveis
```

Suficiente para os 195 dispositivos do projeto, com margem para crescimento.

---

## 21–24. OSPF

Protocolo de roteamento dinâmico escolhido: **OSPF**, configurado na **Área 0** (área backbone), já que a estrutura é pequena e não exige múltiplas áreas.

Motivos da escolha: funcionamento baseado em estado de enlace, cálculo de melhores caminhos, convergência rápida, suporte a múltiplas áreas, métrica de custo, melhor escalabilidade que protocolos como RIP (limitado a 15 saltos e métrica por contagem de saltos).

---

## 25. Configuração histórica do R1

```text
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.10.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.20.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.192

interface GigabitEthernet0/3/0
 ip address 10.0.0.1 255.255.255.252

router ospf 1
 log-adjacency-changes
 network 192.168.0.0 0.0.0.63 area 0
 network 192.168.10.0 0.0.0.63 area 0
 network 192.168.20.0 0.0.0.63 area 0
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.99.0 0.0.0.63 area 0
```

---

## 26. Configuração histórica do R2

```text
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 192.168.30.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.50
 encapsulation dot1Q 50
 ip address 192.168.40.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/3/0
 ip address 10.0.0.2 255.255.255.252

router ospf 1
 log-adjacency-changes
 network 192.168.30.0 0.0.0.63 area 0
 network 192.168.40.0 0.0.0.63 area 0
 network 10.0.0.0 0.0.0.3 area 0
```

---

## 27. Verificação do OSPF

Comandos principais utilizados no troubleshooting:

```bash
show ip ospf neighbor   # verifica adjacência (ex.: estado FULL/BDR)
show ip route           # verifica rotas aprendidas dinamicamente
```

---

## 28. Metodologia de troubleshooting utilizada

Uma parte significativa do desenvolvimento foi dedicada ao diagnóstico de problemas de conectividade — em VLANs, trunks, switches, roteadores, OSPF, DHCP, interfaces, endereçamento e comunicação entre prédios. Os incidentes específicos, com sintoma → diagnóstico → correção, estão documentados em **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)**.

A abordagem geral adotada, camada por camada:

```text
1. Física → 2. VLAN → 3. Trunk → 4. IP → 5. Roteamento → 6. Serviços
```

Com os comandos:

```bash
show ip interface brief
show running-config
show vlan brief
show interfaces trunk
show ip ospf neighbor
show ip route
ping <endereço>
```

---

## 29. Abordagens testadas e descartadas

### Switches Layer 3 na distribuição

Em determinado momento do desenvolvimento, foi considerada uma arquitetura utilizando switches Layer 3 na distribuição, com o roteamento das VLANs sendo feito diretamente nesses switches (via `ip routing`, SVIs, interfaces VLAN).

Essa abordagem introduziu complexidade adicional e problemas de funcionamento no cenário implementado — interfaces com `protocol down`, conflitos de endereçamento, dificuldades de comunicação com o roteador central.

**Por que foi abandonada:** a arquitetura final foi mantida com switches Layer 2 na distribuição, como decisão de projeto para manter simplicidade, previsibilidade, compatibilidade com a topologia já implementada, centralização do roteamento nos roteadores e separação clara entre switching e routing.

### Reconstrução ampla da infraestrutura

Em determinado momento foi considerada a limpeza completa de switches e roteadores para reconstrução do zero. Essa abordagem mostrou como alterações simultâneas em múltiplos pontos dificultam o diagnóstico — a estratégia posterior foi voltar ao modelo conhecido e verificar cada componente individualmente, um de cada vez.

**Princípio adotado para a versão final:**

> Uma arquitetura simples, previsível e funcional é preferível a uma arquitetura teoricamente mais sofisticada que não esteja funcionando corretamente no ambiente de implementação.

---

## 30. Decisões importantes do projeto

| Decisão | Motivo |
|---|---|
| VLAN por setor | Segmentação lógica e organização |
| `/26` nas redes dos setores | Até 62 hosts por VLAN, atendendo aos setores com margem para crescimento |
| `/30` entre os roteadores | Link ponto-a-ponto necessita apenas dois endereços válidos |
| DHCP centralizado | Reduz complexidade e facilita administração |
| VLAN 99 dedicada a serviços | Separa infraestrutura de usuários |
| `ip helper-address` | Necessário para encaminhar DHCP entre VLANs/redes diferentes |
| OSPF | Roteamento dinâmico entre os prédios, sem rotas estáticas manuais |
| Área 0 (única) | Estrutura pequena, não exige múltiplas áreas OSPF |
| Switches L2 na distribuição | Simplicidade arquitetural, roteamento centralizado nos roteadores |
| Fibra entre prédios | Representa o enlace real entre unidades separadas por 600 metros |

---

## 31. Comandos de configuração fundamentais

**Criar VLAN:**
```bash
vlan 10
 name COMPRAS
```

**Porta de acesso:**
```bash
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
```

**Trunk:**
```bash
interface GigabitEthernet0/1
 switchport mode trunk
```

**Router-on-a-stick (subinterface):**
```bash
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
```

**OSPF — wildcard mask:**
```text
/26 → 0.0.0.63
/30 → 0.0.0.3
```

---

## 32. Capacidade das redes — resumo

```text
Cada /26: 64 endereços totais, 62 hosts válidos

VLAN 10 → 62
VLAN 20 → 62
VLAN 30 → 62
VLAN 40 → 62
VLAN 50 → 62
----------------
Total   → 310 endereços disponíveis

Necessidade: 195 dispositivos
Margem: 115 endereços (crescimento futuro sem replanejar endereçamento)
```

---

## 33. Fluxo completo de um computador (novo host na rede)

```text
1. PC conecta ao switch de acesso
2. Porta pertence à VLAN do setor
3. PC envia solicitação DHCP (broadcast)
4. Gateway (subinterface do roteador) recebe a solicitação
5. ip helper-address encaminha ao servidor
6. Servidor DHCP identifica o escopo correto
7. IP/máscara/gateway/DNS são enviados
8. PC passa a utilizar a rede
```

Se o computador precisar acessar outro prédio:

```text
PC → Gateway → Roteador local → OSPF/tabela de roteamento →
Link entre R1 e R2 → Roteador remoto → VLAN remota → Destino
```

---

## 34. Tecnologias e conceitos utilizados

Cisco Packet Tracer · IPv4 · Subnetting · CIDR · VLAN · IEEE 802.1Q · Trunking · Switch Layer 2 · Router-on-a-stick · OSPF · OSPF Area 0 · DHCP · DHCP Relay · `ip helper-address` · DNS · Fibra óptica · SFP · Cabeamento estruturado · TIA/EIA · Cat5e · STP · CDP · ICMP/Ping

---

## 35. Checklist original de implementação

**Infraestrutura:** dois prédios, switches de acesso, switches de distribuição, R1 e R2, interfaces de enlace entre roteadores, conexão física entre prédios.

**VLANs:** criação das VLANs 10/20/30/40/50/99, associação de portas de acesso, configuração de enlaces trunk compatíveis entre switches.

**Endereçamento:** configuração de cada VLAN, `10.0.0.1/30` no R1, `10.0.0.2/30` no R2, `192.168.99.10/26` no servidor.

**Roteamento:** OSPF em R1 e R2, redes na área 0, verificação de vizinhança e tabela de rotas.

**DHCP:** pools de cada VLAN, gateways, máscaras, DNS, `ip helper-address`, teste nos hosts.

**Validação:** `show vlan brief`, `show interfaces trunk`, `show ip interface brief`, `show ip ospf neighbor`, `show ip route`, ping gateway, ping R1↔R2, ping entre VLANs, ping entre prédios, validação de DHCP por setor.

---

## 36. Conclusão da fase de design original

O projeto demonstrou a implementação de uma infraestrutura corporativa completa em ambiente simulado, integrando VLAN, switching, roteamento, OSPF, DHCP relay e conectividade fim-a-fim entre duas unidades físicas.

O processo de troubleshooting foi parte central do desenvolvimento, permitindo identificar e resolver problemas de camada 2, camada 3, endereçamento, STP, VLANs, OSPF e DHCP — processo documentado em detalhe em [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

A arquitetura resultante desta fase foi posteriormente reforçada com uma camada completa de hardening de segurança, documentada no [README principal](../README.md).

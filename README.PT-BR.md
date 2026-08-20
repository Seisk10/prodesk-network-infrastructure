# ProDesk — Infraestrutura de Rede Corporativa

Projeto de infraestrutura de rede para a empresa fictícia **ProDesk**, desenvolvido em Cisco Packet Tracer como parte da unidade curricular *Ambientes Computacionais e Conectividade* (UNISUL). Posteriormente revisado e reforçado com uma camada completa de hardening de segurança, com foco em mitigação de ataques comuns de rede local (Camada 2 e Camada 3).

## Sumário

- [Contexto do projeto](#contexto-do-projeto)
- [Topologia](#topologia)
- [Endereçamento IP](#endereçamento-ip)
- [Roteamento](#roteamento)
- [Serviços de rede](#serviços-de-rede)
- [Hardening de segurança](#hardening-de-segurança)
- [Troubleshooting real](#troubleshooting-real)
- [Como testar](#como-testar)
- [Limitações e decisões de design](#limitações-e-decisões-de-design)

---

## Contexto do projeto

A ProDesk recebeu aporte de investidor para estruturar sua rede de dados em duas unidades físicas distintas, separadas por 600 metros, que precisam compartilhar a mesma estrutura de comunicação interna:

- **Unidade Produção** — setores de P&D, Qualidade e Compras (120 hosts previstos)
- **Unidade Logística** — setores Administrativo e Desenvolvimento (75 hosts previstos)

O projeto original previa 195 hosts no total. Seguindo a premissa do enunciado — *"não será necessário colocar no projeto todos os hosts indicados, mas eles devem ser considerados para dimensionamento dos switches e demais equipamentos da rede"* — a topologia foi implementada com um host representativo por switch de acesso, mas com endereçamento e domínios de broadcast dimensionados para suportar a carga real de cada setor.

## Topologia

Arquitetura hierárquica de três camadas (core / distribuição / acesso), replicada em cada unidade física:

```
                    [Core - Produção] ── fibra óptica (Gi0/3/0) ── [Core - Logística]
                       Router 2911                         Router 2911
                           │                                    │
                  [Switch Distribuição]                [Switch Distribuição]
                    2960-24TT + Server                    2960-24TT
                           │                                    │
              ┌────────────┼────────────┐             ┌─────────┴─────────┐
          [Compras]    [Qualidade]    [P&D]      [Administrativo]  [Desenvolvimento]
          2 switches   2 switches  2 switches      2 switches        2 switches
          de acesso    de acesso   de acesso       de acesso         de acesso
```

**Camada Core** — dois roteadores Cisco 2911, um por unidade física, interligados por fibra óptica (módulo HWIC-1GE-SFP + transceiver GLC-SH-SMD na interface `GigabitEthernet0/3/0`) rodando OSPF.

**Camada Distribuição** — um switch 2960-24TT por unidade, concentrando o tráfego dos switches de acesso via trunk 802.1Q. O switch de Produção também hospeda o servidor de rede (DHCP/DNS) centralizado.

**Camada Acesso** — 10 switches 2960-24TT no total (2 por setor, 5 setores), cada um representando um host do setor correspondente. Porta `Fa0/1` sempre em trunk (uplink para distribuição), porta `Fa0/2` sempre em access (host).

## Endereçamento IP

Endereçamento classe C, sub-redes `/26` (255.255.255.192) por VLAN/setor, permitindo até 62 hosts utilizáveis por sub-rede — suficiente para acomodar o quantitativo real de cada setor (máximo 40 hosts previstos por área).

| VLAN | Setor | Unidade | Rede | Gateway | Faixa de hosts |
|---|---|---|---|---|---|
| 10 | Compras | Produção | 192.168.0.0/26 | 192.168.0.1 | .2 – .62 |
| 20 | Qualidade | Produção | 192.168.10.0/26 | 192.168.10.1 | .2 – .62 |
| 30 | P&D | Produção | 192.168.20.0/26 | 192.168.20.1 | .2 – .62 |
| 40 | Administrativo | Logística | 192.168.30.0/26 | 192.168.30.1 | .2 – .62 |
| 50 | Desenvolvimento | Logística | 192.168.40.0/26 | 192.168.40.1 | .2 – .62 |
| 99 | Gerência / Servidor | Produção | 192.168.99.0/26 | 192.168.99.1 | .2 – .62 |
| 999 | Native (não utilizada) | Ambas | — | — | isolamento de trunk |
| — | Link Roteador↔Roteador | Ambas | 10.0.0.0/30 | — | .1 / .2 |

> A VLAN 999 não possui hosts atribuídos — existe exclusivamente como native VLAN dos enlaces trunk, prática recomendada para mitigar ataques de VLAN hopping (double tagging).

## Roteamento

**OSPF (área 0)** roda em ambos os roteadores, anunciando todas as sub-redes locais e a rede do link interpredial entre as unidades. O roteamento inter-VLAN dentro de cada unidade é feito via **router-on-a-stick**: uma única interface física (`GigabitEthernet0/1`) subdividida em sub-interfaces com encapsulamento `dot1Q`, uma por VLAN. A interligação física entre R1 (Produção) e R2 (Logística) usa um módulo `HWIC-1GE-SFP` com transceiver `GLC-SH-SMD`, na interface `GigabitEthernet0/3/0`, simulando fibra óptica entre as duas unidades — coerente com a distância de 600 metros especificada no estudo de caso original.

Exemplo (Router Produção):
```
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.99.10
```

O `ip helper-address` em cada sub-interface redireciona broadcasts de DHCP para o servidor centralizado, permitindo que hosts de qualquer VLAN — inclusive na unidade Logística, do outro lado do link OSPF — obtenham configuração de rede automaticamente.

## Serviços de rede

Servidor único (`192.168.99.10`), fisicamente conectado à camada de distribuição de Produção, centralizando:

- **DHCP** — um pool por VLAN, com gateway e faixa de endereços dedicados, servindo as duas unidades via `ip helper-address`
- **DNS** — registro A configurado (`intranet.empresa.local` → `192.168.99.10`), suporte a resolução de nome interno

## Hardening de segurança

A topologia original (funcional, mas sem controles de segurança) foi reforçada com seis camadas de proteção, cada uma mitigando uma classe específica de ataque relevante em redes locais empresariais:

| # | Controle | Onde foi aplicado | Ataque mitigado |
|---|---|---|---|
| 1 | Senha em console, VTY e enable secret (com `service password-encryption`) | Todos os dispositivos | Acesso não autorizado à configuração via console ou rede |
| 2 | SSH (RSA 1024-bit) + usuário local, `login local` | Ambos os roteadores | Interceptação de credenciais em texto claro (Telnet) |
| 3 | Native VLAN isolada (VLAN 999) em todos os enlaces trunk | 2 switches de distribuição + 10 switches de acesso | VLAN hopping via double tagging |
| 4 | Port Security (`maximum 1`, `sticky`, `violation restrict`) | Porta de host (Fa0/2) em cada switch de acesso | MAC flooding / CAM table overflow, e conexão de dispositivo não autorizado |
| 5 | Shutdown de todas as portas não utilizadas | Todos os switches de acesso (Fa0/3–24, Gi0/1–2) | Acesso físico não autorizado por porta livre |
| 6 | DHCP Snooping (portas de uplink como *trusted*, opção 82 desabilitada) | 2 switches de distribuição + 10 switches de acesso | Rogue DHCP server / DHCP starvation |

### Detalhe técnico — DHCP Snooping neste cenário

Como o roteamento inter-VLAN e o relay de DHCP são feitos pelo **roteador** (via `ip helper-address`), e não pelo switch, a inserção padrão da **opção 82** no DHCP Snooping precisou ser desabilitada (`no ip dhcp snooping information option`). Sem esse ajuste, o switch descarta pacotes DHCP legítimos vindos do relay do roteador, por reconhecer o campo `giaddr` não-zero como não confiável em porta sem tratamento de opção 82. Esse comportamento — e o processo de diagnóstico até a causa raiz — está detalhado na seção seguinte.

## Troubleshooting real

Dois problemas reais surgiram durante a implementação do hardening, com diagnóstico documentado abaixo — não apenas a correção, mas o raciocínio até chegar nela.

### 1. Mismatch de Native VLAN preso em estado de bloqueio (Spanning Tree)

**Sintoma:** após aplicar `switchport trunk native vlan 999` em todas as portas trunk dos switches de distribuição, o comando `show interfaces trunk` continuava mostrando algumas portas sem a VLAN 999 na tabela de *forwarding state*, mesmo com a configuração idêntica nas pontas.

**Diagnóstico:** comparando a configuração porta a porta (`show running-config`), identificou-se que apenas a primeira porta trunk de cada switch (`Fa0/1`) tinha o comando `switchport mode trunk` explícito — herdado da configuração original do projeto. As demais portas estavam em modo trunk **dinâmico** (negociado via DTP), o que fazia o Spanning Tree tratar a mudança de native VLAN com mais cautela, mantendo um estado de bloqueio residual mesmo após a correção.

**Correção:**
1. Fixação do modo trunk em todas as portas: `switchport mode trunk`
2. Nas portas que permaneceram bloqueadas mesmo após a correção, foi necessário forçar a reconvergência do STP com um bounce manual da interface (`shutdown` / `no shutdown`), já que o protocolo não recalculava automaticamente o estado a partir de um bloqueio já estabelecido.

### 2. DHCP Snooping bloqueando requisições legítimas (giaddr não-zero)

**Sintoma:** logo após habilitar `ip dhcp snooping` nos switches, hosts pararam de conseguir renovar IP (`ipconfig /renew` retornando "DHCP request failed").

**Diagnóstico:** o log do switch de distribuição revelou a causa:
```
%DHCP_SNOOPING-5-DHCP_SNOOPING_NONZERO_GIADDR: DHCP_SNOOPING drop message 
with non-zero giaddr or option82 value on untrusted port
```
Por padrão, o DHCP Snooping do IOS insere a opção 82 (relay agent information) nos pacotes e rejeita, em portas não plenamente confiáveis, qualquer pacote que já chegue com um `giaddr` (gateway IP address) diferente de zero — exatamente o que acontece aqui, já que o **roteador** insere esse campo ao fazer relay via `ip helper-address`. O switch estava interpretando o relay legítimo do roteador como potencialmente malicioso.

**Correção:** desabilitação da inserção de opção 82 em todos os switches com snooping ativo — `no ip dhcp snooping information option` — mantendo as portas de uplink como *trusted*, que é o suficiente para o modelo de ameaça deste cenário (relay feito por dispositivo de Camada 3 confiável).

## Como testar

1. Abra o arquivo `.pkt` mais recente no Cisco Packet Tracer (versão 8.x recomendada)
2. Em qualquer PC, execute `ipconfig /release` seguido de `ipconfig /renew` — deve obter IP, gateway e DNS corretos da VLAN correspondente
3. Teste conectividade entre setores diferentes na mesma unidade (ex.: Compras → P&D) — valida roteamento inter-VLAN
4. Teste conectividade entre unidades (ex.: Compras, em Produção → Administrativo, em Logística) — valida OSPF e o link de fibra entre R1 e R2
5. Em um switch de distribuição, `show ip dhcp snooping binding` — deve listar os hosts com lease ativo
6. Tente conectar um SSH a qualquer roteador: `ssh -l admin <ip>` — deve autenticar com a credencial local configurada

## Limitações e decisões de design

- **Servidor único, não redundante:** por se tratar da mesma empresa em duas unidades próximas (600m), optou-se por centralizar DHCP/DNS em um único servidor em Produção, com relay via `ip helper-address` para Logística — decisão consciente de simplicidade administrativa em vez de alta disponibilidade, aceitável para o porte da rede.
- **Um host por switch de acesso:** a topologia representa a estrutura lógica completa (VLANs, sub-redes, switches dimensionados por setor), mas não popula todos os hosts previstos no enunciado (195 no total) — conforme premissa explícita do trabalho, que dispensa a presença física de todos os hosts, exigindo apenas que o dimensionamento os suporte.
- **Senhas em texto simples no `enable secret`/documentação:** as credenciais usadas (`Cisco123!`) são para fins de laboratório/demonstração e não devem ser reaproduzidas em ambiente real.
- **eJPT / estudo de pentest:** os controles aplicados (port security, DHCP snooping, native VLAN isolation) foram escolhidos deliberadamente por corresponderem a vetores de ataque de Camada 2 comumente testados em certificações ofensivas (MAC flooding, VLAN hopping, rogue DHCP), reforçando a relação direta entre este projeto e estudos em segurança ofensiva.

## Documentação estendida

Este README cobre o estado final do projeto. Para quem quiser se aprofundar no processo de design e nos incidentes técnicos ao longo do desenvolvimento (incluindo abordagens testadas e descartadas), veja:

- **[docs/HISTORICO-DESENVOLVIMENTO.md](docs/HISTORICO-DESENVOLVIMENTO.md)** — registro completo do planejamento original: por que cada decisão de arquitetura foi tomada (VLANs, `/26`, OSPF, DHCP centralizado), incluindo a tentativa de mover o roteamento para switches L3 na distribuição, testada e revertida em favor da arquitetura L2 + router-on-a-stick atual.
- **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** — todos os incidentes reais enfrentados no projeto, das duas fases de desenvolvimento (design original e hardening de segurança posterior): sintoma, diagnóstico e correção.

---

**Ferramentas:** Cisco Packet Tracer · OSPF · VLAN/802.1Q · DHCP/DNS · Port Security · DHCP Snooping

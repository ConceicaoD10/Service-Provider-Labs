# Service-Provider-Labs

# 🌐 MPLS Layer 2 & 3 VPNs: Engenharia e Arquitetura de Redes de Operadora

Bem-vindo ao repositório de documentação e evidências deste laboratório de nível *Service Provider*. Este projeto simula uma infraestrutura de operadora de elevada complexidade, integrando ecossistemas híbridos Cisco (**IOS-XR** e **IOS-XE**) sobre um core moderno.

> 📊 Nota de Navegação: Este repositório contém não apenas este artigo técnico explicativo, mas também todas as configurações finais dos equipamentos e os **prints/capturas de ecrã originais com os outputs de validação de cada tarefa executada.


## 🏗️ Arquitetura da Rede SP (Backbone)

A topologia foi meticulosamente estruturada para garantir isolamento de tráfego, resiliência e alta escalabilidade:

* **Core (P):** Roteador **XRv99** atuando como um *BGP-Free Core* focado exclusivamente no trânsito rápido de pacotes.
* **Borda (PE):** Roteadores **CSR11, CSR13, CSR14, CSR16, XRv12 e XRv15**, responsáveis pela terminação de serviços e aplicação de VRFs.

### 🔀 IGP & Transporte: IS-IS + Segment Routing (SR)

Em substituição ao tradicional protocolo LDP, utilizou-se **Segment Routing (SR)** controlado pelas extensões do **IS-IS (Level-2)**.

* **Otimização:** Ativação do `advertise passive-only` no IS-IS para garantir que apenas os prefixos `/32` das interfaces Loopback0 entrassem na tabela de labels.
* **Plano de Dados (LFIB):** Os rótulos são derivados diretamente do SRGB (*Segment Routing Global Block* - `16000-23999`). Exemplo prático: o prefixo `10.255.255.11/32` recebe deterministicamente o label `16011` (Base 16000 + Index 11).

### 📡 Plano de Controlo VPN: MP-BGP com Route-Reflectors

* **Mitigação de Full-Mesh:** Os equipamentos **CSR11** e **CSR13** operam como **Route-Reflectors (RR)**.
* **Prevenção de Loops:** Configuração estática e manual do `Cluster-ID` usando o IP de Loopback de cada RR para rastreamento via *Cluster List*.
* **Eficiência do Core:** Aplicação do comando `no bgp default ipv4-unicast` para garantir que o plano de controlo transporte estritamente a família de endereços `vpnv4 unicast`.

---

## 🛠️ Implementação de Serviços L3VPN (PE-CE)

### 1. 🏢 Cliente CustA: BGP & Controlo de Site-of-Origin (SoO)

* **Desafio:** Conectar múltiplos sites que partilham o mesmo Autonomous System (**AS 65001**).
* **Solução:** Ativação do `as-override` no PE para permitir a aceitação de rotas. Para mitigar o risco de encaminhamento recursivo instável e loops, aplicou-se o atributo **Site-of-Origin (SoO)**.
* **Validação (Label Stack):** Comandos de *traceroute* expõem o encapsulamento de duas etiquetas (*Double-Label Stack*): a etiqueta de transporte SR (ex: `16015`) e a etiqueta de serviço VPNv4 (ex: `150002`).

### 2. 🌲 Cliente CustB: OSPF & O Desafio do Domain-ID

* **Desafio:** Garantir que rotas injetadas de um site remoto OSPF cheguem ao outro site como rotas internas **Inter-Area (O IA)** e não como rotas Externas (O E2).
* **Interoperabilidade (XR vs XE):** O IOS-XE gera o *Domain-ID* automaticamente a partir do ID do processo, enquanto o IOS-XR exige definição manual. A divergência força as rotas a tornarem-se E2.
* **Solução:** Padronização manual em ambos os PEs (XRv12 e CSR16) utilizando o comando `domain-id type 0005 value 000000000002`. As comunidades estendidas do MP-BGP passam a carregar o atributo de forma homogénea.

### 3. ⚡ Cliente CustC: EIGRP & Preservação da Métrica Composta

* **Abordagem Moderna:** Configuração em **EIGRP Named Mode** devido à compatibilidade exclusiva com o IOS-XR.
* **Inteligência MP-BGP:** Na redistribuição mútua, os vetores de métrica do EIGRP (banda, atraso, fiabilidade, carga e MTU) são convertidos em Extended Communities do BGP (ex: `0x8800`). O PE remoto reconstrói a métrica original sem necessidade de configurar uma *seed metric* manual, mantendo as rotas como `Internal` (Distância Administrativa 90) no destino.

### 4. 📝 Cliente CustD: Rotas Estáticas (Simplicidade e Controlo)

* **Implementação:** Configuração hierárquica dentro de `router static vrf CustD` no IOS-XR e subsequente injeção via `redistribute static` no processo BGP VPNv4.
* **Validação CEF:** O comando `show cef vrf CustD` comprova o comportamento correto do plano de dados, aplicando a pilha correta de labels antes do envio para o core.

---

## 🔀 Route-Leaking entre VRFs (Extranet Inter-VPN)

* **Objetivo:** Permitir que o cliente CustD consiga alcançar um prefixo específico do cliente CustB (Loopback 100 - `192.168.2.254/32`), mantendo o isolamento total do resto das redes.
* **Mecanismo Híbrido:**
* No **CSR16 (IOS-XE)**, utilizou-se um `route-map` associado a um `export map` na VRF CustB para exportar seletivamente o prefixo com o Route-Target específico de inter-vpn (`10.255.255.16:24`).
* No **XRv15 (IOS-XR)**, utilizou-se uma *Route Policy* para anexar múltiplos RTs através do comando crítico `set extcommunity rt (...) additive`. Isto permite que a rota carregue etiquetas de múltiplas VPNs em simultâneo.



---

## 🔀 Implementação de Serviços L2VPN

### 🔗 1. VPWS (xConnect): Ligação Ponto-a-Ponto

* **Metodologia:** Implementação através de **Service Instances (EVC - Ethernet Virtual Circuits)** no IOS-XE, oferecendo maior flexibilidade do que o xConnect tradicional de interface.
* **Sinalização:** Estabelecimento de sessões *Targeted LDP* para a troca de Service Labels entre as pontas do circuito virtual.

### 🌐 2. VPLS Kompella: Emulação de LAN Multiponto

* **Mecanismo:** Utilização do MP-BGP para **Auto-Discovery** (família de endereços `l2vpn vpls`) e **Sinalização** de etiquetas, eliminando a dependência do LDP e escalando o core.
* **Componentes:** Criação de *Bridge-Domains* locais associados a instâncias VFI (*Virtual Forwarding Instance*) com a atribuição de *VE IDs* únicos por PE para o cálculo correto dos blocos de etiquetas.
* **Nota Técnica (CDP):** Por predefinição, o VFI bloqueia tráfego de controlo L2. Foi mandatório aplicar o comando `l2protocol forward cdp` para permitir a visibilidade topológica do cliente.

### 🚀 3. EVPN VPWS & Single-Homed: A Nova Geração L2

A transição das tecnologias legadas de Camada 2 para **EVPN (Ethernet Virtual Private Network)** introduz o MP-BGP como plano de controlo para endereços MAC/IP.

* **Benefícios:** Redução massiva de tráfego de *flooding* (BUM) e suporte nativo a topologias Redundantes Ativo-Ativo sem risco de loops de Spanning Tree.
* **EVPN VPWS:** Circuitos controlados por identificadores explícitos de *Source ID* e *Target ID*, validados pelas rotas BGP de **Tipo 1 (Auto-Discovery Route)**.
* **EVPN Single-Homed (VLAN-Based):** Associação direta de instâncias EVPN a Bridge-Domains em ambientes Single-Homed (onde o ESI é composto por zeros `00000000000000000000`).
* **Route Type 2:** Anúncio e aprendizagem de endereços MAC/IP via BGP.
* **Route Type 3 (Inclusive Multicast):** Orquestração inteligente da replicação de tráfego broadcast (essencial para o funcionamento estável do ARP e pings de broadcast do cliente).



---

## 📂 Conteúdo Técnico Disponível Neste Repositório

Para complementar a leitura deste artigo, navega pelas pastas do repositório para aceder a:

1. 📄 `/Configuracoes` - Os ficheiros de texto limpos com as configurações completas de cada roteador (P e PEs).
2. 📸 `/Prints` - Capturas de ecrã organizadas por tarefas (**Task 09**, **Task 11**, etc.), contendo os comandos de validação descritos (ex: `show bgp vpnv4 unicast`, `show l2vpn evpn mac`, traces, pings com sucesso).



*Documentação desenvolvida com rigor técnico para demonstração de competências avançadas em arquiteturas Cisco Service Provider.*

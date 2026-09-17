# 🛡️ Task: Firewall + DMZ - Segurança de Perímetro e Defense in Depth

**Autor:** André Filipe Gomes Rocha
**Instituição:** Instituto Federal de Alagoas

## PROBLEMAS PRÉVIOS ##
Ao instalar o kathara conforme previsto na Trilha 2 do Alex, fui rodá-lo normalmente, contudo, tive um problema que não consegui identificar, mas, não pareceu se referenciar a instalação e sim, ao DOCKER:
> <img width="1472" height="226" alt="Captura de tela 2026-09-16 194559" src="https://github.com/user-attachments/assets/ebef67a2-cd8d-41bb-88f9-49f9bb796218" />
E quando abri o docker estava desta forma:
> <img width="725" height="563" alt="Captura de tela 2026-09-16 195315" src="https://github.com/user-attachments/assets/cb9e32a0-b98c-4a81-b63b-0de98a1af80d" />

Com isso, tive que realizar o processo de Habilitar o ""Virtual Machine Platform" (Plataforma de Máquina Virtual)", que acredito que não fiz ao instalar o docker (faz tempo que havia instalado, e nunca mais havia utilizado).

Após isso, segui para o desenvolvimento da atividade:

## 1. Política de Segurança de Perímetro Implementada

O firewall de borda (`fw`) foi configurado adotando o princípio do **Default Deny** e o **Princípio do Menor Privilégio**, garantindo que apenas o tráfego estritamente necessário transite entre as redes.

As regras, organizadas e explicadas detalhadamente abaixo, garantem a seguinte política:

* **Default Deny:** Todo o tráfego de entrada, saída e roteamento (`FORWARD`) é bloqueado por padrão.
* **Filtragem Stateful:** Conexões estabelecidas e relacionadas são permitidas, garantindo o retorno de tráfego legítimo.
* **Acessos Permitidos (✅):** LAN para a Internet (com NAT), LAN para o Web/DNS da DMZ, e Internet exclusivamente para a porta HTTP (`80`) do Servidor Web na DMZ.
* **Acessos Bloqueados (❌):** Internet para a LAN e DMZ para a LAN.

### Limpeza do firewall e aplicação do bloqueio padrão

```bash
iptables -F
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

### Permissão de retorno para conexões já estabelecidas

```bash
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

### Permissão de saída da LAN para a Internet

```bash
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.10.0/24 -j ACCEPT
```

### Permissão de acesso da LAN aos servidores Web e DNS na DMZ

```bash
iptables -A FORWARD -i eth1 -o eth2 -s 192.168.10.0/24 -d 172.16.10.80 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -s 192.168.10.0/24 -d 172.16.10.53 -j ACCEPT
```

### Permissão de acesso da Internet ao Servidor Web da DMZ

O acesso é restrito à porta HTTP (`80`):

```bash
iptables -A FORWARD -i eth0 -o eth2 -d 172.16.10.80 -p tcp --dport 80 -j ACCEPT
```

### Configuração do NAT para a rede interna

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

# 2. Experimentos e Evidências dos Controles (L2, L3 e L4)

## 🔗 L2 — Enlace: Bloqueio de Dispositivo por MAC

### Objetivo

Impedir o tráfego do dispositivo `pc2` utilizando seu endereço MAC original.

### Antes da Regra

O `pc2` possuía acesso normal à Internet.

### Regra Implementada

```bash
iptables -I FORWARD 1 -m mac --mac-source fe:d0:b1:99:7f:ea -j DROP
```

> ⚠️ **Observação:** Durante a aplicação, o módulo `mac` retornou `not supported` devido a limitações do kernel no ambiente simulado pelo Docker/WSL.

**Evidência:**

> <img width="879" height="92" alt="Captura de tela 2026-09-16 202851" src="https://github.com/user-attachments/assets/a71d7d89-d5e1-4ada-b7bf-952822f1b23b" />


### Depois da Regra

Devido à limitação técnica da simulação, a regra não instanciou o bloqueio e o dispositivo manteve a conectividade.

**Evidência:**

> 📷 <img width="899" height="293" alt="Captura de tela 2026-09-16 202910" src="https://github.com/user-attachments/assets/39c77d76-c506-42fd-99f4-f1b38384a1e7" />


### Respostas da Investigação L2

**O endereço MAC acompanha o pacote durante todo o seu percurso pela Internet? Em quais condições o firewall consegue enxergar o MAC original do `pc2`?**

> Não. O MAC atua exclusivamente na Camada de Enlace e só é válido dentro de um mesmo domínio de broadcast (rede local). A cada salto em um roteador, os endereços MAC de origem e destino são substituídos. O firewall só enxerga o MAC original do `pc2` pois ambos compartilham fisicamente/logicamente a mesma sub-rede (LAN).

---

## 🌐 L3 — Rede: Bloqueio de ICMP e de Endereços IP

### Objetivo

Bloquear tráfego do protocolo ICMP originado na LAN e destinado ao Servidor DNS, e impedir o acesso da LAN ao IP externo `8.8.8.8`.

### Antes da Regra

A comunicação ICMP do `pc1` para a DMZ ocorria com **0% de perda**.

**Evidência:**

> <img width="776" height="239" alt="Captura de tela 2026-09-16 202949" src="https://github.com/user-attachments/assets/462d415b-5f7f-47a1-831f-0706d140413e" />


### Regra Implementada — Bloqueio ICMP

```bash
iptables -I FORWARD 1 -p icmp -s 192.168.10.0/24 -d 172.16.10.53 -j DROP
```

### Depois da Regra

O tráfego foi interrompido, resultando em **100% de perda de pacotes (Timeout)**.

**Evidência:**

> <img width="838" height="146" alt="Captura de tela 2026-09-16 203805" src="https://github.com/user-attachments/assets/ffa008e5-bb8a-4ae4-9e5b-4cccbdbfbff6" />


### Respostas da Investigação L3

**Bloquear o endereço IP é uma boa solução para impedir o acesso a determinado site?**

> Não é a solução ideal. Na Internet atual, um domínio geralmente é suportado por CDNs (*Content Delivery Networks*) e resolve para múltiplos IPs dinâmicos.
> Além disso, infraestruturas de nuvem frequentemente hospedam centenas de serviços em um único endereço IP.
>
> Bloquear um IP pode falhar em restringir o alvo e, colateralmente, derrubar serviços legítimos que compartilham o mesmo endereço.

---

## 🚪 L4 — Transporte: Bloqueio de Serviços P2P

### Objetivo

Simular o bloqueio de uma aplicação do tipo BitTorrent, restringindo o acesso nas portas TCP `6881` a `6889`.

### Antes da Regra

A conexão TCP via Netcat na porta `6881` do servidor Web foi estabelecida com sucesso.

**Evidência:**

> <img width="728" height="70" alt="Captura de tela 2026-09-16 205259" src="https://github.com/user-attachments/assets/673e3907-1b0d-4f78-856c-3f30ef67f265" />


### Regra Implementada

```bash
iptables -I FORWARD 1 -p tcp --dport 6881:6889 -s 192.168.10.0/24 -j DROP
```

### Depois da Regra

A mesma requisição via Netcat foi filtrada pelo firewall, retornando erro de **Timeout**.

**Evidência:**

> 📷 <img width="971" height="70" alt="Captura de tela 2026-09-16 205425" src="https://github.com/user-attachments/assets/c43776f0-354d-4493-92da-fecd770b28bc" />


### Respostas da Investigação L4

**Bloquear portas é suficiente para garantir que uma aplicação como BitTorrent não funcione?**

> Não. Softwares P2P contemporâneos operam com portas dinâmicas (*port hopping*) e são capazes de mascarar seu tráfego encapsulando-o em portas tradicionais de web, como `80` e `443`.
>
> Filtros passivos baseados em L4 não conseguem barrar esse comportamento evasivo.

---

# 3. Proposta para Controle em L7 (Aplicação)

Para superar as limitações dos bloqueios L3 e L4, a estratégia recomendada na Camada 7 é a implantação de um **NGFW (Next-Generation Firewall)** integrado com **Application Control**, utilizando **DPI (Deep Packet Inspection)**.

Em vez de tomar decisões baseadas em portas fixas, o DPI realiza uma inspeção profunda do *payload* (conteúdo interno) do tráfego.

Ele reconhece a "assinatura" característica do protocolo, como um handshake específico do BitTorrent, permitindo ao firewall derrubar a conexão independentemente da porta aleatória ou ofuscada que o aplicativo esteja tentando utilizar.

---

# 4. Defense in Depth (Defesa em Profundidade) na Topologia

A configuração atual utiliza múltiplas barreiras independentes para proteger a infraestrutura.

Caso o Servidor Web seja comprometido por um atacante, ele não conseguirá acessar `pc1` ou `pc2` diretamente.

A rede implementa **Defense in Depth** ao combinar:

* Segmentação física/lógica;
* DMZ separada da LAN;
* Política de **Default Deny** no firewall;
* Controle de tráfego entre os diferentes segmentos da rede.

Essa arquitetura impõe um limite estrito na movimentação lateral.

Como não existe regra que permita a inicialização de conexões de dentro da DMZ com destino à LAN, um atacante que eventualmente comprometa o Servidor Web fica isolado no segmento exposto.

---

# 5. Respostas Finais

## A) Quem pode se comunicar com quem?

| Origem   | Destino            | Comunicação |
| -------- | ------------------ | ----------- |
| LAN      | Internet           | ✅ Permitida |
| LAN      | DMZ                | ✅ Permitida |
| Internet | Servidor Web — DMZ | ✅ Permitida |
| Internet | LAN                | ❌ Bloqueada |
| DMZ      | LAN                | ❌ Bloqueada |

### Resumo

* A **LAN** consegue se comunicar ativamente com a Internet e com a DMZ.
* A **Internet** só consegue iniciar comunicação com o Servidor Web localizado na DMZ.
* A **DMZ** está restrita a responder requisições e não pode iniciar comunicação para a LAN.

---

## B) Que tipos de comunicação são permitidos ou bloqueados?

### ✅ Permitidos

* Tráfego HTTP (`TCP 80`) originário da Internet para o Web Server;
* Conexões variadas de saída da LAN para a Internet;
* Conexões da LAN para os servidores da DMZ;
* Fluxos de retorno das conexões previamente estabelecidas.

### ❌ Bloqueados

* Qualquer conexão não solicitada originada fora da organização direcionada à LAN;
* Ping (`ICMP`) interno para o DNS da DMZ após a aplicação do teste L3;
* Tráfego na faixa de portas P2P (`TCP 6881–6889`) após a aplicação do teste L4;
* Comunicação iniciada pela DMZ em direção à LAN.

---

## C) Se uma camada de segurança falhar, quais outras ainda protegem a infraestrutura?

Se o perímetro falhar, a segurança da rede interna ainda dependerá de múltiplas camadas complementares, incluindo:

* **Firewalls baseados em Host:** regras de segurança implementadas diretamente nos sistemas operacionais dos endpoints e servidores;
* **Hardening:** manutenção de serviços desnecessários desativados e redução da superfície de ataque;
* **Autenticação e Gestão de Identidade:** aplicação do princípio do menor privilégio e utilização de MFA;
* **Criptografia ponta a ponta:** proteção dos dados críticos durante sua transmissão;
* **Segmentação de rede:** isolamento entre LAN, DMZ e Internet para dificultar a movimentação lateral.

Essa abordagem caracteriza o princípio de **Defense in Depth**, no qual a segurança não depende de uma única barreira, mas de várias camadas independentes de proteção.

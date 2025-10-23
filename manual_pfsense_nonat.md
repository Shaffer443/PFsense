# Manual de Implementação: pfSense NO NAT para Autenticação Fortinet Individual

## 📋 Sumário

1. [Contexto e Objetivo](#contexto)
2. [Arquitetura da Solução](#arquitetura)
3. [Pré-requisitos](#prerequisitos)
4. [Implementação no pfSense](#implementacao-pfsense)
5. [Configuração no Fortinet](#configuracao-fortinet)
6. [Validação e Testes](#validacao)
7. [Solução de Problemas](#problemas)
8. [Checklist de Implementação](#checklist)

---

## 🎯 1. Contexto e Objetivo {#contexto}

### Problema Identificado

Por padrão, o pfSense realiza **NAT de Saída** (masquerade/SNAT), traduzindo todos os IPs internos (10.x.x.x) para o IP da interface WAN (ex: 200.196.184.130). Como resultado:

- O Fortinet recebe todas as requisições com um único IP de origem
- Após o primeiro login no captive portal, toda a rede (600-800 dispositivos) é liberada
- Não há autenticação individual por usuário/dispositivo

### Objetivo da Solução

Configurar o pfSense em **modo de roteamento puro (NO NAT)** para que:

- Cada dispositivo seja identificado pelo seu IP real (ex: 10.4.0.33)
- O Fortinet exija autenticação individual por dispositivo
- Os serviços internos (AD, DHCP, DNS) continuem funcionando normalmente

---

## 🏗️ 2. Arquitetura da Solução {#arquitetura}

### Fluxo de Tráfego Antes (COM NAT - Problema)

```
Desktop (10.4.0.33) → pfSense LAN (10.0.0.1) 
→ pfSense WAN (200.196.184.130) [NAT aplicado] 
→ Fortinet vê apenas 200.196.184.130 
→ Login único libera TODOS os dispositivos
```

### Fluxo de Tráfego Depois (SEM NAT - Solução)

```
Desktop (10.4.0.33) → pfSense LAN (10.0.0.1) 
→ pfSense WAN [SEM NAT - roteamento puro] 
→ Fortinet vê 10.4.0.33 (IP real)
→ Captive Portal exige login individual
→ Fortinet faz NAT final para Internet
```

### Componentes Afetados

- **pfSense**: Configuração de NAT de Saída (Outbound NAT)
- **Fortinet**: Rotas estáticas para redes internas
- **Clientes**: Nenhuma alteração necessária

---

## ✅ 3. Pré-requisitos {#prerequisitos}

### Informações Necessárias

- **Redes internas**: 10.0.0.0/24, 10.3.0.0/24, 10.4.0.0/23, 10.5.0.0/24
- **IP WAN do pfSense**: 200.196.184.130
- **Gateway do Fortinet**: 200.196.184.129 (provável)
- **Servidor AD/DNS**: 10.4.0.9

### Acessos Requeridos

- Acesso administrativo ao pfSense (WebGUI)
- Coordenação com responsável pelo Fortinet (Consórcio PE Conectado)

### Janela de Manutenção

- **Tempo estimado**: 30-60 minutos
- **Impacto**: Interrupção temporária do acesso à Internet durante a configuração

---

## 🔧 4. Implementação no pfSense {#implementacao-pfsense}

### Passo 1: Alterar o Modo de NAT de Saída

1. Acesse o pfSense via navegador (WebGUI)
2. Navegue até **Firewall > NAT**
3. Clique na aba **Saída (Outbound)**
4. Selecione **Geração de regra NAT de Saída Manual (AON - Advanced Outbound NAT)**
5. Clique em **Salvar** e depois em **Aplicar Mudanças**

> **💡 Nota**: O modo **Hybrid Outbound NAT** pode ser usado se houver necessidade de manter algumas regras automáticas (ex: VPN IPsec/ISAKMP porta 500).

---

### Passo 2: Criar Alias para Redes Internas (Recomendado)

Este passo simplifica o gerenciamento quando há múltiplas sub-redes.

1. Navegue até **Firewall > Aliases > IP**
2. Clique em **Add**
3. Configure:
   - **Name**: `REDE_LOCAL`
   - **Type**: `Network(s)`
   - **Networks**: Adicione todas as redes internas:
     - `10.0.0.0/24`
     - `10.3.0.0/24`
     - `10.4.0.0/23`
     - `10.5.0.0/24`
4. Clique em **Salvar** e **Aplicar**

---

### Passo 3: Criar a Regra NO NAT (Sem Tradução)

Esta é a regra **crítica** que impede o mascaramento de IP.

1. Retorne para **Firewall > NAT > Outbound**
2. Clique em **Adicionar** (botão com seta para cima, no topo da lista)
3. Configure os campos:

| Campo                            | Valor              | Observação                               |
| -------------------------------- | ------------------ | ---------------------------------------- |
| **Não faça NAT**                 | ✅ **MARCADO**      | **CRUCIAL** - Mantém o IP de origem real |
| **Desabilitado**                 | ❌ Desmarcado       | A regra deve estar ativa                 |
| **Dispositivo (Interface)**      | `WAN`              | Interface de saída para o Fortinet       |
| **Família de endereço**          | `IPv4`             | -                                        |
| **Protocolo**                    | `Any`              | Qualquer protocolo                       |
| **Fonte (Source) / Tipo**        | `Network or Alias` | -                                        |
| **Fonte / Rede**                 | `REDE_LOCAL`       | O alias criado anteriormente             |
| **Destino (Destination) / Tipo** | `Network or Alias` | Necessário para evitar erros             |
| **Destino / Rede**               | `0.0.0.0`          | Representa "Qualquer Destino"            |
| **Destino / Máscara**            | `0`                | Equivale a 0.0.0.0/0                     |
| **Tradução / Endereço**          | `None`             | Confirma o "No NAT"                      |

4. Clique em **Salvar** e **Aplicar Mudanças**

> ⚠️ **IMPORTANTE**: Se o campo "Destino" não aceitar valores vazios, use exatamente **0.0.0.0** com máscara **0**. Não use "any" literal.

---

### Passo 4: Ajustar a Ordem das Regras (CRÍTICO)

A regra NO NAT **DEVE** ser processada PRIMEIRO.

1. Na aba **Outbound NAT**, localize a regra NO NAT recém-criada
2. **Arraste a regra para o TOPO da lista** (primeira posição)
3. **Desative ou delete** quaisquer regras automáticas antigas que fazem NAT para "WAN Address"
4. Clique em **Salvar** e **Aplicar Mudanças**

> 💡 **Dica**: Regras são processadas de cima para baixo. A primeira regra que corresponder ao tráfego será aplicada.

---

### Passo 5: Limpar Estados de Conexão (Recomendado)

Para garantir que conexões antigas não continuem usando o NAT antigo:

1. Navegue até **Diagnostics > States > Reset States**
2. Confirme a operação

> ⚠️ **Atenção**: Isso desconectará temporariamente todas as sessões ativas.

---

### Passo 6: Verificar Configurações de Roteamento

1. Vá em **System > Routing > Gateways**
2. Confirme que o gateway padrão da WAN aponta para o IP do Fortinet (ex: 200.196.184.129)
3. Verifique em **Firewall > Rules > LAN** se existe uma regra **Pass** permitindo tráfego de `LAN net` para `any`

---

## 🌐 5. Configuração no Fortinet {#configuracao-fortinet}

### Rotas Estáticas de Retorno (ESSENCIAL)

Ao desabilitar o NAT no pfSense, os pacotes chegam ao Fortinet com IPs privados (10.x.x.x). O Fortinet precisa saber como devolver as respostas.

### Ação Necessária

Solicitar ao responsável pelo Fortinet (Consórcio PE Conectado) a criação das seguintes **Rotas Estáticas**:

| Rede de Destino | Máscara               | Gateway (Next Hop) |
| --------------- | --------------------- | ------------------ |
| `10.4.0.0`      | `255.255.254.0` (/23) | `200.196.184.130`  |
| `10.5.0.0`      | `255.255.255.0` (/24) | `200.196.184.130`  |
| `10.0.0.0`      | `255.255.255.0` (/24) | `200.196.184.130`  |
| `10.3.0.0`      | `255.255.255.0` (/24) | `200.196.184.130`  |

**Gateway**: Sempre o IP WAN do pfSense (200.196.184.130)

> ⚠️ **CRÍTICO**: Sem estas rotas, o tráfego de saída funcionará, mas o Fortinet não conseguirá retornar as respostas, resultando em falha total de navegação.

---

## ✔️ 6. Validação e Testes {#validacao}

### Teste 1: Verificar Roteamento (Traceroute)

Execute de um desktop cliente (10.x.x.x):

```
tracert 8.8.8.8
```

**Resultado esperado**:

- **1º salto**: Gateway LAN do pfSense (ex: 10.0.0.1)
- **2º salto**: IP do Fortinet/Gateway WAN (ex: 200.196.184.129)
- **3º salto**: IP do provedor de Internet

---

### Teste 2: Autenticação Individual

1. Abra um navegador em um dispositivo não autenticado
2. Tente acessar qualquer site (ex: google.com)
3. **Resultado esperado**: Página de login (Captive Portal) do Fortinet deve aparecer
4. Faça login com credenciais válidas
5. Verifique se a navegação funciona após autenticação
6. **Teste crítico**: Em outro dispositivo (IP diferente), repita o processo
   - O segundo dispositivo **DEVE** pedir login novamente
   - Se o segundo dispositivo já tiver acesso sem login, o NO NAT não está funcionando

---

### Teste 3: Conectividade com Servidores Internos

1. Verifique se o logon de rede no domínio (AD) funciona normalmente
2. Teste acesso a recursos de rede (ex: servidor de arquivos 10.4.0.9)
3. Confirme que o DNS interno está resolvendo nomes corretamente

**Resultado esperado**: Nenhum impacto nos serviços internos

---

### Teste 4: Verificar Logs

**No pfSense**:

- Acesse **Status > System Logs > Firewall**
- Verifique se não há bloqueios inesperados de tráfego da LAN para WAN

**No Fortinet** (solicitar ao responsável):

- Verificar logs de autenticação
- Confirmar que os logins estão sendo registrados contra IPs reais (10.x.x.x) e não contra 200.196.184.130

---

## 🔧 7. Solução de Problemas {#problemas}

### Problema 1: Login Único Ainda Libera Toda a Rede

**Causa**: A regra NO NAT não está sendo aplicada corretamente.

**Soluções**:

1. Verifique se o modo está em **Manual Outbound NAT** (não Automático)
2. Confirme que a regra NO NAT está **marcada** com checkbox "Não faça NAT"
3. Garanta que a regra NO NAT está no **TOPO da lista** (primeira posição)
4. Delete ou desative regras antigas que fazem masquerade para "WAN Address"
5. Clique em **Aplicar Mudanças** e execute **Reset States**

---

### Problema 2: Navegação Falha Após Autenticação no Fortinet

**Causa**: Fortinet não possui rotas de retorno para as redes 10.x.x.x.

**Soluções**:

1. Solicitar urgentemente a criação de rotas estáticas no Fortinet (ver seção 5)
2. Executar `tracert 8.8.8.8` do cliente - o 2º salto deve ser o Fortinet
3. Verificar no Fortinet se há erros de roteamento nos logs

---

### Problema 3: pfSense Não Salva a Regra NO NAT

**Causa**: Erro de sintaxe no campo "Destino/Rede" - o pfSense não aceita "any" literal.

**Solução**:

- Use exatamente **0.0.0.0** no campo "Rede"
- Use **0** no campo "Máscara de sub-rede"
- Isso representa 0.0.0.0/0 (qualquer destino)

---

### Problema 4: pfSense Não Alcança o Fortinet/Internet

**Causa**: Gateway da WAN do pfSense está incorreto.

**Solução**:

1. Acesse **System > Routing > Gateways**
2. Edite o gateway da WAN
3. Confirme que o IP está correto (ex: 200.196.184.129)
4. Salve e aplique

---

### Problema 5: Servidores Internos (AD) Param de Funcionar

**Causa**: Configuração incorreta de DNS ou gateway nos clientes.

**Solução**:

1. Confirme que o DNS primário dos clientes aponta para o servidor AD (10.4.0.9)
2. Verifique que o gateway dos clientes é o pfSense LAN (10.0.0.1)
3. A configuração NO NAT não deve afetar tráfego interno (10.x.x.x para 10.x.x.x)

---

## 📋 8. Checklist de Implementação {#checklist}

### Antes da Implementação

- [ ] Documentar configuração atual do pfSense (backup)
- [ ] Confirmar IPs: WAN pfSense, Gateway Fortinet, redes internas
- [ ] Agendar janela de manutenção
- [ ] Notificar usuários sobre interrupção temporária
- [ ] Coordenar com responsável pelo Fortinet para criação de rotas

### Durante a Implementação (pfSense)

- [ ] Mudar modo para "Manual Outbound NAT"
- [ ] Criar alias REDE_LOCAL com todas as sub-redes
- [ ] Criar regra NO NAT com checkbox marcado
- [ ] Configurar destino como 0.0.0.0/0
- [ ] Mover regra NO NAT para o TOPO da lista
- [ ] Desativar/deletar regras antigas de masquerade
- [ ] Salvar e Aplicar Mudanças
- [ ] Executar Reset States

### Durante a Implementação (Fortinet)

- [ ] Solicitar criação de rotas estáticas para 10.x.x.x via 200.196.184.130
- [ ] Confirmar que as rotas foram aplicadas

### Validação

- [ ] Executar `tracert 8.8.8.8` de cliente - confirmar 2º salto
- [ ] Testar captive portal em primeiro dispositivo
- [ ] Testar captive portal em segundo dispositivo (DEVE pedir login)
- [ ] Verificar acesso ao servidor AD (10.4.0.9)
- [ ] Testar DHCP e resolução DNS
- [ ] Revisar logs do pfSense para bloqueios inesperados
- [ ] Solicitar verificação de logs do Fortinet (IPs reais registrados)

### Pós-Implementação

- [ ] Documentar configuração final
- [ ] Notificar usuários sobre retorno do serviço
- [ ] Monitorar por 24-48h para identificar problemas
- [ ] Atualizar documentação de rede

---

## 📞 Contatos e Responsabilidades

| Responsabilidade      | Responsável                     | Observação             |
| --------------------- | ------------------------------- | ---------------------- |
| Configuração pfSense  | Administrador Local             | Passos 1-6             |
| Configuração Fortinet | Consórcio PE Conectado          | Rotas estáticas        |
| Suporte a usuários    | TI Local                        | Orientação sobre login |
| Validação final       | Administrador Local + Consórcio | Testes conjuntos       |

---

## 📚 Glossário

- **NAT (Network Address Translation)**: Tradução de endereços de rede
- **NO NAT**: Configuração que desabilita a tradução, mantendo IPs originais
- **Captive Portal**: Página de login forçada para acesso à Internet
- **Outbound NAT**: NAT de saída (tráfego da LAN para WAN)
- **Masquerade/SNAT**: Tradução do IP de origem para o IP da interface de saída
- **Next Hop**: Próximo roteador no caminho até o destino
- **Alias**: Grupo nomeado de IPs/redes no pfSense para simplificar regras

---

**Versão do Manual**: 1.0  
**Data**: Outubro 2025  
**Autor**: Rafael Gouveia

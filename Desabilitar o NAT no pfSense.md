📌 Situação atual

* Cada usuário já tem login de rede (AD/Windows Server 2012).
  
* O **Fortinet** solicita login individual para liberar internet (captive portal ou proxy autenticado).
  
* O **pfSense** está “na frente” e muda o IP WAN para um IP LAN (NAT) → isso pode “mascarar” quem é quem, então o Fortinet só vê o IP do pfSense, não o usuário real.
  
* Resultado: quando um usuário loga no Windows, o tráfego sai, e o Fortinet pode estar liberando para todos sem pedir login novamente.
  

## Soluções Viáveis:

**1.Integração AD ↔ Fortinet (melhor opção)**

* O Fortinet (dependendo do modelo) suporta integração com **Active Directory**.
  
* Assim, quando o usuário loga no Windows (domínio AD), o Fortinet já reconhece quem é e aplica a regra de internet sem pedir login de novo.
  
* É o mais limpo, porque a autenticação é transparente.
  
* Precisa configurar **FSSO (Fortinet Single Sign-On)**.
  

2.**PfSense como autenticador (RADIUS/Proxy)**

* O pfSense pode ser configurado para exigir login RADIUS antes de liberar tráfego para a WAN.
  
* Esse RADIUS pode estar ligado ao AD (Windows Server).
  
* Dessa forma, ao logar no Windows, o usuário **também é validado para internet**.
  
* Mas você teria duas camadas: pfSense + Fortinet → pode complicar.
  

3.**Remover o pfSense do caminho da autenticação**

* Se o Fortinet já é o firewall principal, talvez seja melhor que ele faça direto a autenticação dos usuários, sem depender do pfSense no meio.
  
* Isso evita o problema de NAT mascarar tudo em um IP só.
  

* * *

# Boa, agora entendi exatamente, Rafael 👌.

Você **não quer** que o login no Windows libere automaticamente a internet (como seria com FSSO).👉 Você **quer manter o portal de login do Fortinet (captive portal)**, mas garantir que **cada usuário/dispositivo** seja obrigado a logar individualmente.O problema atual é que, quando um usuário digita usuário/senha no portal do Fortinet, **toda a rede fica liberada**, porque o Fortinet só está vendo **um único IP** (o do pfSense NAT).

* * *

🔍 Onde está o problema

O **pfSense está fazendo NAT (masquerade)**.Isso significa que:

* Todos os PCs da sua LAN → saem para a WAN com o **mesmo IP** (10.0.0.1, por exemplo).
  
* O Fortinet enxerga apenas esse IP e entende que o login vale para todos.
  

Por isso, depois que **um usuário autentica**, o Fortinet associa o IP (do pfSense) a esse login, e libera a navegação de todos que passam por ele.

* * *

✅ Soluções possíveis

### 1. **Desabilitar o NAT no pfSense (bridge/roteamento puro)**

* Configurar o pfSense para não traduzir todos os IPs em um só.
  
* Assim, cada dispositivo mantém seu IP real (ex: 10.0.0.101, 10.0.0.102, etc).
  
* O Fortinet vai enxergar cada IP individualmente, e o captive portal pedirá login **para cada PC**.
  
* Essa é a solução **mais correta**.
  

No pfSense:

* Vá em **Firewall > NAT > Outbound**.
  
* Troque para **Manual Outbound NAT**.
  
* Remova as regras de NAT que mascaram toda a LAN.
  
* Com isso, o pfSense age como roteador/bridge e o Fortinet passa a ver os IPs originais.
  

* * *

### 2. **Captive Portal baseado em MAC (no Fortinet)**

* Alguns FortiGate permitem configurar a autenticação por **endereço MAC** em vez de só IP.
  
* Assim, mesmo com NAT, o Fortinet pode diferenciar dispositivos.
  
* Não é tão confiável (MAC pode ser falsificado), mas pode resolver em redes menores.
  

* * *

### 3. **PfSense apenas como roteador interno**

* Outra opção é deixar o Fortinet ser o **firewall/gateway real de saída**, e o pfSense apenas para roteamento ou serviços internos.
  
* Dessa forma, a autenticação sempre cai direto no Fortinet, sem NAT atrapalhando.
  

* * *

🔑 Resumindo

O **problema não está no Fortinet**, e sim no **pfSense fazendo NAT**.Se você quer que **cada usuário veja a tela de login do Fortinet individualmente**, precisa que o Fortinet enxergue os IPs de cada máquina.👉 Isso só acontece se **o pfSense não mascarar (NAT)** todos os clientes em um único IP.

* * *

Boa, Rafael 👌, a tela que você mostrou é do **Encaminhamento de Portas (Port Forward)** → não é ali que a gente mexe para resolver seu problema de autenticação no Fortinet.

O ponto que precisamos ver é o **Outbound NAT**, porque é nele que o pfSense está transformando todos os IPs internos em **um único IP de saída** (o que engana o Fortinet).

* * *

🔧 Onde mexer no pfSense

1. Vá no menu: Firewall > NAT > Saída (Outbound)
  

2. Você verá três opções de modo NAT:
  
  * **Automatic Outbound NAT** → padrão, mascara tudo em um IP só.
    
  * **Hybrid Outbound NAT** → permite misturar regras automáticas e manuais.
    
  * **Manual Outbound NAT** → você controla totalmente as regras.
    

👉 Para o seu caso, escolha **Manual Outbound NAT**.

* * *

🔧 O que mudar

* Hoje o pfSense está pegando tudo da sua rede interna (ex: `10.4.0.0/24`, `10.0.0.0/24` etc.) e trocando para o IP WAN dele.
  
* O que você precisa: **desativar esse NAT**, para que o tráfego siga com o IP real de cada máquina até o Fortinet.
  

No modo **Manual**:

1. Vai aparecer uma lista de regras automáticas.
  
2. Você pode **remover ou desativar** as regras que fazem “masquerade” (source = sua rede LAN → translation = interface address).
  
3. Depois aplique e salve.
  

* * *

📌 E sobre seus 600 IPs fixos

* Não precisa mexer em cada um manualmente.
  
* Se eles já estão atribuídos por IP fixo/MAC no pfSense, **eles continuarão sendo usados normalmente**.
  
* A diferença é que, em vez do pfSense mascarar todos como **um IP só**, ele vai deixar **cada IP sair como ele é**.
  
* O Fortinet, então, vai enxergar **600 conexões diferentes**, e para cada uma vai pedir login individual no portal (do jeito que você precisa).
  

* * *

⚠️ Atenção: ao desativar o NAT, o pfSense passa a funcionar mais como **roteador** do que como **firewall mascarador**.Isso significa:

* O Fortinet precisa ter rotas para essas redes internas (ex: saber chegar em `10.4.0.0/16`).
  
* Normalmente, basta colocar o pfSense como **gateway** no Fortinet para essas redes.
  

* * *

Show, Rafael 🙌 agora ficou bem claro com essa tela.

Hoje o seu **pfSense está em modo "Automatic Outbound NAT"**, e por isso **todos os IPs internos (10.4.x.x, 10.3.x.x, 10.5.x.x, etc.) saem NATeados como o endereço da WAN** (200.196.184.130).

👉 Isso explica por que o Fortinet só enxerga **um IP (o do pfSense)** e, quando um usuário loga no portal, todos passam a ter internet.

* * *

✅ O que você precisa

Que o Fortinet enxergue os **IPs reais** (10.4.0.33, 10.5.0.10, etc.), para poder exigir autenticação individual.

* * *

🔧 Ajustando o pfSense

1. Vá em **Firewall > NAT > Saída (Outbound)**.
  
2. Troque de **Automática** para **Manual Outbound NAT**.
  
  * Vai aparecer a lista de regras atuais (as duas que você vê aí: “Regras criadas automaticamente…”).
3. **Apague** (ou desative) as regras que fazem NAT para “WAN Address”.
  
  * Essas regras são justamente as que transformam tudo em 200.196.184.130.
4. Salve e aplique.
  

* * *

🔄 Depois disso

* O pfSense não vai mais mascarar todos os IPs → ele passa a **rotear**.
  
* Ou seja, o desktop `10.4.0.33` vai sair para a internet como **10.4.0.33** mesmo.
  
* O Fortinet vai receber essa conexão e pedir login **só para esse IP/dispositivo**.
  
* Outro desktop (`10.5.0.20`, por exemplo) terá que fazer login separado, porque o Fortinet vai enxergar como origem diferente.
  

* * *

⚠️ Pontos importantes

1. O Fortinet precisa **saber que redes internas existem** (10.0.0.0/24, 10.4.0.0/23, 10.3.0.0/24, etc.).
  
  * Ou seja, no Fortinet você deve criar **rotas estáticas** apontando para o pfSense como gateway para cada rede interna.
    
  * Exemplo: rota para `10.4.0.0/23` via `10.0.0.1` (IP do pfSense).
    
2. Se o Fortinet não tiver essas rotas, ele não vai conseguir responder para os IPs internos → e aí a internet não funciona.
  
3. No seu caso, com ~600 IPs fixos, não precisa mexer em cada um: todos já têm endereços dentro dessas faixas (10.x.x.x).
  
  * O pfSense vai simplesmente encaminhar o tráfego com o IP original.
    
  * O Fortinet vai diferenciar cada dispositivo automaticamente.
    

* * *

### 🔹 Onde clicar

1. **Acesse o pfSense** via navegador (`https://ip-do-firewall`).
  
2. Vá em:**Firewall > NAT > Outbound**.
  
3. Se estiver em **Automatic Outbound NAT**, primeiro mude para:
  
  * **Manual Outbound NAT** (ou Hybrid se quiser misturar manual + automático).
    
  * Clique em **Save** e depois **Apply Changes**.
    

* * *

### 🔹 O que vai aparecer

* Vai listar as **regras NAT atuais**.
  
* Normalmente você verá entradas assim:
  

    Interface: WAN
    Source: 10.4.0.0/24
    Translation / Target: WAN address

👉 Essas são justamente as regras que fazem com que tudo da sua rede LAN (10.4.0.x) saia NATeado pelo IP da WAN.

* * *

### 🔹 O que você faz

* **Desative** (ícone do lápis + checkbox “disable” ou o botão vermelho de “disable”).
  
* Ou, se tiver certeza, **apague** essas regras.
  

Depois você pode **criar uma nova regra NAT** (se necessário), apontando para outra interface (ex.: vmnic2 / LAN ou para outro gateway).

* * *

### 🔹 Campos para preencher se for criar regra nova

Quando clicar em **Add** (setinha verde em cima ou embaixo da lista):

* **Interface** → escolha a interface pela qual o tráfego deve sair (ex.: `WAN`, `LAN2`, etc.).
  
* **Source** → sua rede interna (ex.: `10.4.0.0/24`).
  
* **Translation / Address** → escolha:
  
  * `Interface address` (para NAT sair pelo IP da interface escolhida), ou
    
  * Um IP específico (se você tem um IP público dedicado ou outra saída configurada).
    

Salve e clique em **Apply Changes**.

## Explicação:

Boa pergunta, Rafael 👌.

O que você está vendo aí são **as regras NAT automáticas que o pfSense gerou**. Vou te explicar cada uma:

* * *

### 🔎 Regras que aparecem:

1. **Regra ISAKMP (porta 500)** Interface: WAN Source: 10.4.0.0/23, 10.0.0.0/24 Destino porta 500 NAT: WAN address
  

👉 Essa regra é criada para **IPsec VPN (ISAKMP/IKE na porta 500 UDP)**. Se você não usa IPsec (VPN site-to-site ou cliente VPN IPsec), ela não é necessária.

* * *

2. **Regra NAT padrão** Interface: WAN Source: 10.4.0.0/23, 10.0.0.0/24 NAT: WAN address
  

👉 Essa é a regra que **traduziu toda a rede interna para o IP da WAN**. É ela que está “escondendo” todos atrás de **200.196.184.130**.

* * *

### ✅ O que fazer no seu caso

* Se o **objetivo é que o Fortinet veja cada IP interno (10.4.0.x, 10.0.0.x etc.) individualmente**:
  
  * **NÃO replique nenhuma dessas duas regras**.
    
  * Crie **novas regras manuais** para cada rede (10.4.0.0/23, 10.0.0.0/24, 10.3.0.0/24, etc.), mas em **"Translation: None (No NAT)"**.
    
  * Isso vai fazer com que o tráfego saia sem NAT, e o Fortinet faça a autenticação por usuário/IP.
    
* Se **você ainda usa VPN IPsec no pfSense**, aí sim precisaria manter a regra da porta 500 (só nesse caso).
  

* * *

👉 Resumindo para você não se perder:

* **Não precisa copiar essas duas regras automáticas**.
  
* No modo manual, você só cria as que interessam (No NAT para as redes internas).
  
* Só recria a de porta 500 se usar **VPN IPsec**.
  

* * *

Exatamente, Rafael 👍

Você pegou a lógica certinha:

* Se quiser que **cada host tenha autenticação individual no Fortinet**, pode fazer **regras por host** (ex: 10.0.0.121/32, 10.0.0.105/32 etc.).
  
* Se quiser liberar uma **rede inteira**, pode usar a sub-rede (ex: 10.4.0.0/24 ou /23, 10.5.0.0/24, etc.).
  

* * *

### Como ficaria:

1. Vai em:**Firewall > NAT > Saída (Outbound) → Geração de regra NAT de Saída Manual**
  
2. Clique em **Adicionar**.
  
3. Configure a regra para cada rede (ou host):
  
  * **Interface:** WAN
    
  * **Source:**
    
    * Para host único → `10.0.0.121/32`
      
    * Para rede → `10.4.0.0/23` (ou a máscara certa da rede)
      
  * **Destination:** any
    
  * **Translation / Endereço NAT:** `None` (No NAT)
    
  * **Descrição:** algo como `No NAT 10.4.0.0/23 para Fortinet`
    
4. Salvar e aplicar.
  

* * *

Boa escolha 👌 — se você fizer por **rede inteira**, o trabalho cai drasticamente e o comportamento será o mesmo: cada host dentro daquela rede vai ter de autenticar individualmente no Fortinet, porque o NAT não vai mascarar tudo em um único IP.

No seu caso, basta criar **uma regra manual por faixa de rede** que você usa. Por exemplo:

* `10.0.0.0/24`
  
* `10.4.0.0/23` (engloba 10.4.0.xxx e 10.4.1.xxx)
  
* `10.5.0.0/24`
  
* `10.3.0.0/24`
  

* * *

### Resumindo o passo-a-passo:

1. Vá em **Firewall > NAT > Saída**.
  
2. Selecione **Geração de regra NAT de Saída Manual**.
  
3. Apague ou desative as regras que fazem NAT para “WAN Address”.
  
4. Clique em **Adicionar** e crie 1 regra para cada rede:
  
  * **Interface:** WAN
    
  * **Origem (Source):** rede inteira (ex.: `10.4.0.0/23`)
    
  * **Destino (Destination):** any
    
  * **Translation / Endereço NAT:** `None` (sem NAT)
    
  * **Descrição:** `No NAT 10.4.0.0/23 para Fortinet`
    
5. Salvar e aplicar.
  

* * *

Perfeito, Rafael 😎 — vou te mostrar **como preencher cada campo no pfSense** para criar a regra de NAT manual “No NAT” para uma rede inteira, seguindo a lógica que falamos:

* * *

### 📝 Preenchimento sugerido

| Campo | Valor sugerido | Observações |
| --- | --- | --- |
| **Desabilitado** | ❌ desmarcado | A regra precisa estar ativa. |
| **Não faça NAT** | ✅ marcado | Essencial: impede que o NAT transforme o IP da rede interna em IP da WAN. |
| **Dispositivo (Interface)** | WAN | A interface que conecta sua LAN à Internet/Fortinet. |
| **Família de endereço** | IPv4 | Suas redes internas são IPv4. |
| **Protocolo** | Any (Qualquer) | Para abranger todos os tipos de tráfego. |
| **Fonte (Source) / Tipo** | Rede | Vamos mapear **uma rede inteira**, não um host específico. |
| **Fonte / Rede** | ex.: 10.4.0.0/23 | Engloba os IPs 10.4.0.xxx e 10.4.1.xxx. Repita para outras redes depois (10.5.0.0/24, 10.3.0.0/24, etc.). |
| **Fonte / Porta ou Intervalo** | deixar em branco | Só precisa se for filtrar porta específica. |
| **Destino (Destination) / Tipo** | any | Permite acesso a qualquer destino. |
| **Destino / Rede** | any | Igual acima. |
| **Não Inverter o sentido da correspondência de destino** | ✅ marcado (default) | Não altera nada nesse caso. |
| **Tradução / Endereço (Translation / Address)** | None | “No NAT” → crucial para Fortinet ver o IP real. |
| **Tradução / Porta ou Intervalo** | deixar em branco | Só se fosse fazer remapeamento de porta. |
| **Porta estática** | ❌ desmarcado | Não é necessária aqui. |
| **Sem XMLRPC Sync** | ❌ desmarcado | Só se tiver múltiplos pfSense em HA; normalmente deixa desmarcado. |
| **Descrição** | ex.: `No NAT 10.4.0.0/23 para Fortinet` | Para referência futura. |

* * *

### 🔄 Repetir para outras redes

Depois que criar essa primeira regra para 10.4.0.0/23, você repete o processo para:

* 10.0.0.0/24
  
* 10.3.0.0/24
  
* 10.5.0.0/24
  

Cada uma como **regra separada**, sempre marcando **No NAT**.

* * *

💡 **Resultado esperado:**

* Cada dispositivo da rede sairá com **seu próprio IP interno**.
  
* O Fortinet vai **pedir login individual** para cada IP.
  
* A comunicação interna com AD e outros servidores da LAN permanece normal.
  

* * *

Ah, ótimo detalhe, Rafael 👌

No **pfSense**, no campo **Fonte (Source) / Tipo**, você precisa escolher a opção que corresponda à **rede interna que vai sair para a internet**. Explicando cada opção que você listou:

| Opção | Quando usar | No seu caso |
| --- | --- | --- |
| **Qualquer (Any)** | Todos os IPs | Não recomendado aqui, porque você quer regras por rede. |
| **This firewall (o próprio)** | IP do firewall | Não é o que você quer; isso só aplica NAT ao próprio firewall. |
| **Networks or Alias** | Um grupo de IPs ou rede definida via alias | Pode ser usado se você criar um **alias** para todas as redes internas (10.4.0.0/23, 10.0.0.0/24 etc.) → facilita criar uma regra só. |
| **WAN subnets** | Sub-redes da interface WAN | Não, isso é para IPs públicos da WAN. |
| **LAN subnets** | Sub-redes da interface LAN | ✅ Perfeito se você quer aplicar regra a toda a LAN. Ex.: 10.4.0.0/23. |
| **WireGuard networks** | VPN WireGuard | Não se aplica aqui. |

* * *

### 🔹 Conclusão para você

* Se você vai criar uma regra **por rede interna**, escolha:**Fonte (Source) / Tipo → LAN subnets**
  
  * Depois ajusta o **Source / Rede** para o range exato (ex.: 10.4.0.0/23).

💡 **Dica:**Se tiver muitas redes e quiser simplificar, você pode criar **um alias chamado “Redes Internas”** que inclua todas as sub-redes (10.4.0.0/23, 10.0.0.0/24, 10.3.0.0/24 etc.) e depois, no Source / Tipo, escolher **Networks or Alias → Redes Internas**.Isso permite ter **uma única regra NAT** cobrindo todas as redes.

* * *

Se quiser, posso te mostrar **exatamente como preencher a tela do pfSense usando o alias para todas as suas redes**, pra simplificar o processo. Quer que eu faça isso?

* * *

# Cuidados com Logon em Rede:

Boa pergunta, Rafael 👍

Pelo que você descreveu, o servidor **10.4.0.9** parece ser um **controlador de domínio Windows (Active Directory)**. Ele gerencia **usuários e grupos de rede**, autenticação de logon e permissões.

A alteração que você está fazendo (ligada ao VMware/ESXi e redes LAN/WAN) **não deve impactar diretamente o servidor de autenticação do Windows**, desde que alguns pontos sejam garantidos:

1. **A rede LAN continue alcançando o servidor 10.4.0.9**
  
  * Os desktops/usuários precisam conseguir resolver o nome do domínio (ex.: `dominio.local`) e acessar o IP 10.4.0.9.
    
  * Se o gateway, VLAN ou máscara mudar, os clientes podem não enxergar o servidor AD.
    
2. **DNS permanece configurado corretamente**
  
  * O AD do Windows também atua como servidor DNS interno.
    
  * Se os clientes apontarem para outro DNS (por exemplo, o da WAN ou do roteador) em vez do DNS do AD, o login de rede pode falhar.
    
  * Importante: a interface **LAN** do ESXi deve entregar o DNS correto para as VMs.
    
3. **Rotas entre WAN, LAN e ESXi bem definidas**
  
  * Se você está isolando vmnic0 (WAN), vmnic1 (Gerenciamento ESXi) e vmnic2 (LAN), precisa garantir que a rede **10.4.0.x** (onde está o AD 10.4.0.9) continue acessível para as VMs que participam do domínio.
4. **Firewall/Regras de VLAN**
  
  * Se houver firewall no meio (pfSense, Mikrotik, etc.), certifique-se de que portas usadas pelo AD (como 389 LDAP, 445 SMB, 53 DNS, 88 Kerberos) não sejam bloqueadas.

👉 Em resumo:

* **Se a rede 10.4.0.x não mudar**, os clientes continuarão autenticando normalmente no servidor 10.4.0.9.
  
* **Se mudar o range ou gateway**, é preciso revisar o DHCP/DNS para garantir que os PCs continuem encontrando o AD.
  

* * *

Ah, Rafael, agora ficou claro o que está acontecendo 👍

No pfSense, se o campo **“Rede de origem para o mapeamento NAT de saída”** está **bloqueado e não permite edição**, isso acontece porque **você ainda está no modo de NAT Automático**.

* **Modo Automático (Automatic Outbound NAT)** → o pfSense gera as regras sozinho e **não permite editar manualmente** o source (origem).
  
* Para conseguir preencher a rede que você quer, é preciso **mudar para o modo Manual** (ou Hybrid se quiser misturar automático + manual).
  

* * *

### 🔹 Como fazer

1. Vá em **Firewall > NAT > Outbound**.
  
2. No topo da página, você verá:
  
  * Automatic Outbound NAT
    
  * Hybrid Outbound NAT
    
  * Manual Outbound NAT
    
3. **Selecione “Manual Outbound NAT”**.
  
4. Clique em **Save** e depois em **Apply Changes**.
  
5. Agora, quando você clicar em **Add** para criar uma regra, o campo **Rede de origem para o mapeamento NAT de saída** estará **liberado para edição**, e você poderá colocar a rede exata (ex.: 10.4.0.0/23).
  

* * *

💡 **Dica extra:**Se você não quiser perder as regras automáticas existentes que ainda funcionam para VPNs ou outras coisas, pode usar **Hybrid Outbound NAT**.

* Nesse modo, você mantém regras automáticas e cria suas regras manuais “No NAT” para as redes internas, sem quebrar nada.

* * *

Se quiser, posso te montar **um passo a passo exato no pfSense, já com todas as suas redes internas, pronto para criar as regras “No NAT”**. Isso evita erro e deixa tudo pronto para o Fortinet pedir login individual.

* * *

Compreendo perfeitamente. O erro que você está encontrando, onde o firewall **não permite salvar** a regra quando o campo **Destino / Rede** está configurado como **"any"** (qualquer), é um comportamento de segurança ou de sintaxe comum em muitos sistemas de firewall, especialmente quando se lida com regras de NAT mais específicas.

Apesar de o "Preenchimento sugerido" indicar "any", o erro sugere que é necessário **especificar um destino válido** ou que o campo **"any"** não é o valor aceito pelo sistema para representar "qualquer rede/IP".

* * *

1. O Problema: Como o Sistema Interpreta "Any"

* * *

Se o seu sistema é um **pfSense**, por exemplo (dado o termo "XMLRPC Sync"), ou outro firewall de código aberto, o campo de rede/IP geralmente espera um valor formatado como CIDR (por exemplo, `192.168.1.0/24`).

* O valor literal **`any`** pode não ser aceito.
  
* O valor correto para representar "qualquer endereço IPv4" na maioria dos sistemas é: **`0.0.0.0/0`**.
  

2. Solução Recomendada: Substitua "any" por `0.0.0.0/0`

* * *

Tente configurar a regra de NAT de saída (Source NAT) da seguinte forma, focando em como configurar o destino:

| Campo | Valor Sugerido | Explicação |
| --- | --- | --- |
| **Destino (Destination) / Tipo** | Rede | Mantemos o tipo de rede. |
| **Destino / Rede** | **`0.0.0.0/0`** | **Tente este valor.** Ele representa "Todas as redes IPv4" e é a sintaxe universalmente aceita pelo TCP/IP para "any". |

**Por que isso resolve?**

Ao usar `0.0.0.0/0`, você está fornecendo um valor CIDR válido para o campo de rede, o que satisfaz o requisito de sintaxe do firewall, mas funcionalmente ainda significa "qualquer destino".

* * *

3. Verifique Outros Campos (em caso de erro persistente)

* * *

Se a substituição por `0.0.0.0/0` não funcionar, o erro pode estar sendo disparado por outro campo que precisa de um valor específico em conjunto com o destino "any":

* **Porta de Destino:** Se você deixou a porta de destino em branco, e o sistema exige que _alguma_ porta seja definida junto com o destino `0.0.0.0/0`, tente colocar o valor **`any`** ou **`*`** (asterisco) no campo de porta.
  
* **"Não faça NAT" (Tradução / Endereço):** Certifique-se de que a opção **"None"** ou **"No NAT"** na seção **Tradução / Endereço** esteja configurada corretamente. Em alguns firewalls, para que a regra de "Não fazer NAT" funcione (como você deseja, para o Fortinet ver o IP real), é preciso que o _Destino_ seja o mais abrangente possível (`0.0.0.0/0`).
  

**Minha sugestão final é:** Use **`0.0.0.0/0`** no campo **Destino / Rede**. Se isso resolver, você poderá salvar a regra e ela funcionará conforme o esperado (tráfego da sua LAN passando para o Fortinet sem tradução de IP de origem).

* * *

Compreendi perfeitamente o seu problema. A situação é clássica: o **pfSense está fazendo NAT de Saída (Source NAT)** em todo o tráfego da sua rede interna, mascarando os IPs reais de origem.

O Fortinet, que está mais à frente, só "vê" o endereço IP da interface WAN do pfSense (seu IP público, `200.196.184.130`) para todos os usuários. Por isso, quando a primeira pessoa se autentica no portal do Fortinet, todos os demais são liberados, pois o Fortinet associa o IP `200.196.184.130` como já autenticado.

A solução é justamente o que você estava tentando fazer: **desabilitar o NAT de Saída (Source NAT ou SNAT) no pfSense** para as redes que o Fortinet precisa inspecionar.

* * *

Solução no pfSense: Criar Regras de "No NAT" (Manual Outbound NAT)

Para que o Fortinet enxergue os IPs reais, você deve mudar o modo de NAT de saída no pfSense e criar regras explícitas de **"Não Fazer NAT"** (conhecidas como "No NAT" ou "Bypass") para as suas redes internas, direcionando-as para a interface WAN.

### 1. Mudar o Modo de NAT

Primeiro, você deve mudar a configuração de NAT de Saída de **Automático** para **Manual** (se já não tiver feito):

1. No pfSense, vá para **Firewall** > **NAT**.
  
2. Clique na aba **Outbound**.
  
3. Selecione a opção **Manual Outbound NAT rule generation (AON - Advanced Outbound NAT)**.
  
4. Clique em **Save** (Salvar) e depois em **Apply Changes** (Aplicar Alterações).
  

Ao fazer isso, o pfSense deve copiar as regras automáticas existentes para o modo manual.

### 2. Criar a Regra de "No NAT"

Agora, você precisa criar uma nova regra (ou editar a primeira regra, dependendo do seu pfSense) para **excluir** suas redes de serem traduzidas:

| Campo | Valor | Ação / Observação |
| --- | --- | --- |
| **Desabilitado** | **❌ Desmarcado** | A regra deve estar ativa. |
| **Não faça NAT** | **✅ Marcado** | **CRUCIAL.** Isso instrui o pfSense a não tocar no IP de origem. |
| **Interface** | **WAN** | A interface de saída para a rede externa (Fortinet). |
| **Família de Endereço** | **IPv4** | Suas redes são IPv4. |
| **Protocolo** | **any** | Para cobrir todo o tráfego. |
| **Fonte (Source) / Tipo** | **Rede** | Para mapear sub-redes inteiras. |
| **Fonte (Source) / Rede** | **ex.: `10.4.0.0/23`** | Repita o processo para cada sub-rede interna (`10.3.0.0/24`, `10.5.0.0/24`, etc.) OU crie um _Alias_ (apelido) que contenha todas elas e use o _Alias_ aqui. |
| **Destino (Destination) / Tipo** | **any** | Permite acesso a qualquer destino. |
| **Destino (Destination) / Rede** | **`0.0.0.0/0`** | **USE ESTE VALOR!** Como vimos, ele representa "qualquer rede IPv4" e **resolve o erro de não salvar** o campo vazio ou com "any" literal. |
| **Tradução / Endereço** | **None** | Confirma a ação de "Não fazer NAT". |

### 3. Ordem da Regra (Muito Importante)

No modo Manual, o pfSense processa as regras de NAT de Saída **de cima para baixo**.

* A regra de **"No NAT"** que você está criando deve estar **ACIMA** das regras padrões que fazem a tradução (`Mapear para endereço de interface`).
  
* **Arraste sua nova regra** de `10.4.0.0/23` com a opção "Não faça NAT" marcada para o **topo da lista** de regras de _Outbound NAT_.
  

Ao fazer isso, o tráfego das suas redes internas **corresponderá à primeira regra**, será excluído do NAT de Saída e os IPs reais (10.x.x.x) serão enviados ao Fortinet. Todo o restante do tráfego seguirá pelas regras padrão abaixo e continuará sendo NATeado normalmente. Muito obrigado por enviar as imagens! Elas confirmam que você está no **pfSense** e mostram exatamente onde o problema acontece.

Como você está no pfSense, a solução para o campo **Destino** não aceitar o valor **"Qualquer"** é simples, mas requer uma pequena mudança de tipo, forçando-o a aceitar a sintaxe de rede.

O erro acontece porque, quando você define o **Tipo** de Destino como **"Qualquer"**, o pfSense não sabe qual valor usar no campo de Rede/Subnet (que está logo à direita).

* * *

Solução Rápida: Altere o Tipo para "Network or Alias"

Para resolver isso, você deve mudar o **Tipo de Destino** para **"Network or Alias"** e, em seguida, usar a notação universal de rede que significa **"Qualquer"**:

1. No campo **Destino**, mude o **Tipo** de **"Qualquer"** para **"Network or Alias"**.
  
2. Ao mudar o tipo, o pfSense ativará o campo de texto/dropdown de rede à direita.
  
3. Nesse novo campo de rede, digite: **`0.0.0.0/0`**
  

### Por que `0.0.0.0/0` funciona?

* O endereço **`0.0.0.0/0`** é a notação CIDR que significa **"toda a rede IPv4"**. É o equivalente técnico de "Qualquer".
  
* Ao mudar o **Tipo** para "Network or Alias", você força o pfSense a esperar e aceitar uma notação de rede válida (`0.0.0.0/0`), o que resolverá o erro ao tentar salvar.
  

### Configuração Final Sugerida

| Campo | Configuração | Observações |
| --- | --- | --- |
| **Não faça NAT** | **✅ Habilitado** | Perfeito, é isso que você precisa. |
| **Fonte / Tipo** | Network or Alias | Correto. |
| **Fonte / Rede** | **`10.4.0.0/23`** | Correto. |
| **Destino / Tipo** | **Network or Alias** | **MUDE PARA ESTE!** |
| **Destino / Rede** | **`0.0.0.0/0`** | **USE ESTE VALOR!** (Funciona como "Qualquer") |
| **Tradução / Endereço** | **None (Ou deixe em branco)** | O **"Não faça NAT"** já cuida disso. |

Faça essa alteração no campo **Destino** e tente salvar novamente. A regra deve ser aceita e, o mais importante, deve ser posicionada **no topo da lista de regras de NAT de Saída** (Outbound NAT) para que seja processada antes das regras padrões que fazem a tradução.

* * *

Ótimo que você enviou a tela! Isso torna o diagnóstico muito mais preciso.

Você está no caminho certo! Usar **`0.0.0.0/0`** é a notação correta. O erro **"Um destino válido deve ser especificado"** está sendo causado pela forma como o **pfSense interpreta o campo de _máscara de sub-rede_** (o `/24` no lado direito) quando você digita `0.0.0.0/0`.

Quando você digita um endereço de rede completo (como `10.4.0.0/23`) nos dois campos, o campo de máscara (`/23` ou `/24`) é preenchido automaticamente. No entanto, o `0.0.0.0/0` é especial.

### A Solução

O pfSense está esperando que você **digite o valor `0`** no campo de máscara de sub-rede.

1. No campo **Destino / Tipo** você já escolheu **`Network or Alias`** e digitou **`0.0.0.0`** no campo de rede.
  
2. Agora, no campo à direita que está como **`/24`**, **apague o `24` e digite `0`**.
  

| Campo | Configuração | Ação Necessária |
| --- | --- | --- |
| **Destino / Rede** | **`0.0.0.0`** | Mantenha este valor. |
| **Destino / Máscara** | **`/ 0`** | **MUDE PARA ZERO.** |

Ao especificar explicitamente **`/0`**, você informa ao pfSense que a máscara para `0.0.0.0` é zero, o que representa o bloco completo de endereços IPv4 ("Qualquer").

* * *

### Resumo para Correção:

Corrija o Destino da seguinte forma:

1. **Destino (Rede):** `0.0.0.0`
  
2. **Destino (Máscara):** `0`
  

Tente salvar a regra novamente. Deve funcionar! Lembre-se, esta regra precisa estar no **topo da lista** na aba **Outbound NAT**.

* * *

Solução Encontrada e Próximos Passos

Que bom que você encontrou um valor que o pfSense aceitou! Você está quase lá.

É importante notar que, embora o pfSense não mostre o valor `/0` na lista suspensa (o que é comum), o uso de **`0.0.0.0/24`** no campo Destino é uma **gambiarra que o sistema aceita**, mas pode não ser a solução ideal para o seu objetivo.

* * *

### Entendendo a "Gambiarra" do Destino

1. **O que o pfSense aceitou:** Ao usar **`0.0.0.0/24`**, você está dizendo para a regra aplicar o "No NAT" (Não fazer NAT) somente ao tráfego cujo destino seja a rede **`0.0.0.0` até `0.0.0.255`**.
  
2. **O que você realmente precisa:** Para que o Fortinet veja **todos** os IPs internos, você precisa de uma regra que abranja **qualquer destino na internet**, que é o **`0.0.0.0/0`**.
  

### Como forçar o `/0`

A lista suspensa não mostra o `/0`, mas você pode **digitar o número `0`** no campo, se ele for um campo de texto editável (e não um _dropdown_ estrito).

* **Tente novamente:** Clique no campo da máscara de sub-rede (`/24`) e **digite o número `0`** em vez de selecioná-lo na lista. Se o campo aceitar texto, ele salvará corretamente como `0.0.0.0/0`.

Se, no entanto, o campo for _apenas_ de seleção (dropdown):

### ⚠️ Próximo Passo Crítico: Reverter a Regra Padrão

O pfSense tem uma regra padrão que faz o NAT para o endereço da WAN. Esta regra padrão geralmente está abaixo de todas as regras manuais.

**Para a sua regra de "No NAT" funcionar**, ela precisa ser processada **primeiro**.

1. **Vá para a lista principal de regras de NAT de Saída** (**Firewall > NAT > Outbound**).
  
2. Localize a regra que você acabou de criar (`10.0.0.0/24 No NAT para Fortinet`).
  
3. **Arraste esta regra para o topo da lista** (acima de todas as outras regras padrão que fazem o NAT para o IP da WAN).
  
4. Clique em **Save** (Salvar) e **Apply Changes** (Aplicar Alterações).
  

Se a sua regra estiver no topo, o pfSense vai ver o tráfego da rede `10.0.0.0/24` e decidir **NÃO fazer NAT** (mesmo com o destino `/24`), e o tráfego seguirá com o IP de origem real para o Fortinet.

* * *

### Por Fim: O Fortinet

Depois de garantir que o tráfego está saindo do pfSense com o IP `10.x.x.x` real, você precisa configurar o **Fortinet** para:

1. **Aceitar** o tráfego da rede `10.x.x.x`.
  
2. Garantir que as regras de segurança/portal do Fortinet estejam preparadas para ver e inspecionar **múltiplos IPs de origem privados** (em vez de apenas um IP público).
  

* * *

Essa é uma excelente observação e um procedimento de segurança muito inteligente! Você está certo em querer desativar as regras antigas primeiro para garantir que a transição seja limpa.

* * *

1. O Motivo da Mão Vermelha

* * *

A **mãozinha vermelha** (que parece um ícone de "parada" ou "proibido") não está ali porque você não mudou o modo NAT, mas sim porque você acabou de **criar regras e não as ativou**.

No pfSense (e em muitos firewalls), quando você adiciona ou edita regras, você precisa de duas etapas:

1. **Salvar** a regra (o que você fez ao sair do editor).
  
2. **Aplicar as Mudanças** no sistema para que elas entrem em vigor.
  

**A mãozinha vermelha indica que a regra está desativada no sistema.**

### Ação Necessária

Apesar de a sua regra estar correta no editor, ela não está em uso. Para ativá-la:

1. **Mude o modo de NAT de Saída:** Vá para **Firewall > NAT > Outbound** e selecione **Manual Outbound NAT rule generation (AON)**.
  
2. Clique em **Save** e depois em **Apply Changes**.
  

Ao fazer isso, a mãozinha vermelha provavelmente desaparecerá, e a regra entrará em vigor.

* * *

2. Ponto Crítico: Ordem das Regras (NAT de Saída Manual)

* * *

Você está correto em se preocupar com as configurações antigas.

Ao mudar para o modo **Manual Outbound NAT (AON)**, o pfSense fará o seguinte:

1. Ele **copiará** as regras automáticas de NAT para a lista manual.
  
2. A partir desse momento, ele **processará as regras de cima para baixo**.
  

### O Seu Plano (Revisado)

Para garantir que o NAT seja desativado para suas redes, siga estes passos na aba **Outbound NAT**:

1. **Mude para o modo Manual (AON)**, salve e aplique.
  
2. Localize as regras de **"No NAT"** que você acabou de criar (as 4 regras `10.x.x.x` que têm **"NO NAT"** na coluna **Endereço NAT**).
  
3. **Arraste e Solte estas 4 regras para o TOPO da lista.** Elas devem ser as primeiras regras a serem processadas.
  
4. Localize a regra padrão que está fazendo o NAT para o seu IP público (`200.196.184.130`). Esta regra provavelmente usa `any` ou sua **LAN Subnets** na origem e o **Endereço da Interface WAN** na tradução.
  
5. **Desative (ou exclua) as regras padrão** que fazem o NAT para as suas redes `10.x.x.x`. _Por precaução, eu sugeriria desativar em vez de excluir, caso você precise reverter._
  
6. Clique em **Save** e **Apply Changes**.
  

Ao colocar suas 4 regras de **"NO NAT"** no topo, o tráfego dessas redes será enviado diretamente para o Fortinet com seus IPs internos, resolvendo o seu problema de autenticação individual!

* * *

Informações:

Você está no caminho certo e essa é uma pergunta **fundamental** para garantir que tudo funcione como planejado!

Sim, sua observação sobre a mudança do **Endereço NAT** está perfeitamente correta.

* * *

O Significado da Mudança no Endereço NAT

| Regra | Endereço NAT | Significado Funcional |
| --- | --- | --- |
| **Regras Atuais** (padrão) | **WAN address** | O pfSense está traduzindo o endereço de origem (ex: `10.4.0.33`) para o **IP público da sua WAN** (`200.196.184.130`). _O Fortinet só vê esse IP público._ |
| **Suas Novas Regras** | **NO NAT** | O pfSense está instruído a **NÃO** traduzir o endereço de origem. _O Fortinet verá o IP real do usuário_ (ex: `10.4.0.33`). |

**Portanto, a resposta é: Sim, o Endereço NAT _tem_ que ser "NO NAT" para suas novas regras.**

Este é o objetivo principal: desabilitar o SNAT (Source NAT) para que o Fortinet receba o tráfego com o IP de origem real do usuário (`10.x.x.x`) e possa exigir a autenticação individual.

* * *

Próximos Passos Críticos (A Mão Vermelha)

Olhando sua imagem mais recente, você já:

1. Criou as regras de "No NAT" (elas estão no topo).
  
2. Manteve as regras antigas desativadas (a mãozinha vermelha).
  

O que falta para as regras entrarem em vigor é exatamente o que o aviso em verde no topo da tela diz:

1. **Mudar o Modo de NAT:** O modo atual ainda é **"Automática geração de regra NAT de saída"** (o primeiro botão está selecionado). Você precisa selecionar o botão **"Geração de regra NAT de saída Manual (AON)"** (o terceiro botão).
  
2. **Aplicar Mudanças:** Depois de mudar para o modo Manual, clique em **Salvar** e, em seguida, no botão verde **"Aplicar Mudanças"** no canto superior direito.
  

Ao aplicar as mudanças no modo **Manual**, suas 4 regras de "NO NAT" (que já estão no topo da lista) serão ativadas, a mãozinha vermelha sumirá, e o pfSense começará a enviar o tráfego com os IPs reais para o Fortinet.

---

Essa é uma ótima pergunta, e é crucial para entender como o pfSense está funcionando neste momento!

* * *

O Significado dos Ícones no pfSense

Você identificou corretamente a diferença entre as seções. O que está valendo (ou seja, ativo e sendo processado) depende da seção onde a regra se encontra, e dos ícones.

### 1. O "X" no Quadrado (A Coluna de Ações)

O "X" que você vê no quadrado na primeira coluna (na seção **Mapeamentos**):

* Este ícone **❌** é o _checkbox_ de **seleção** para a ação **"Excluir"** (o botão vermelho "Excluir" está logo abaixo).
  
* Ele não tem nada a ver com a ativação ou desativação da regra. Ele serve apenas para que você possa marcar várias regras de uma vez e deletá-las em massa.
  

### 2. A Mãozinha Vermelha (Status da Regra)

* A **mãozinha vermelha** (o ícone de "pare" na segunda coluna) é o que realmente indica que a regra está **desativada/inativa**.
  
* Como você já fez a mudança para o modo Manual e suas regras de **"NO NAT"** estão no topo, esta mãozinha vermelha significa que **nenhuma dessas regras na seção "Mapeamentos" está em vigor**.
  

### 3. As Regras que Estão Valendo AGORA

Você está **parcialmente correto**.

**O que está valendo neste momento são as regras da seção "Automatic Rules".**

* A seção **"Automatic Rules"** só existe porque você está no modo **"Automática geração de regra NAT de saída"** (o botão que está selecionado na sua tela).
  
* Neste modo, o pfSense **ignora** todas as regras em **"Mapeamentos"** (por isso a mão vermelha) e usa apenas as regras que ele gera automaticamente (as regras com o **✓ _check_** verde).
  

* * *

Ação Final para Resolver o Problema

Para que suas regras de **"NO NAT"** entrem em vigor, você precisa sair do modo Automático e ir para o modo Manual, que força o sistema a usar as regras de "Mapeamentos" que você criou:

1. Selecione a opção **"Geração de regra NAT de saída Manual (AON - Avançado)"**.
  
2. Clique no botão **Salvar** logo abaixo do Modo NAT de Saída.
  
3. Clique no botão verde **"Aplicar Mudanças"** no canto superior direito.
  

Ao aplicar as mudanças no modo Manual, você verá o seguinte:

* A seção **"Automatic Rules"** **desaparecerá** (pois o pfSense não gera mais regras automáticas).
  
* As regras em **"Mapeamentos"** serão ativadas (a mãozinha vermelha sumirá e será substituída por um _check_ ou um ícone verde).
  

Neste ponto, suas 4 regras de **"NO NAT"** estarão no topo e em vigor, e o Fortinet passará a ver os IPs internos dos seus usuários.

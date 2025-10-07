### 📌 Situação atual

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

Você **não quer** que o login no Windows libere automaticamente a internet (como seria com FSSO).  
👉 Você **quer manter o portal de login do Fortinet (captive portal)**, mas garantir que **cada usuário/dispositivo** seja obrigado a logar individualmente.  
O problema atual é que, quando um usuário digita usuário/senha no portal do Fortinet, **toda a rede fica liberada**, porque o Fortinet só está vendo **um único IP** (o do pfSense NAT).

* * *

🔍 Onde está o problema
-----------------------

O **pfSense está fazendo NAT (masquerade)**.  
Isso significa que:

* Todos os PCs da sua LAN → saem para a WAN com o **mesmo IP** (10.0.0.1, por exemplo).

* O Fortinet enxerga apenas esse IP e entende que o login vale para todos.

Por isso, depois que **um usuário autentica**, o Fortinet associa o IP (do pfSense) a esse login, e libera a navegação de todos que passam por ele.

* * *

✅ Soluções possíveis
--------------------

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
------------

O **problema não está no Fortinet**, e sim no **pfSense fazendo NAT**.  
Se você quer que **cada usuário veja a tela de login do Fortinet individualmente**, precisa que o Fortinet enxergue os IPs de cada máquina.  
👉 Isso só acontece se **o pfSense não mascarar (NAT)** todos os clientes em um único IP.

* * *

Boa, Rafael 👌, a tela que você mostrou é do **Encaminhamento de Portas (Port Forward)** → não é ali que a gente mexe para resolver seu problema de autenticação no Fortinet.

O ponto que precisamos ver é o **Outbound NAT**, porque é nele que o pfSense está transformando todos os IPs internos em **um único IP de saída** (o que engana o Fortinet).

* * *

🔧 Onde mexer no pfSense
------------------------

1. Vá no menu:
      Firewall > NAT > Saída (Outbound)

2. Você verá três opções de modo NAT:
   
   * **Automatic Outbound NAT** → padrão, mascara tudo em um IP só.
   
   * **Hybrid Outbound NAT** → permite misturar regras automáticas e manuais.
   
   * **Manual Outbound NAT** → você controla totalmente as regras.

👉 Para o seu caso, escolha **Manual Outbound NAT**.

* * *

🔧 O que mudar
--------------

* Hoje o pfSense está pegando tudo da sua rede interna (ex: `10.4.0.0/24`, `10.0.0.0/24` etc.) e trocando para o IP WAN dele.

* O que você precisa: **desativar esse NAT**, para que o tráfego siga com o IP real de cada máquina até o Fortinet.

No modo **Manual**:

1. Vai aparecer uma lista de regras automáticas.

2. Você pode **remover ou desativar** as regras que fazem “masquerade” (source = sua rede LAN → translation = interface address).

3. Depois aplique e salve.

* * *

📌 E sobre seus 600 IPs fixos
-----------------------------

* Não precisa mexer em cada um manualmente.

* Se eles já estão atribuídos por IP fixo/MAC no pfSense, **eles continuarão sendo usados normalmente**.

* A diferença é que, em vez do pfSense mascarar todos como **um IP só**, ele vai deixar **cada IP sair como ele é**.

* O Fortinet, então, vai enxergar **600 conexões diferentes**, e para cada uma vai pedir login individual no portal (do jeito que você precisa).

* * *

⚠️ Atenção: ao desativar o NAT, o pfSense passa a funcionar mais como **roteador** do que como **firewall mascarador**.  
Isso significa:

* O Fortinet precisa ter rotas para essas redes internas (ex: saber chegar em `10.4.0.0/16`).

* Normalmente, basta colocar o pfSense como **gateway** no Fortinet para essas redes.

* * *

Show, Rafael 🙌 agora ficou bem claro com essa tela.

Hoje o seu **pfSense está em modo "Automatic Outbound NAT"**, e por isso **todos os IPs internos (10.4.x.x, 10.3.x.x, 10.5.x.x, etc.) saem NATeados como o endereço da WAN** (200.196.184.130).

👉 Isso explica por que o Fortinet só enxerga **um IP (o do pfSense)** e, quando um usuário loga no portal, todos passam a ter internet.

* * *

✅ O que você precisa
--------------------

Que o Fortinet enxergue os **IPs reais** (10.4.0.33, 10.5.0.10, etc.), para poder exigir autenticação individual.

* * *

🔧 Ajustando o pfSense
----------------------

1. Vá em **Firewall > NAT > Saída (Outbound)**.

2. Troque de **Automática** para **Manual Outbound NAT**.
   
   * Vai aparecer a lista de regras atuais (as duas que você vê aí: “Regras criadas automaticamente…”).

3. **Apague** (ou desative) as regras que fazem NAT para “WAN Address”.
   
   * Essas regras são justamente as que transformam tudo em 200.196.184.130.

4. Salve e aplique.

* * *

🔄 Depois disso
---------------

* O pfSense não vai mais mascarar todos os IPs → ele passa a **rotear**.

* Ou seja, o desktop `10.4.0.33` vai sair para a internet como **10.4.0.33** mesmo.

* O Fortinet vai receber essa conexão e pedir login **só para esse IP/dispositivo**.

* Outro desktop (`10.5.0.20`, por exemplo) terá que fazer login separado, porque o Fortinet vai enxergar como origem diferente.

* * *

⚠️ Pontos importantes
---------------------

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

2. Vá em:  
   **Firewall > NAT > Outbound**.

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

1. **Regra ISAKMP (porta 500)**
      Interface: WAN
      Source: 10.4.0.0/23, 10.0.0.0/24
      Destino porta 500
      NAT: WAN address
   👉 Essa regra é criada para **IPsec VPN (ISAKMP/IKE na porta 500 UDP)**.  
   Se você não usa IPsec (VPN site-to-site ou cliente VPN IPsec), ela não é necessária.

* * *

2. **Regra NAT padrão**
      Interface: WAN
      Source: 10.4.0.0/23, 10.0.0.0/24
      NAT: WAN address
   👉 Essa é a regra que **traduziu toda a rede interna para o IP da WAN**.  
   É ela que está “escondendo” todos atrás de **200.196.184.130**.

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

1. Vai em:  
   **Firewall > NAT > Saída (Outbound) → Geração de regra NAT de Saída Manual**

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

| Campo                                                    | Valor sugerido                          | Observações                                                                                               |
| -------------------------------------------------------- | --------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Desabilitado**                                         | ❌ desmarcado                            | A regra precisa estar ativa.                                                                              |
| **Não faça NAT**                                         | ✅ marcado                               | Essencial: impede que o NAT transforme o IP da rede interna em IP da WAN.                                 |
| **Dispositivo (Interface)**                              | WAN                                     | A interface que conecta sua LAN à Internet/Fortinet.                                                      |
| **Família de endereço**                                  | IPv4                                    | Suas redes internas são IPv4.                                                                             |
| **Protocolo**                                            | Any (Qualquer)                          | Para abranger todos os tipos de tráfego.                                                                  |
| **Fonte (Source) / Tipo**                                | Rede                                    | Vamos mapear **uma rede inteira**, não um host específico.                                                |
| **Fonte / Rede**                                         | ex.: 10.4.0.0/23                        | Engloba os IPs 10.4.0.xxx e 10.4.1.xxx. Repita para outras redes depois (10.5.0.0/24, 10.3.0.0/24, etc.). |
| **Fonte / Porta ou Intervalo**                           | deixar em branco                        | Só precisa se for filtrar porta específica.                                                               |
| **Destino (Destination) / Tipo**                         | any                                     | Permite acesso a qualquer destino.                                                                        |
| **Destino / Rede**                                       | any                                     | Igual acima.                                                                                              |
| **Não Inverter o sentido da correspondência de destino** | ✅ marcado (default)                     | Não altera nada nesse caso.                                                                               |
| **Tradução / Endereço (Translation / Address)**          | None                                    | “No NAT” → crucial para Fortinet ver o IP real.                                                           |
| **Tradução / Porta ou Intervalo**                        | deixar em branco                        | Só se fosse fazer remapeamento de porta.                                                                  |
| **Porta estática**                                       | ❌ desmarcado                            | Não é necessária aqui.                                                                                    |
| **Sem XMLRPC Sync**                                      | ❌ desmarcado                            | Só se tiver múltiplos pfSense em HA; normalmente deixa desmarcado.                                        |
| **Descrição**                                            | ex.: `No NAT 10.4.0.0/23 para Fortinet` | Para referência futura.                                                                                   |

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

| Opção                         | Quando usar                                | No seu caso                                                                                                                           |
| ----------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Qualquer (Any)**            | Todos os IPs                               | Não recomendado aqui, porque você quer regras por rede.                                                                               |
| **This firewall (o próprio)** | IP do firewall                             | Não é o que você quer; isso só aplica NAT ao próprio firewall.                                                                        |
| **Networks or Alias**         | Um grupo de IPs ou rede definida via alias | Pode ser usado se você criar um **alias** para todas as redes internas (10.4.0.0/23, 10.0.0.0/24 etc.) → facilita criar uma regra só. |
| **WAN subnets**               | Sub-redes da interface WAN                 | Não, isso é para IPs públicos da WAN.                                                                                                 |
| **LAN subnets**               | Sub-redes da interface LAN                 | ✅ Perfeito se você quer aplicar regra a toda a LAN. Ex.: 10.4.0.0/23.                                                                 |
| **WireGuard networks**        | VPN WireGuard                              | Não se aplica aqui.                                                                                                                   |

* * *

### 🔹 Conclusão para você

* Se você vai criar uma regra **por rede interna**, escolha:  
  **Fonte (Source) / Tipo → LAN subnets**
  
  * Depois ajusta o **Source / Rede** para o range exato (ex.: 10.4.0.0/23).

💡 **Dica:**  
Se tiver muitas redes e quiser simplificar, você pode criar **um alias chamado “Redes Internas”** que inclua todas as sub-redes (10.4.0.0/23, 10.0.0.0/24, 10.3.0.0/24 etc.) e depois, no Source / Tipo, escolher **Networks or Alias → Redes Internas**.  
Isso permite ter **uma única regra NAT** cobrindo todas as redes.

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

💡 **Dica extra:**  
Se você não quiser perder as regras automáticas existentes que ainda funcionam para VPNs ou outras coisas, pode usar **Hybrid Outbound NAT**.

* Nesse modo, você mantém regras automáticas e cria suas regras manuais “No NAT” para as redes internas, sem quebrar nada.

* * *

Se quiser, posso te montar **um passo a passo exato no pfSense, já com todas as suas redes internas, pronto para criar as regras “No NAT”**. Isso evita erro e deixa tudo pronto para o Fortinet pedir login individual.

---

Compreendo perfeitamente. O erro que você está encontrando, onde o firewall **não permite salvar** a regra quando o campo **Destino / Rede** está configurado como **"any"** (qualquer), é um comportamento de segurança ou de sintaxe comum em muitos sistemas de firewall, especialmente quando se lida com regras de NAT mais específicas.

Apesar de o "Preenchimento sugerido" indicar "any", o erro sugere que é necessário **especificar um destino válido** ou que o campo **"any"** não é o valor aceito pelo sistema para representar "qualquer rede/IP".

* * *

1. O Problema: Como o Sistema Interpreta "Any"

----------------------------------------------

Se o seu sistema é um **pfSense**, por exemplo (dado o termo "XMLRPC Sync"), ou outro firewall de código aberto, o campo de rede/IP geralmente espera um valor formatado como CIDR (por exemplo, `192.168.1.0/24`).

* O valor literal **`any`** pode não ser aceito.

* O valor correto para representar "qualquer endereço IPv4" na maioria dos sistemas é: **`0.0.0.0/0`**.
2. Solução Recomendada: Substitua "any" por `0.0.0.0/0`

-------------------------------------------------------

Tente configurar a regra de NAT de saída (Source NAT) da seguinte forma, focando em como configurar o destino:

| Campo                            | Valor Sugerido  | Explicação                                                                                                             |
| -------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Destino (Destination) / Tipo** | Rede            | Mantemos o tipo de rede.                                                                                               |
| **Destino / Rede**               | **`0.0.0.0/0`** | **Tente este valor.** Ele representa "Todas as redes IPv4" e é a sintaxe universalmente aceita pelo TCP/IP para "any". |

**Por que isso resolve?**

Ao usar `0.0.0.0/0`, você está fornecendo um valor CIDR válido para o campo de rede, o que satisfaz o requisito de sintaxe do firewall, mas funcionalmente ainda significa "qualquer destino".

* * *

3. Verifique Outros Campos (em caso de erro persistente)

--------------------------------------------------------

Se a substituição por `0.0.0.0/0` não funcionar, o erro pode estar sendo disparado por outro campo que precisa de um valor específico em conjunto com o destino "any":

* **Porta de Destino:** Se você deixou a porta de destino em branco, e o sistema exige que _alguma_ porta seja definida junto com o destino `0.0.0.0/0`, tente colocar o valor **`any`** ou **`*`** (asterisco) no campo de porta.

* **"Não faça NAT" (Tradução / Endereço):** Certifique-se de que a opção **"None"** ou **"No NAT"** na seção **Tradução / Endereço** esteja configurada corretamente. Em alguns firewalls, para que a regra de "Não fazer NAT" funcione (como você deseja, para o Fortinet ver o IP real), é preciso que o _Destino_ seja o mais abrangente possível (`0.0.0.0/0`). 

**Minha sugestão final é:** Use **`0.0.0.0/0`** no campo **Destino / Rede**. Se isso resolver, você poderá salvar a regra e ela funcionará conforme o esperado (tráfego da sua LAN passando para o Fortinet sem tradução de IP de origem).

---

Compreendi perfeitamente o seu problema. A situação é clássica: o **pfSense está fazendo NAT de Saída (Source NAT)** em todo o tráfego da sua rede interna, mascarando os IPs reais de origem.

O Fortinet, que está mais à frente, só "vê" o endereço IP da interface WAN do pfSense (seu IP público, `200.196.184.130`) para todos os usuários. Por isso, quando a primeira pessoa se autentica no portal do Fortinet, todos os demais são liberados, pois o Fortinet associa o IP `200.196.184.130` como já autenticado.

A solução é justamente o que você estava tentando fazer: **desabilitar o NAT de Saída (Source NAT ou SNAT) no pfSense** para as redes que o Fortinet precisa inspecionar.

* * *

Solução no pfSense: Criar Regras de "No NAT" (Manual Outbound NAT)
------------------------------------------------------------------

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

| Campo                            | Valor                  | Ação / Observação                                                                                                                                            |
| -------------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Desabilitado**                 | **❌ Desmarcado**       | A regra deve estar ativa.                                                                                                                                    |
| **Não faça NAT**                 | **✅ Marcado**          | **CRUCIAL.** Isso instrui o pfSense a não tocar no IP de origem.                                                                                             |
| **Interface**                    | **WAN**                | A interface de saída para a rede externa (Fortinet).                                                                                                         |
| **Família de Endereço**          | **IPv4**               | Suas redes são IPv4.                                                                                                                                         |
| **Protocolo**                    | **any**                | Para cobrir todo o tráfego.                                                                                                                                  |
| **Fonte (Source) / Tipo**        | **Rede**               | Para mapear sub-redes inteiras.                                                                                                                              |
| **Fonte (Source) / Rede**        | **ex.: `10.4.0.0/23`** | Repita o processo para cada sub-rede interna (`10.3.0.0/24`, `10.5.0.0/24`, etc.) OU crie um _Alias_ (apelido) que contenha todas elas e use o _Alias_ aqui. |
| **Destino (Destination) / Tipo** | **any**                | Permite acesso a qualquer destino.                                                                                                                           |
| **Destino (Destination) / Rede** | **`0.0.0.0/0`**        | **USE ESTE VALOR!** Como vimos, ele representa "qualquer rede IPv4" e **resolve o erro de não salvar** o campo vazio ou com "any" literal.                   |
| **Tradução / Endereço**          | **None**               | Confirma a ação de "Não fazer NAT".                                                                                                                          |

### 3. Ordem da Regra (Muito Importante)

No modo Manual, o pfSense processa as regras de NAT de Saída **de cima para baixo**.

* A regra de **"No NAT"** que você está criando deve estar **ACIMA** das regras padrões que fazem a tradução (`Mapear para endereço de interface`).

* **Arraste sua nova regra** de `10.4.0.0/23` com a opção "Não faça NAT" marcada para o **topo da lista** de regras de _Outbound NAT_.

Ao fazer isso, o tráfego das suas redes internas **corresponderá à primeira regra**, será excluído do NAT de Saída e os IPs reais (10.x.x.x) serão enviados ao Fortinet. Todo o restante do tráfego seguirá pelas regras padrão abaixo e continuará sendo NATeado normalmente.

Muito obrigado por enviar as imagens! Elas confirmam que você está no **pfSense** e mostram exatamente onde o problema acontece.

Como você está no pfSense, a solução para o campo **Destino** não aceitar o valor **"Qualquer"** é simples, mas requer uma pequena mudança de tipo, forçando-o a aceitar a sintaxe de rede.

O erro acontece porque, quando você define o **Tipo** de Destino como **"Qualquer"**, o pfSense não sabe qual valor usar no campo de Rede/Subnet (que está logo à direita).

* * *

Solução Rápida: Altere o Tipo para "Network or Alias"
-----------------------------------------------------

Para resolver isso, você deve mudar o **Tipo de Destino** para **"Network or Alias"** e, em seguida, usar a notação universal de rede que significa **"Qualquer"**:

1. No campo **Destino**, mude o **Tipo** de **"Qualquer"** para **"Network or Alias"**.

2. Ao mudar o tipo, o pfSense ativará o campo de texto/dropdown de rede à direita.

3. Nesse novo campo de rede, digite: **`0.0.0.0/0`**

### Por que `0.0.0.0/0` funciona?

* O endereço **`0.0.0.0/0`** é a notação CIDR que significa **"toda a rede IPv4"**. É o equivalente técnico de "Qualquer".

* Ao mudar o **Tipo** para "Network or Alias", você força o pfSense a esperar e aceitar uma notação de rede válida (`0.0.0.0/0`), o que resolverá o erro ao tentar salvar.

### Configuração Final Sugerida

| Campo                   | Configuração                  | Observações                                    |
| ----------------------- | ----------------------------- | ---------------------------------------------- |
| **Não faça NAT**        | **✅ Habilitado**              | Perfeito, é isso que você precisa.             |
| **Fonte / Tipo**        | Network or Alias              | Correto.                                       |
| **Fonte / Rede**        | **`10.4.0.0/23`**             | Correto.                                       |
| **Destino / Tipo**      | **Network or Alias**          | **MUDE PARA ESTE!**                            |
| **Destino / Rede**      | **`0.0.0.0/0`**               | **USE ESTE VALOR!** (Funciona como "Qualquer") |
| **Tradução / Endereço** | **None (Ou deixe em branco)** | O **"Não faça NAT"** já cuida disso.           |

Faça essa alteração no campo **Destino** e tente salvar novamente. A regra deve ser aceita e, o mais importante, deve ser posicionada **no topo da lista de regras de NAT de Saída** (Outbound NAT) para que seja processada antes das regras padrões que fazem a tradução.



---

Ótimo que você enviou a tela! Isso torna o diagnóstico muito mais preciso.

Você está no caminho certo! Usar **`0.0.0.0/0`** é a notação correta. O erro **"Um destino válido deve ser especificado"** está sendo causado pela forma como o **pfSense interpreta o campo de _máscara de sub-rede_** (o `/24` no lado direito) quando você digita `0.0.0.0/0`.

Quando você digita um endereço de rede completo (como `10.4.0.0/23`) nos dois campos, o campo de máscara (`/23` ou `/24`) é preenchido automaticamente. No entanto, o `0.0.0.0/0` é especial.

### A Solução

O pfSense está esperando que você **digite o valor `0`** no campo de máscara de sub-rede.

1. No campo **Destino / Tipo** você já escolheu **`Network or Alias`** e digitou **`0.0.0.0`** no campo de rede.

2. Agora, no campo à direita que está como **`/24`**, **apague o `24` e digite `0`**.

| Campo                 | Configuração  | Ação Necessária      |
| --------------------- | ------------- | -------------------- |
| **Destino / Rede**    | **`0.0.0.0`** | Mantenha este valor. |
| **Destino / Máscara** | **`/ 0`**     | **MUDE PARA ZERO.**  |

Ao especificar explicitamente **`/0`**, você informa ao pfSense que a máscara para `0.0.0.0` é zero, o que representa o bloco completo de endereços IPv4 ("Qualquer").

* * *

### Resumo para Correção:

Corrija o Destino da seguinte forma:

1. **Destino (Rede):** `0.0.0.0`

2. **Destino (Máscara):** `0`

Tente salvar a regra novamente. Deve funcionar! Lembre-se, esta regra precisa estar no **topo da lista** na aba **Outbound NAT**.

---

Solução Encontrada e Próximos Passos
------------------------------------

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

---

Essa é uma excelente observação e um procedimento de segurança muito inteligente! Você está certo em querer desativar as regras antigas primeiro para garantir que a transição seja limpa.

* * *

1. O Motivo da Mão Vermelha

---------------------------

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

--------------------------------------------------------

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

---

Informações:



Você está no caminho certo e essa é uma pergunta **fundamental** para garantir que tudo funcione como planejado!

Sim, sua observação sobre a mudança do **Endereço NAT** está perfeitamente correta.

* * *

O Significado da Mudança no Endereço NAT
----------------------------------------

| Regra                      | Endereço NAT    | Significado Funcional                                                                                                                                      |
| -------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Regras Atuais** (padrão) | **WAN address** | O pfSense está traduzindo o endereço de origem (ex: `10.4.0.33`) para o **IP público da sua WAN** (`200.196.184.130`). _O Fortinet só vê esse IP público._ |
| **Suas Novas Regras**      | **NO NAT**      | O pfSense está instruído a **NÃO** traduzir o endereço de origem. _O Fortinet verá o IP real do usuário_ (ex: `10.4.0.33`).                                |

**Portanto, a resposta é: Sim, o Endereço NAT _tem_ que ser "NO NAT" para suas novas regras.**

Este é o objetivo principal: desabilitar o SNAT (Source NAT) para que o Fortinet receba o tráfego com o IP de origem real do usuário (`10.x.x.x`) e possa exigir a autenticação individual.

* * *

Próximos Passos Críticos (A Mão Vermelha)
-----------------------------------------

Olhando sua imagem mais recente, você já:

1. Criou as regras de "No NAT" (elas estão no topo).

2. Manteve as regras antigas desativadas (a mãozinha vermelha).

O que falta para as regras entrarem em vigor é exatamente o que o aviso em verde no topo da tela diz:

1. **Mudar o Modo de NAT:** O modo atual ainda é **"Automática geração de regra NAT de saída"** (o primeiro botão está selecionado). Você precisa selecionar o botão **"Geração de regra NAT de saída Manual (AON)"** (o terceiro botão).

2. **Aplicar Mudanças:** Depois de mudar para o modo Manual, clique em **Salvar** e, em seguida, no botão verde **"Aplicar Mudanças"** no canto superior direito.

Ao aplicar as mudanças no modo **Manual**, suas 4 regras de "NO NAT" (que já estão no topo da lista) serão ativadas, a mãozinha vermelha sumirá, e o pfSense começará a enviar o tráfego com os IPs reais para o Fortinet.

---

---

Essa é uma ótima pergunta, e é crucial para entender como o pfSense está funcionando neste momento!
---

* * *

O Significado dos Ícones no pfSense
-----------------------------------

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
-----------------------------------

Para que suas regras de **"NO NAT"** entrem em vigor, você precisa sair do modo Automático e ir para o modo Manual, que força o sistema a usar as regras de "Mapeamentos" que você criou:

1. Selecione a opção **"Geração de regra NAT de saída Manual (AON - Avançado)"**.

2. Clique no botão **Salvar** logo abaixo do Modo NAT de Saída.

3. Clique no botão verde **"Aplicar Mudanças"** no canto superior direito.

Ao aplicar as mudanças no modo Manual, você verá o seguinte:

* A seção **"Automatic Rules"** **desaparecerá** (pois o pfSense não gera mais regras automáticas).

* As regras em **"Mapeamentos"** serão ativadas (a mãozinha vermelha sumirá e será substituída por um _check_ ou um ícone verde).

Neste ponto, suas 4 regras de **"NO NAT"** estarão no topo e em vigor, e o Fortinet passará a ver os IPs internos dos seus usuários.

---

Perfeito, agora entendi 100% seu cenário 👌

📌 O desenho é este:
    [Internet] ⇄ [Fortinet - autenticação individual] ⇄ [pfSense] ⇄ [Rede interna]

Ou seja:

* **Fortinet** é quem fala direto com a internet e controla **login/senha por usuário**.

* **pfSense** não pode “esconder” (NAT) os IPs internos, senão o Fortinet só veria um único IP (o do pfSense) e **não conseguiria aplicar as regras de login individual**.

* Portanto:
  
  * Para os **servidores/usuários que devem ser autenticados no Fortinet**, você precisa de **NO NAT** no pfSense.
  
  * Para qualquer outra rede que precise sair sem autenticação (ou se o Fortinet estiver configurado só como gateway), aí sim você pode usar NAT normal.

* * *

### ✅ O que deve ficar no pfSense

1. **Manter as regras de NO NAT** para as redes que precisam aparecer “com seus IPs reais” no Fortinet:
   
   * 10.0.0.0/24
   
   * 10.3.0.0/24
   
   * 10.4.0.0/23
   
   * 10.5.0.0/24  
     (essas já estão corretas na sua tela!)

2. **Criar apenas UMA regra de NAT “normal” (mascaramento)** para casos em que:
   
   * Alguma rede precisa sair à internet sem passar pelo controle do Fortinet (ex.: um servidor do pfSense, atualizações, logs, ou outra subrede).
   
   * Nesse caso, configure NAT = Interface Address (WAN).
   
   * Mas cuidado: se aplicar a todas as redes, o Fortinet vai perder visibilidade dos IPs e o login individual não vai funcionar.

* * *

📌 Resumindo:

* Se **todos os usuários precisam ser autenticados no Fortinet**, então o pfSense deve ficar só com **NO NAT** (do jeito que já está) — ou seja, ele só roteia.

* Se **alguns hosts precisam sair sem autenticação**, aí sim você cria uma regra NAT “normal” só para essa rede/host.

* * *

👉 Pergunta prática para ajustar a resposta:  
Você precisa que **todos os usuários internos façam login no Fortinet** (rede inteira controlada), ou só **algumas redes/usuários**?



---

Show, agora clareou totalmente o cenário 👍

➡️ Você tem:

* Redes **10.4.0.x / 10.4.1.x / 10.5.0.x** → desktops e Wi-Fi.

* Cada desktop já tem **MAC cadastrado** no servidor (provavelmente via DHCP estático no Windows Server ou no Fortinet).

* Quando o usuário conecta, o Fortinet usa **IP fixo + login** para controlar o acesso.

* * *

### 🔎 Sobre o impacto no mapeamento por MAC

Não, **não vai afetar** o mapeamento que já existe, porque:

1. **O Fortinet precisa ver o IP real** de cada máquina (derivado do MAC).
   
   * E com o **NO NAT no pfSense**, isso continua funcionando, já que o pfSense apenas roteia o tráfego e **não troca o IP de origem**.

2. O servidor 10.4.0.1 continua entregando sempre o mesmo IP para o mesmo MAC.
   
   * O Fortinet vai receber exatamente esse IP, do jeito que já funciona hoje.

3. A autenticação do Fortinet também não muda.
   
   * O login individual continua aparecendo quando o usuário abre o navegador, só que agora **cada máquina terá que logar**, porque o Fortinet verá cada IP separadamente (e não tudo vindo do IP do pfSense).

* * *

✅ Portanto:

* **O cadastro de MAC → IP fixo não será impactado.**

* **O Fortinet continuará controlando login individual**, já que você deixou as redes internas em **NO NAT** no pfSense.

* O pfSense passa a ser apenas o “roteador interno”, sem esconder os IPs.

* * *

👉 Agora uma dúvida final para garantir que nada quebre:  
Seu pfSense está como **gateway padrão** dessas redes (10.4.x, 10.5.x), certo? Ou essas redes já saem direto no Fortinet e o pfSense só é usado como firewall extra?

---

Boa, então ficou claro:

📌 **Situação atual**

* Todos os desktops, servidores, impressoras etc. têm como **gateway padrão o pfSense (10.0.0.1)**.

* O pfSense, por sua vez, manda tudo para o **Fortinet**, que valida login individual antes de liberar a internet.

* Isso confirma que o **pfSense está roteando tudo**, e por isso o ajuste de NAT era realmente necessário.

* * *

### ✅ Impacto na configuração que você fez

* Como você aplicou **NO NAT** nas redes internas (10.4.0.0/23, 10.5.0.0/24, 10.3.0.0/24 etc.), o pfSense não troca mais os IPs de origem.

* Ou seja, o **Fortinet agora enxerga os IPs reais das máquinas** (que já estão amarrados ao MAC via DHCP).

* Assim, o Fortinet consegue aplicar a regra de login individual sem que tudo apareça como “um IP só” do pfSense.

👉 Isso significa que o **cadastro por MAC e IP não será alterado**.  
👉 O **login por usuário no Fortinet funcionará** normalmente, mas cada estação precisará logar com seu usuário individual.

* * *

### ⚠️ Só um ponto de atenção:

O que pode estar te dando a **mão vermelha** na LAN/WAN não é o NAT, mas:

1. **Rota de saída no pfSense**: a WAN do pfSense (200.196.184.130) talvez esteja configurada para sair direto na internet, em vez de sair pelo Fortinet.
   
   * O certo é a WAN do pfSense ter como **gateway o Fortinet**, e não a operadora.

2. **Firewall rules** na aba LAN: precisam permitir tráfego da rede interna para “any” (ou pelo menos até o Fortinet).

* * *

👉 Pergunto para confirmar:  
O IP **200.196.184.130** que aparece na WAN do pfSense, é um IP público da operadora **direto no pfSense**, ou é o IP interno que o Fortinet entrega para o pfSense?

Isso vai mudar totalmente como devemos ajustar o **gateway** no pfSense.

---

Perfeito — vamos organizar tudo para você validar e deixar funcionando com segurança. Vou explicar o que **exatamente** precisa estar configurado (e onde checar) para que o fluxo

**[desktops (10.4.x / 10.5.x)] → pfSense (gateway: 10.0.0.1) → Fortinet → Internet (200.196.x.x)**

funcione com **NO NAT** (para que o Fortinet veja os IPs reais e peça login individual), e sem quebrar DHCP por MAC ou o acesso ao AD.

* * *

Resumo da ideia (em 1 frase)
----------------------------

Se o pfSense não faz NAT, **o Fortinet precisa saber roteamento de volta** para as redes 10.x; o pfSense precisa ter **gateway apontando para o Fortinet**. Ambos os lados (pfSense e Fortinet) devem ter rotas coerentes e regras de firewall liberando o tráfego.

* * *

O que verificar / ajustar (passo-a-passo)
-----------------------------------------

### 1) Confirme qual é o _next-hop_ entre pfSense e Fortinet

* No pfSense acesse: **Status > Interfaces** e veja o IP da **WAN** (ex.: `200.196.184.130`) e o **gateway** atribuído.

* Em **System > Routing > Gateways** confirme qual é o _default gateway_ da WAN.
  
  * **O gateway da WAN deve ser o IP do Fortinet** (o equipamento que fica “antes” do pfSense).
  
  * Se hoje o gateway está configurado direto para o roteador do provedor, edite-o para usar o Fortinet (IP no link WAN).

> Se você não souber qual IP usar como gateway, peça ao responsável do Fortinet/operadora o IP do próximo salto (o Fortinet ou roteador) para a sub-rede pública.

* * *

### 2) Rotas estáticas no Fortinet (obrigatórias se NO NAT)

Como o pfSense vai **não NATear** as redes internas, o Fortinet precisa **saber como devolver o tráfego** para as sub-redes 10.x:

No FortiGate (exemplo de preenchimento):

* Destination: `10.4.0.0/23` → Gateway/Next Hop: `200.196.184.130` (o IP WAN do pfSense)

* Destination: `10.5.0.0/24` → Gateway/Next Hop: `200.196.184.130`

* Destination: `10.3.0.0/24` → Gateway/Next Hop: `200.196.184.130`

* Destination: `10.0.0.0/24` → Gateway/Next Hop: `200.196.184.130`

> Em suma: para cada rede interna que você colocou em **NO NAT**, crie uma rota no Fortinet apontando para o IP WAN do pfSense.

* * *

### 3) Regras NAT no pfSense (revisão)

Você configurou regras **NO NAT** para as redes internas — ok. Mas mantenha **uma regra catch-all** (se precisar de saída sem controle) somente para hosts/ redes que você queira mascarar.  
No seu cenário, para **todos os desktops Wi-Fi e cabeados** você DEVE deixar **No NAT** (conforme já fez).

* * *

### 4) Firewall rules no pfSense

No pfSense: **Firewall > Rules > LAN**:

* Garanta uma regra que permita:
  
  * Action: **Pass**
  
  * Interface: **LAN**
  
  * Protocol: **Any**
  
  * Source: **LAN net** (ou as subnets específicas)
  
  * Destination: **any**

Sem isso, mesmo com rotas e NAT corretos, não passará tráfego.

* * *

### 5) Firewall / políticas no Fortinet

* No FortiGate, a política que aplica o captive portal deve permitir tráfego **originado** das subnets 10.x para Internet e possuir autenticação de usuário habilitada.

* Se tiver política por IP / grupo, ajuste para incluir as subnets.

* * *

### 6) DHCP estático por MAC (servidor 10.4.0.1) — sem impacto

* Seu DHCP estático (bind MAC → IP) continua funcionando normalmente.

* Com **NO NAT**, o IP atribuído pelo DHCP é justamente o IP que o Fortinet verá — portanto **não é afetado**.

* Não precisa mudar os cadastros MAC→IP.

* * *

Testes e diagnóstico (faça estes passos para validar)
-----------------------------------------------------

1. **Do pfSense (Diagnostics > Ping)**:
   
   * Ping `8.8.8.8` — verifica se o pfSense alcança a internet.
   
   * Ping o Gateway do Fortinet (next-hop) — verifica link entre pfSense e Fortinet.

2. **Do cliente (desktop)**:
   
   * `ipconfig /all` — confirme gateway = `10.4.0.1` e DNS.
   
   * `tracert 8.8.8.8` — o primeiro salto deve ser `10.4.0.1`, em seguida deve ir para o pfSense/WAN e depois Fortinet.

3. **Do Fortinet**:
   
   * Teste rota: ping para `10.4.0.33` (um host interno). Deve responder (se ICMP liberado).
   
   * Verifique tabela de rotas do Fortinet (`get router info routing-table` ou via GUI).

4. **Logs**:
   
   * pfSense: **Status > System Logs > Firewall** para ver se pacotes estão sendo bloqueados.
   
   * Fortinet: logs de autenticação/captive portal para ver se autenticação por IP está ocorrendo.

* * *

Possíveis problemas comuns e soluções rápidas
---------------------------------------------

* **Fortinet não tem rota para 10.x → tráfego chega ao Fortinet, mas as respostas não retornam**  
  → Solução: adicionar rotas estáticas no Fortinet apontando para o IP WAN do pfSense.

* **pfSense gateway incorreto (não aponta para Fortinet)**  
  → Solução: ajustar **System > Routing > Gateways** para usar o IP do Fortinet como gateway.

* **Regras de firewall bloqueando tráfico**  
  → Solução: checar `Firewall > Rules > LAN` e criar regra "Pass" para LAN net → any (temporariamente para teste).

* **DNS / AD não resolvem**  
  → Certifique-se de que seus clientes usam os DNS internos do AD (10.4.0.9 etc.) e que pfSense não está forçando DNS externo via NAT/DNS forward.

* * *

Exemplo prático (o que você deve fazer agora — checklist rápido)
----------------------------------------------------------------

1. No pfSense: confirme WAN IP (200.196.184.130) e gateway. Se gateway ≠ Fortinet IP, altere para o Fortinet.

2. No Fortinet: adicione rotas estáticas para as redes internas apontando para `200.196.184.130` (IP WAN do pfSense).

3. No pfSense: mantenha suas regras **NO NAT** para 10.4.0.0/23, 10.5.0.0/24, etc.

4. No pfSense: confira **Firewall > Rules > LAN** e garanta regra Pass.

5. Teste a partir de um desktop: abra navegador, verifique se o captive portal do Fortinet aparece; faça login; verifique navegação.

6. Se não navegar, faça ping/trace a partir do pfSense para internet e a partir do Fortinet para um host 10.x.

* * *

Se você quiser eu **posso gerar para você**:

* os comandos/valores exatos a colocar no Fortinet (ex.: como ficaria a rota estática, com campos nome/destino/gateway), **ou**

* um roteiro clique-a-clique no pfSense para checar gateway, regras LAN e testes de ping.

Qual prefere agora? Quer que eu escreva as rotas estáticas prontas para você colar no Fortinet (me confirme qual é o IP do próximo-salto do Fortinet — pode ser .129 ou .128 como você disse) ou prefere o passo-a-passo no pfSense primeiro?



---

Este é um ótimo cenário, e a documentação que você já tem é bastante detalhada! Vou usá-la como base para criar um **guia prático e detalhado**, focando nos pontos de configuração do seu **pfSense** com base nas imagens que você enviou.

O objetivo é garantir que o **pfSense** (gateway 10.0.0.1) envie o tráfego da LAN diretamente para o **Fortinet** (_next-hop_), e que o Fortinet possa devolver esse tráfego (já que o pfSense não fará **NAT**).

* * *

Análise das Configurações Atuais do pfSense
-------------------------------------------

Antes de tudo, vamos analisar as informações das suas imagens para ter certeza dos IPs e interfaces:

### Status / Interfaces

* **Interface LAN (bce1):**
  
  * **Endereço IP:** 10.0.0.1 (Este é o seu gateway de LAN, correto, e o que os clientes 10.4.x e 10.5.x devem usar).

* **Interface WAN (bce0):**
  
  * **Endereço IP:** 200.196.184.130
  
  * **Gateway IPv4:** 200.196.184.129

### Gateways

* **WANGW (Padrão):**
  
  * **Interface:** WAN
  
  * **Gateway:** 200.196.184.129 (O _next-hop_ para a Internet).

* **REDE_DHCP:**
  
  * **Interface:** LAN
  
  * **Gateway:** 10.0.0.2 (Provavelmente um gateway interno ou um IP não usado para o tráfego de Internet). **Este não deve ser o _default gateway_ para a Internet.**

### Regras / LAN (Firewall)

* A regra 4a ("Default allow LAN to any rule") é um **Pass** para **IPv4** de **LAN subnets** para *** (any) destination** na **porta * (any)**. **Esta regra é essencial e está OK.** Ela permite que o tráfego da LAN (10.x.x.x) saia do pfSense.

* * *

# Modo de Configuração 02:

Passo-a-Passo para a Configuração sem NAT
-----------------------------------------

Com base na sua análise e nas imagens, o maior ponto de atenção é garantir que o **Gateway WANGW** (200.196.184.129) seja o IP do seu **Fortinet**.

### 1. Confirmar o Next-Hop (Gateway) da WAN

**Objetivo:** Garantir que todo o tráfego que sai do pfSense pela interface WAN (200.196.184.130) seja enviado para o **Fortinet**. O IP do Fortinet deve ser o gateway 200.196.184.129.

1. **Acesse:** **Sistema** → **Roteamento** → **Gateways**.

2. Confirme se o **Gateway** de nome **WANGW** (200.196.184.129) é o IP da **interface do Fortinet** conectada ao pfSense.
   
   * **Se for:** Ótimo, o pfSense está enviando o tráfego para o Fortinet. Não precisa fazer nada aqui.
   
   * **Se não for:** Você precisa **Editar** o gateway **WANGW** ou **Criar** um novo para apontar para o IP correto do Fortinet e torná-lo o _default gateway_ IPv4. _Pelos seus logs, o 200.196.184.129 é o gateway padrão, então vamos assumir que este é o Fortinet._

### 2. Rotas Estáticas no Fortinet (Obrigatoriedade do NO NAT)

**Objetivo:** Como o pfSense não está fazendo NAT, o Fortinet precisa saber que para _responder_ ao tráfego vindo das redes 10.4.x.x ou 10.5.x.x, ele deve enviar a resposta de volta para o **IP WAN do pfSense (200.196.184.130)**.

**Você deve configurar as seguintes rotas estáticas no seu Fortinet:**

| Destino (Rede Interna) | Máscara                | Gateway (Next Hop)                      | Interface                        | Descrição                    |
| ---------------------- | ---------------------- | --------------------------------------- | -------------------------------- | ---------------------------- |
| **10.4.0.0**           | 255.255.254.0 (ou /23) | **200.196.184.130** (IP WAN do pfSense) | [Interface conectada ao pfSense] | Rota para LAN 10.4/23        |
| **10.5.0.0**           | 255.255.255.0 (ou /24) | **200.196.184.130** (IP WAN do pfSense) | [Interface conectada ao pfSense] | Rota para LAN 10.5/24        |
| **10.0.0.0**           | 255.255.255.0 (ou /24) | **200.196.184.130** (IP WAN do pfSense) | [Interface conectada ao pfSense] | Rota para Rede pfSense (LAN) |

_**OBS:** Peça ao responsável pelo Fortinet para configurar estas rotas. Sem elas, o tráfego de saída funcionará, mas o tráfego de resposta da Internet morrerá no Fortinet, pois ele não saberá como chegar nas redes 10.x.x.x._

### 3. Regras de NAT (Mantenha o NO NAT)

**Objetivo:** Garantir que o pfSense _não_ mascare os IPs 10.x.x.x para que o Fortinet veja o IP real para aplicar o Captive Portal.

* A regra de NO NAT deve ser configurada em **Firewall** → **NAT** → **Outbound** (Regras de Saída).

* Você mencionou que já configurou regras **NO NAT** para as redes internas (10.4.x/10.5.x).

* **Verifique se a regra _Automática_ (MASCARAR TUDO) está desativada ou se as suas regras NO NAT estão acima dela, com um "Stop processing rules" marcado (para garantir que não haja NAT).**

### 4. Regras de Firewall (LAN)

**Objetivo:** Confirmar que o tráfego da LAN está livre para sair.

1. **Acesse:** **Firewall** → **Regras** → **LAN**.

2. Confirme a existência e o _status_ (ativo/verde) da regra que permite o tráfego de saída:
   
   * **Regra em questão (4ª linha da sua imagem):**
     
     * **Ação:** Pass (Verde, ✓4.508K/4.18GiB)
     
     * **Protocolo:** IPv4 *
     
     * **Origem:** LAN subnets
     
     * **Destino:** * (any)
     
     * **Descrição:** Default allow LAN to any rule
   
   * **Status:** A regra está **ativa** e **funcionando**. **Nenhuma alteração é necessária aqui.**

* * *

Testes e Diagnóstico (Checklist de Validação)
---------------------------------------------

Depois de garantir que as **Rotas Estáticas** foram configuradas no **Fortinet** (Passo 2), faça os seguintes testes:

### No pfSense (Diagnóstico → Ping)

1. **Ping o Gateway do Fortinet (Next-Hop):**
   
   * **Ping:** 200.196.184.129
   
   * **Resultado esperado:** Sucesso. (Verifica a conectividade direta entre pfSense e Fortinet).

2. **Ping a Internet:**
   
   * **Ping:** 8.8.8.8
   
   * **Resultado esperado:** Sucesso. (Verifica se o pfSense consegue sair para a Internet via Fortinet).

### No Cliente (Desktop 10.4.x ou 10.5.x)

1. **Confirme o Gateway e DNS:**
   
   * Abra o terminal/prompt e use `ipconfig /all` ou `ip a`.
   
   * **Verifique:** O Gateway deve ser **10.0.0.1** e o DNS deve ser o seu **servidor AD (10.4.0.9 etc.)**.

2. **Teste o Roteamento:**
   
   * **`tracert 8.8.8.8`** (ou `traceroute 8.8.8.8` em Linux/macOS)
   
   * **Primeiro Salto:** 10.0.0.1 (pfSense LAN)
   
   * **Segundo Salto:** 200.196.184.130 (pfSense WAN)
   
   * **Terceiro Salto:** 200.196.184.129 (Fortinet)
   
   * **Resultado esperado:** Se o trace passar pelo Fortinet, o roteamento **NO NAT** está funcionando. Você deve ser direcionado para o Captive Portal do Fortinet no navegador.

### No Fortinet

1. **Teste a Rota de Retorno:**
   
   * Ping para um host interno (ex.: 10.4.0.33).
   
   * **Resultado esperado:** Sucesso. (Confirma que as rotas estáticas que você adicionou estão funcionando).

Este roteiro detalhado, com foco na correção da rota no Fortinet e a validação do gateway no pfSense, deve garantir que seu ambiente funcione com segurança e visibilidade total dos IPs de origem.

Ficou claro qual é a sua próxima ação, que é **garantir as rotas estáticas no Fortinet**, já que o pfSense está bem configurado para a saída?

---

Compreendo perfeitamente. Como você não tem acesso ao Fortinet, vamos focar no que é 100% controlável e essencial no seu pfSense para que o tráfego saia **sem NAT**.

O ponto mais crucial para que o Fortinet veja os IPs reais (10.4.x.x e 10.5.x.x) é a configuração do **NAT de Saída (Outbound NAT)**.

A sua configuração de Gateway e Regras LAN já foi validada e está correta para o fluxo. Agora, o passo final é garantir o **NO NAT**.

* * *

1. Configurar o NAT de Saída (Outbound NAT)

-------------------------------------------

Por padrão, o pfSense faz NAT (mascaramento) em todo o tráfego que sai pela WAN, trocando o IP de origem 10.x.x.x pelo IP 200.196.184.130. Precisamos desativar isso para as suas redes internas.

### A. Mudar para o Modo de Geração Manual de Regras

1. **Acesse:** **Firewall** → **NAT** → **Outbound**.

2. No menu suspenso "Mode" (Modo), selecione a opção:
   Manual Outbound NAT rule generation (Avanc¸​ado)

3. Clique em **Salvar**.

Ao salvar, o pfSense deve gerar automaticamente as regras padrão. Agora, você pode adicionar a sua regra de **NO NAT** no topo.

### B. Adicionar a Regra de NO NAT (Mapeamento Estático)

Vamos adicionar uma regra para **NÃO** fazer NAT para as suas redes internas. Esta regra deve ser a **primeira da lista** ou estar acima de qualquer regra que faça NAT.

1. Clique no botão **Adicionar** na aba **Mapeamentos**.

2. Preencha (ou verifique) os seguintes campos:
   
   * **Disable (Desabilitar):** _Deixar desmarcado._
   
   * **Interface:** **WAN** (O tráfego sai por aqui).
   
   * **Protocol:** **any**
   
   * **Source (Origem):**
     
     * **Network (Rede):** Insira a sua rede interna principal: `10.4.0.0/23`
     
     * _Opcional: Se tiver outras redes, crie regras separadas, como 10.5.0.0/24._
   
   * **Destination (Destino):**
     
     * **Network (Rede):** `any` (Qualquer destino na Internet).
   
   * **Translation (Tradução/Alvo):**
     
     * **Type:** **None** (Este é o crucial **NO NAT**).

3. Clique em **Salvar**.

4. Clique em **Apply Changes** (Aplicar Alterações) no topo da página.

Resultado Esperado:

O tráfego das redes 10.4.0.0/23 e 10.5.0.0/24 sairá do pfSense com o IP de origem original.

* * *

2. Teste Final de Roteamento (Cliente)

--------------------------------------

Com a regra **NO NAT** aplicada, o pfSense está configurado corretamente. O teste de rastreamento de rota é a única maneira de confirmar se o Fortinet está recebendo o IP de origem correto.

Execute este teste a partir de um desktop (10.4.x.x ou 10.5.x.x):

| Comando           | Resultado Esperado                           | Observação                                         |
| ----------------- | -------------------------------------------- | -------------------------------------------------- |
| `tracert 8.8.8.8` | 1o salto: 10.0.0.1 (pfSense LAN)             | Confirma que o gateway do cliente está correto.    |
|                   | 2o salto: 200.196.184.129 (Fortinet Gateway) | Confirma que o tráfego vai direto para o Fortinet. |
|                   | 3o salto: IP do Provedor                     | O Fortinet está encaminhando o tráfego.            |

### Validação do Captive Portal

Após o `tracert` ser bem-sucedido, tente acessar qualquer site no navegador do desktop:

1. O navegador deve exibir a **página de login (Captive Portal)** do Fortinet.

2. Se a página de login aparecer, isso significa que:
   
   * O tráfego saiu do pfSense com o IP **10.x.x.x (NO NAT)**.
   
   * O Fortinet recebeu esse IP e, como não o conhece, acionou a autenticação.

Atenção - O Único Ponto de Falha Restante
-----------------------------------------

Como você não tem acesso ao Fortinet, o **único ponto de falha** restante é o **roteamento de retorno**.

Se o login for bem-sucedido no Fortinet, mas você não conseguir navegar (o tráfego de resposta não volta), a causa é quase certamente a **falta das rotas estáticas** no Fortinet, conforme detalhei antes:

* **Sintoma:** Você consegue fazer o `tracert` e ver a página de login, mas a navegação é lenta ou não funciona.

* **Causa:** O Fortinet sabe enviar o tráfego para a Internet, mas quando a Internet responde, o Fortinet não sabe que para chegar em 10.4.x.x, ele precisa enviar o pacote de volta para o IP WAN do pfSense (200.196.184.130).

* **Solução:** É necessário que o responsável pelo Fortinet adicione as rotas estáticas:
  
  > "Para redes 10.4.0.0/23 e 10.5.0.0/24, use o gateway 200.196.184.130."

Foque no Passo 1 (NAT Outbound). Ele é o suficiente no pfSense para que o Fortinet receba o tráfego com o IP de origem correto.

---

Perfeito — vou te passar **um roteiro passo-a-passo, direto e aplicável** (pfSense + FortiGate), usando os IPs que você mostrou. Siga cada etapa com calma e, se possível, aplique em horário de baixa produção (ou em janela de manutenção).

* * *

Resumo do objetivo
------------------

Fazer com que o **Fortinet** enxergue os **IPs reais** dos clientes (10.x.x.x) para forçar **login individual** no captive portal, mantendo acesso interno (AD/DHCP) e internet funcionando. Para isso:

* o **pfSense** deve **não NATear** as sub-redes de usuários (NO NAT);

* o **Fortinet** precisa ter **rotas estáticas** apontando para o **IP WAN do pfSense (200.196.184.130)** para devolver o tráfego às redes 10.x;

* o **pfSense** deve usar o **Fortinet** como gateway de saída.

* * *

Passo a passo — pfSense
=======================

### 1) Conferir gateway WAN (pfSense → Fortinet)

1. `System > Routing > Gateways`

2. Verifique que o gateway `WANGW` está com **Gateway = 200.196.184.129** (na sua print está assim).

3. Em **Gateway padrão IPv4** selecione `WANGW` (ou deixe em _Automatic_ se já apontar corretamente).

4. Salve.

> Observação: o gateway da WAN deve ser o IP do Fortinet/next-hop (no seu caso `200.196.184.129`). Se já está assim, ok.

* * *

### 2) Outbound NAT — deixar NO NAT para as redes de usuários

1. `Firewall > NAT > Outbound`

2. Seleccione **Manual Outbound NAT** (ou **Hybrid** se quiser manter regras automáticas para VPNs).

3. Para cada faixa de rede de usuários (faça por rede, não por host) adicione uma regra **NO NAT**:

Exemplo (adicionar regra):

* **Interface:** `WAN`

* **Source / Tipo:** `Network` (ou `Network or Alias`)

* **Source / Rede:** `10.4.0.0/23` _(isso cobre 10.4.0.xxx e 10.4.1.xxx)_

* **Destination:** `any`

* **Translation / Address:** `None` (No NAT)

* **Descrição:** `No NAT 10.4.0.0/23 para Fortinet`

Repita para:

* `10.5.0.0/24`

* `10.3.0.0/24`

* `10.0.0.0/24` (ou as máscaras reais que você usa)
4. Salve e **Apply Changes**.

> IMPORTANTE: mantenha apenas **NO NAT** para as redes que devem autenticar no Fortinet. Se você depender de NAT para alguma rede específica (por exemplo um servidor que precisa sair com IP público), crie regra específica para isso (mas não para as redes de desktops/wi-fi).

* * *

### 3) Verificar regras de firewall (LAN)

1. `Firewall > Rules > LAN`

2. Garanta que exista uma regra **Pass** permitindo `LAN subnets → any` (ou regras equivalentes que permitam tráfego para internet).
   
   * Action: **Pass**
   
   * Interface: **LAN**
   
   * Protocol: **any**
   
   * Source: **LAN net** (ou os subnets)
   
   * Destination: **any**

3. Salve e aplique.

* * *

### 4) Reiniciar estados (limpar estados antigos)

1. `Diagnostics > States > Reset States` — clique para reiniciar estados do PF.
   
   * Isso evita que sessões antigas NATeadas continuem bloqueando tráfego novo.

* * *

### 5) Testes iniciais no pfSense

* Em `Diagnostics > Ping` do pfSense:
  
  * Ping `200.196.184.129` (gateway Fortinet) — deve responder.
  
  * Ping `8.8.8.8` — deve responder (se o Fortinet estiver permitindo tráfego).  
    Se o pfSense não consegue pingar o gateway, corrija gateway/ligação física antes de prosseguir.

* * *

Passo a passo — Fortinet (rotas estáticas)
==========================================

> Objetivo: dizer ao FortiGate **como voltar** às redes internas (10.x) — encaminhando para o pfSense WAN (200.196.184.130).

### 1) Adicionar rotas estáticas (GUI genérico)

No FortiGate (GUI):

* Vá em **Network > Static Routes** (ou Router > Static > Static Routes) → **Create New**.

Para cada rede, preencha:

* **Destination**: `10.4.0.0/23`

* **Device / Interface**: interface conectada ao seu link com pfSense (ex.: `portX` ou `wanx`)

* **Gateway**: `200.196.184.130` _(IP WAN do pfSense)_

* **Distance**: `10` (ou default)

Repita para:

* `10.5.0.0/24` → gateway `200.196.184.130`

* `10.3.0.0/24` → gateway `200.196.184.130`

* `10.0.0.0/24` → gateway `200.196.184.130`

Salve.

### 2) Verificar política do captive portal

* A política que exige o login no Fortinet deve aceitar tráfego com **source = subnets 10.x**.

* Normalmente você tem uma policy `LAN (internal) → Internet` onde ativa autenticação. Verifique se essa policy cobre as sub-redes 10.4.x / 10.5.x / 10.3.x.

* * *

Testes práticos e diagnóstico (faça nessa ordem)
================================================

1. **Do pfSense** (Diagnostics > Ping):
   
   * `ping 200.196.184.129` (gateway Fortinet) → OK
   
   * `ping 8.8.8.8` → OK (se Fortinet permitir)

2. **Do desktop** (ex.: 10.4.0.33):
   
   * `ipconfig /all` → confirme gateway = `10.4.0.1` e DNS = 10.4.0.9 (ou seu DNS interno).
   
   * Tente `ping 8.8.8.8` → **antes de autenticar no Fortinet**, deverá falhar.
   
   * Abra navegador → captive portal do Fortinet deve aparecer → faça login com usuário.
   
   * Após autenticar: `ping 8.8.8.8` e `curl http://ifconfig.me` (ou acessar sites) → OK.

3. **Se algo não funcionar**:
   
   * No pfSense: `Status > System Logs > Firewall` e `Diagnostics > States` → ver bloqueios/estados.
   
   * No Fortinet: logs de roteamento e logs do captive portal.
   
   * Teste `traceroute` a partir do desktop para 8.8.8.8:
     
     * 1º salto: 10.4.0.1 (DHCP server/router local)
     
     * 2º salto: pfSense LAN/WAN (deve aparecer)
     
     * etc.

* * *

Pontos de atenção / riscos e como mitigar
=========================================

* **Bloqueio de RFC1918 na WAN do pfSense?**  
  A regra padrão “Bloquear redes privadas” na interface WAN não impede o funcionamento normal do retorno (porque o estado foi criado pelo cliente interno). Normalmente não precisa desativar. Só se o Fortinet enviar tráfego com origem RFC1918 direto ao pfSense e houver políticas específicas, reveja.

* **IP do gateway errado** → se o gateway do pfSense não for o Fortinet, o tráfego não chegará ao Fortinet. Confirme `System > Routing > Gateways`.

* **Se usar IPsec/serviços que dependem de NAT**, mantenha a regra ISAKMP (porta 500) que o pfSense cria automaticamente.

* **Sincronização HA / XMLRPC**: se tiver cluster pfSense, marque `No XMLRPC Sync` conforme necessário.

* * *

Checklist final (faça e confirme cada item)
===========================================

* `System > Routing > Gateways`: gateway WAN = `200.196.184.129` (Fortinet).

* `Firewall > NAT > Outbound`: Manual (ou Hybrid) com regras **NO NAT** para `10.4.0.0/23`, `10.5.0.0/24`, `10.3.0.0/24`, `10.0.0.0/24`.

* `Firewall > Rules > LAN`: existe regra **allow LAN subnets → any**.

* `Diagnostics > States > Reset States` executado.

* FortiGate: rotas estáticas para todas as 10.x apontando para `200.196.184.130` (WAN do pfSense).

* Teste: cliente tenta acessar → captive portal aparece → após login navega normalmente.

* * *

Se quiser eu já **te escrevo as rotas prontas** para colar no FortiGate (GUI + CLI exemplo) e **o JSON/formulário exato** para cada regra NO NAT no pfSense (linha por linha) — me diga se prefere **comandos CLI do FortiGate** ou apenas instruções GUI.  
Também posso montar um mini-script/checklist para você rodar logo após aplicar as mudanças (pings/traceroutes/onde olhar logs). Quer que eu gere isso agora?

---

Boa observação 👌 — no **pfSense**, quando você cria regra de **Outbound NAT (Manual/Hybrid)**, o campo **Destination** não aceita simplesmente `any` como no firewall normal.

👉 O jeito certo é:

* colocar **Destination = Network**

* e informar `0.0.0.0/0` (que equivale a “qualquer destino”).

* * *

### Exemplo de regra correta (NO NAT):

* **Interface:** `WAN`

* **Source:** `10.4.0.0/23`

* **Source Port:** _any_

* **Destination:** `Network`

* **Address:** `0.0.0.0/0`

* **Destination Port:** _any_

* **Translation / Address:** `None (no NAT)`

* **Descrição:** `No NAT 10.4.0.0/23 → qualquer destino`

* * *

### Repita para cada rede:

* `10.5.0.0/24 → 0.0.0.0/0`

* `10.3.0.0/24 → 0.0.0.0/0`

* `10.0.0.0/24 → 0.0.0.0/0`

* * *

🔎 Dica: se preferir mais “enxuto”, pode criar **uma única regra** cobrindo todas suas sub-redes, usando um **Alias** com as redes internas (Firewall > Aliases > Networks).  
Depois na regra Outbound NAT → `Source = [Nome do Alias]` e `Destination = 0.0.0.0/0`.

* * *

Quer que eu já te monte o **alias com as redes** e a **regra única pronta** (fica mais limpo do que várias repetidas)?

---

Show 👍 bora simplificar. Você pode resolver com **1 Alias + 1 regra de Outbound NAT**.

* * *

### Passo 1 → Criar o **Alias**

No pfSense:

1. Vá em **Firewall > Aliases > IP**.

2. Clique em **Add**.

3. Preencha:
   
   * **Name:** `REDE_LOCAL`
   
   * **Type:** `Network(s)`
   
   * **Networks:**
     
     * `10.0.0.0/24`
     
     * `10.3.0.0/24`
     
     * `10.4.0.0/23` (cobre `10.4.0.x` e `10.4.1.x`)
     
     * `10.5.0.0/24`
   
   * **Description:** `Todas as redes internas`

4. Salve e aplique.

* * *

### Passo 2 → Criar a **regra Outbound NAT (No NAT)**

1. Vá em **Firewall > NAT > Outbound**.

2. Mude o modo para **Manual Outbound NAT** (se já não estiver).

3. Clique em **Add** (+).

4. Configure:
   
   * **Interface:** `WAN`
   
   * **Source:** `REDE_LOCAL` (alias que você criou)
   
   * **Source Port:** `any`
   
   * **Destination:** `Network`
   
   * **Address:** `0.0.0.0/0`
   
   * **Destination Port:** `any`
   
   * **Translation / Address:** `None (no NAT)`
   
   * **Descrição:** `No NAT para redes internas → Internet via Fortinet`

5. Salve e **Apply Changes**.

* * *

✅ Com isso:

* Todas as redes internas (`10.0`, `10.3`, `10.4`, `10.5`) vão sair para o Fortinet **sem NAT do pfSense**.

* O Fortinet (200.196.x.x) continua responsável pelo login individual e NAT de saída real.

* * *

Quer que eu te monte também uma **regra extra** para que apenas as redes **LAN internas** saiam SEM NAT, mas se um dia você precisar usar o pfSense sozinho (sem o Fortinet), ele ainda tenha uma regra fallback NAT?

---

Boa 👌  
A ideia é você ter **duas camadas** de regra Outbound NAT:

1. **No NAT** → quando o tráfego for das redes internas para a WAN (passa pelo Fortinet).

2. **NAT normal (fallback)** → se um dia o pfSense precisar sair direto pra Internet sem Fortinet, ele faz NAT pelo IP da WAN (200.196.184.130).

* * *

### 🔹 Como configurar no pfSense

#### 1. Regra de **No NAT** (já falamos, mas recapitulando):

* **Interface:** `WAN`

* **Source:** `REDE_LOCAL` (alias com 10.0.0.0/24, 10.3.0.0/24, 10.4.0.0/23, 10.5.0.0/24)

* **Destination:** `0.0.0.0/0`

* **Translation Address:** `None (no NAT)`

* **Descrição:** `No NAT → Redes internas passam pelo Fortinet`

* * *

#### 2. Regra de **NAT Fallback**:

* **Interface:** `WAN`

* **Source:** `LAN net` (ou `REDE_LOCAL` também, se quiser cobrir todas)

* **Destination:** `any`

* **Translation Address:** `Interface Address` (WAN do pfSense → 200.196.184.130)

* **Descrição:** `Fallback NAT → pfSense sai direto pra Internet`

* * *

⚠️ Importante:

* A **ordem das regras** no pfSense Outbound NAT conta!

* A regra **No NAT** tem que ficar **acima** da regra fallback.

* O pfSense lê de cima para baixo e aplica a primeira que casar com o tráfego.

* * *

✅ Assim, no cenário atual:

* Todo mundo sai pelo Fortinet → sem NAT no pfSense.

* Se um dia o Fortinet cair ou for removido, você só precisa **desabilitar a No NAT** e o fallback NAT já funciona automaticamente.

* * *

Quer que eu te monte o **passo a passo com prints simulados** (valores exatos nos campos do pfSense) para você só copiar e colar, ou prefere em formato de **export de config XML** que você importa direto no pfSense?

---

Perfeito, agora entendi o ponto 👍

Na tela que você me mostrou (essa de edição da regra avançada), você está editando uma regra de **No NAT**.  
Por isso o pfSense não mostra o campo **Translation / Endereço NAT**, já que essa opção está desabilitada quando você marca **Não faça NAT**.

👉 Ou seja:

* Essa tela serve apenas para a **Regra 1 (No NAT)**.

* Para criar a **Regra 2 (Fallback NAT)**, você precisa criar **uma nova regra**, mas **sem marcar** a opção **Não faça NAT**.

* * *

### Como fazer a **Regra 2 (Fallback NAT)**

1. Vá em **Firewall > NAT > Saída**.

2. Clique em **+Adicionar** para criar uma nova regra.

3. Preencha assim:
* **Interface:** `WAN`

* **Família de endereço:** `IPv4`

* **Protocolo:** `Qualquer`

* **Fonte:** `LAN net` (ou melhor: um alias com todas suas redes internas: 10.0.0.0/24, 10.3.0.0/24, 10.4.0.0/23, 10.5.0.0/24)

* **Destino:** `Qualquer`

* **Endereço de Tradução (Translation Address):** `Interface Address` (WAN – 200.196.184.130)

* **Descrição:** `Fallback NAT → WAN`

* * *

### Resultado final esperado:

* **Primeira regra:** No NAT (sem tradução, tráfego passa para o Fortinet).

* **Segunda regra (fallback):** NAT normal, saindo pelo IP da WAN, caso necessário.

🔺 Lembre-se: a ordem importa → a regra **No NAT** deve ficar **acima** da regra de fallback.

* * *

Boa! Agora ficou claro 👍.

Esse **passo 2 (No NAT)** que falamos antes você já está exatamente na tela certa — só que no print que você mandou tem duas opções marcadas que não devem estar juntas:

* **Desabilitado** ❌ → não deve estar marcado (senão a regra nunca entra em ação).

* **Não faça NAT** ✅ → esse sim deve ficar marcado, porque é justamente o que cria o _No NAT_.

* * *

### Como preencher o **Passo 2 – No NAT**

1. Desmarque **Desabilitar essa regra**.

2. Deixe marcado **Não faça NAT**.

3. Configure assim:
* **Dispositivo:** `WAN`

* **Família de endereço:** `IPv4`

* **Protocolo:** `Qualquer`

* **Fonte (Source):** → aqui você escolhe a rede que NÃO vai ser mascarada (ex.: `10.4.0.0/23`, ou se tiver várias, cria uma regra pra cada rede).

* **Destino (Destination):** `Qualquer` (se o pfSense não deixar `any`, você pode colocar `*` ou `0.0.0.0/0`).

* **Tradução / Endereço NAT:** não aparece (porque está em modo "No NAT").

* **Descrição:** algo como `No NAT 10.4.0.0/23 → Fortinet`.

* * *

👉 Isso é o **Passo 2 (No NAT)**: o tráfego dessas redes internas vai sair “cru”, sem ser NATeado pelo pfSense, e vai chegar no Fortinet, que é quem vai aplicar o login.

Depois, logo **abaixo dessa regra**, você cria a **Fallback NAT** (que vai fazer NAT normal para a WAN Address).

* * *

Quer que eu te mande um exemplo pronto de como ficaria a tabela de regras (com as duas: No NAT e Fallback) igualzinho ao que apareceria no pfSense?



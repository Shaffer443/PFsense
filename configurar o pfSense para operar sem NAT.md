O cenário que você descreveu — com um Fortinet distribuindo os IPs e a intenção de ter dispositivos se autenticando individualmente — é a aplicação perfeita para o pfSense em modo de ponte.

Entendendo a topologia da sua rede

* **Roteador Principal (Fortinet):** Atualmente, ele gerencia a rede, atua como gateway, e provavelmente tem um servidor DHCP ativo.
* **pfSense (em modo de ponte):** Vai atuar como um "filtro" ou "porta de entrada" entre o Fortinet e a sua rede interna. Ele inspecionará o tráfego sem fazer NAT, permitindo que os IPs do Fortinet cheguem diretamente aos dispositivos internos.
* **Dispositivos Internos:** Receberão os IPs do Fortinet e, por sua vez, farão a autenticação ou o login individualmente, se essa for a regra configurada no Fortinet.

Confirmação sobre o DHCP

Com a sua nova configuração em modo de ponte:

1. **Não haverá mais DHCP no pfSense.** Você desativou o servidor DHCP na interface LAN e, ao criar a ponte, a função de distribuição de IPs do pfSense foi eliminada.
2. **O DHCP continuará no Fortinet.** Como o Fortinet é o roteador principal, ele continuará a ser responsável por distribuir os endereços IP para os dispositivos na rede.
3. **Os dispositivos internos receberão os IPs do Fortinet.** Quando os dispositivos se conectarem à rede, eles enviarão uma solicitação DHCP, que será respondida pelo servidor DHCP do Fortinet.
4. **A internet funcionará.** Os dispositivos usarão o Fortinet como gateway, e o pfSense, como ponte, permitirá que o tráfego passe sem modificações de endereço.

Próximos passos

1. **Recupere o acesso ao WebGUI do pfSense** usando o console da VM no VirtualBox, conforme explicado anteriormente.
2. **Siga os passos para criar a regra de permissão** na nova interface de gerenciamento (`OPT1`), conforme detalhado na resposta anterior.
3. **Desative a interface LAN antiga (`em1`)** e o pfSense funcionará como uma ponte transparente.
4. **Conecte um dos dispositivos internos** à rede e verifique se ele recebe um IP do Fortinet e se consegue acessar a internet.
5. **Crie as regras de firewall** na aba `OPT1` para controlar e inspecionar o tráfego que passa pela ponte, conforme necessário.

Com esta configuração, o Fortinet continua a gerenciar a rede e distribuir os IPs, enquanto o pfSense (na camada 2) apenas inspeciona o tráfego de forma transparente, permitindo que cada dispositivo seja identificado e autenticado individualmente pelo Fortinet.



Considerando a sua topologia de rede, onde o pfSense está agindo como uma ponte transparente entre o Fortinet e os seus dispositivos, os desktops

**continuarão recebendo endereços via DHCP**, mas do **servidor DHCP do Fortinet**. 

É crucial entender que, no modo de ponte transparente, o pfSense não atua como roteador ou servidor DHCP para a sua rede principal. Ele simplesmente deixa o tráfego fluir entre as interfaces que foram "unidas" na ponte, inspecionando e filtrando as conexões de acordo com as regras de firewall que você definir na interface `OPT1`.

Como isso funciona na sua rede

1. **Fortinet distribui IPs:** O servidor DHCP do Fortinet (que já está ativo) continuará a enviar ofertas de endereços IP para todos os dispositivos na sua rede interna.
2. **pfSense é transparente:** O pfSense, em modo de ponte, não interfere neste processo. Ele simplesmente encaminha as requisições DHCP dos seus desktops para o Fortinet e as respostas do Fortinet de volta para os desktops.
3. **Desktops recebem DHCP:** Os desktops receberão as configurações de rede (endereço IP, máscara de sub-rede, gateway e DNS) diretamente do Fortinet, sem saber que o pfSense está no meio do caminho.
4. **Gateway é o Fortinet:** O gateway dos desktops será o endereço IP do Fortinet (por exemplo, `192.168.1.1` ou o que ele estiver usando), e não o IP de gerenciamento do pfSense (`192.168.0.1`). 

O papel do pfSense

Mesmo em modo de ponte, o pfSense desempenha um papel fundamental:

* **Inspeciona o tráfego:** Ele monitora e filtra todas as conexões entre a rede do Fortinet e os seus desktops, usando as regras de firewall que você criou em **`OPT1`**.
* **Permite autenticação individual:** Como não há NAT, os dispositivos internos manterão seus endereços IP originais ao acessar a internet. Isso permite que o Fortinet os identifique e exija autenticação individual, conforme o seu objetivo. 

Resumo para os desktops

* **Configuração:** Eles devem manter a configuração padrão de **"Obter um endereço IP automaticamente" (DHCP)**.
* **IPs:** Receberão os endereços IP do servidor DHCP do Fortinet.
* **Gateway:** Usarão o IP do Fortinet como gateway.

-----------------------------------------------------------------------------------------------------------------

Essa é uma excelente observação que aponta para a causa do problema. O fato de a sua máquina física estar em uma sub-rede diferente das suas VMs exige um tratamento especial na configuração do VirtualBox. A perda de conexão ocorre porque o tráfego do seu computador host (172.16.x.x) para a VM do pfSense (192.168.0.x) não está sendo roteado corretamente.

O problema é a configuração da rede interna (`Internal Network`) do VirtualBox, que está isolando as VMs do seu computador físico.

Solução: Use uma configuração de rede mista no VirtualBox

Você precisa de uma forma para que a sua máquina host (`172.16.x.x`) possa se comunicar com a interface de gerenciamento do pfSense (`192.168.0.1`), que está em uma sub-rede diferente. A melhor maneira de fazer isso é usando uma configuração mista de redes virtuais.

1. Configurar a rede no VirtualBox

2. **Acesse `Arquivo > Preferências > Rede > Redes somente Host`**.

3. Adicione uma nova rede "Somente Host".

4. Defina o endereço IP e a máscara de sub-rede para esta rede, por exemplo:
   
   * **Endereço IPv4 do Adaptador:** `192.168.0.10`
   * **Máscara de Rede IPv4:** `255.255.255.0`

5. **Desmarque** a opção "Habilitar Servidor DHCP".

6. Reconfigurar os adaptadores da VM do pfSense

7. **Adaptador de Rede 1 (WAN):**
   
   * **Conectado a:** **"Rede Interna"**.
   * **Nome:** `wan_network` (use o mesmo nome que você definiu anteriormente).
   * **Observação:** O adaptador WAN deve continuar em "Rede Interna", pois ele simula a conexão com o Fortinet, que está na rede `172.16.x.x`.

8. **Adaptador de Rede 2 (LAN/OPT1):**
   
   * **Conectado a:** **"Adaptador somente Host"**.
   * **Nome:** `vboxnet0` (ou o nome que você deu à sua rede "Somente Host").
   * **Observação:** Isso permitirá que a sua máquina física (`172.16.32.3`) se comunique com a interface de gerenciamento do pfSense (`192.168.0.1`). A comunicação entre as sub-redes será tratada pelo VirtualBox.

9. Crie a regra de firewall para o acesso de gerenciamento

Após reconfigurar a rede, você pode seguir os passos já discutidos para criar a regra de firewall para acesso de gerenciamento.

* **Action:** `Pass`

* **Interface:** `OPT1` (associada ao Adaptador somente Host).

* **Protocol:** `TCP`

* **Source:** `Single host or alias`. Digite o endereço IP da sua máquina física (`172.16.32.3`).

* **Destination:** `OPT1 address` (`192.168.0.1`).

* **Destination Port Range:** `HTTP` (porta 80) e `HTTPS` (porta 443).

* **Salve e aplique as alterações.**
4. Configure os adaptadores das outras VMs (desktops)
* **Adaptador de Rede 1:** Mantenha a configuração de **"Rede Interna"** com o nome `lan_network`, como planejado. Isso garantirá que elas recebam os IPs do Fortinet através da ponte.

Resumo da topologia de rede

1. O tráfego da internet entra pela **"Rede Interna" (`wan_network`)**, passa pela ponte no pfSense e vai para a **"Rede Interna" (`lan_network`)** onde estão os desktops.
2. O **acesso de gerenciamento** do seu computador host (`172.16.32.3`) para o pfSense (`192.168.0.1`) é feito através da rede **"Somente Host"**. O VirtualBox cuida do roteamento entre a sua máquina física e essa rede.

Essa abordagem garante que o tráfego da rede principal continue a passar pela ponte transparente, enquanto o acesso de gerenciamento do seu computador físico é estabelecido de forma segura e separada.

------------------------

Para configurar o pfSense para operar sem NAT (Network Address Translation), transformando-o em um roteador puro ou em uma ponte transparente, você precisa desativar o recurso de NAT de saída

. 

Cenário 1: Desabilitar o NAT para atuar como um roteador puro

Esse método é usado quando você tem endereços IP públicos roteáveis na sua rede local (LAN) ou em redes segmentadas.

1. Acesse o painel de administração do pfSense.
2. Vá em **Firewall > NAT**.
3. Clique na aba **Outbound** (Saída).
4. Selecione a opção **"Manual Outbound NAT rule generation"** (Geração manual de regras NAT de saída).
5. Clique em **Salvar**.
6. Na lista de regras que será gerada automaticamente, **exclua as regras de NAT** que se aplicam às sub-redes que não devem ter NAT.
7. Clique em **Aplicar Alterações**. 

**Observação**: Para desativar completamente o NAT de saída para todas as interfaces, você pode selecionar a opção **"Disable Outbound NAT rule generation"**. No entanto, isso é recomendado apenas para usuários avançados que sabem o que estão fazendo, pois pode causar perda de conectividade com a internet. 

Cenário 2: Configurar o pfSense como uma ponte transparente

Neste modo, o pfSense atua como uma ponte (bridge), permitindo que o tráfego passe entre duas interfaces (por exemplo, WAN e LAN) sem alterar endereços IP, sem atuar como um roteador. 

1. Acesse a interface do pfSense e vá em **Interfaces > Assignments**.
2. Clique na aba **Bridges**.
3. Clique em **Adicionar** para criar uma nova ponte.
4. Selecione as interfaces que você deseja "unir" na ponte (por exemplo, LAN e WAN).
5. Clique em **Salvar**.
6. Após criar a ponte, desabilite as interfaces originais em **Interfaces > Assignments**.
7. Vá em **Firewall > Rules**. As regras de firewall agora se aplicarão ao tráfego que passa pela ponte. Lembre-se que, como não há NAT, as regras de firewall de entrada (na interface WAN) serão cruciais para a segurança da rede.
8. Para garantir que todo o tráfego passe livremente, adicione regras de permissão adequadas nas interfaces da ponte. 

Pontos importantes

* **Regras de Firewall**: Ao desabilitar o NAT, as regras de firewall de entrada (na interface WAN) tornam-se ainda mais críticas. Você deve ser rigoroso ao configurar as permissões, permitindo apenas o tráfego necessário para a sua rede.
* **Encaminhamento de Portas (Port Forwarding)**: Com o NAT desativado, o encaminhamento de portas se torna redundante, já que os endereços IP da sua rede já estão publicamente roteáveis (se essa for a configuração do seu provedor de internet).
* **Topologia de rede**: A escolha entre desabilitar o NAT ou usar o modo de ponte depende da sua arquitetura de rede. Se a sua rede precisa atuar como um roteador de rede local, desabilite o NAT de saída. Se o objetivo é apenas inspecionar o tráfego que passa, o modo de ponte é a melhor opção.

### --------------------------------------------------------------------------------------------------------------------------------------------

Considerando o ambiente em que o pfSense está rodando como uma VM no VirtualBox e o seu objetivo de criar uma ponte transparente, a configuração das placas de rede virtuais é a chave para o sucesso

. 

Configuração da VM do pfSense

Sua VM do pfSense tem duas placas de rede. A configuração de cada uma delas deve ser diferente para simular o ambiente de rede real.

* **Adaptador de Rede 1 (para a WAN):**
  * **Em `Configurações > Rede > Adaptador 1`**
  * **Conectado a:** **"Rede Interna"**.
  * **Nome:** `wan_network` (crie um nome que identifique essa rede interna).
  * **Por quê?:** Isso simula a conexão com o seu roteador Fortinet. O tráfego da WAN do pfSense estará em uma rede isolada do VirtualBox que representa a conexão "pública".
* **Adaptador de Rede 2 (para a LAN):**
  * **Em `Configurações > Rede > Adaptador 2`**
  * **Conectado a:** **"Rede Interna"**.
  * **Nome:** `lan_network` (crie um nome que identifique a rede local).
  * **Por quê?:** Isso simula a conexão com a sua rede local. O tráfego da LAN do pfSense estará em outra rede isolada do VirtualBox, que representa a sua rede interna. 

Configuração das outras duas VMs (desktops)

As outras duas VMs (que representam os seus desktops) devem se conectar à rede interna que representa a sua rede local, que passa pela ponte do pfSense.

* **Adaptador de Rede 1 (para o Desktop 1 e 2):**
  * **Em `Configurações > Rede > Adaptador 1`** (de cada desktop)
  * **Conectado a:** **"Rede Interna"**.
  * **Nome:** `lan_network` (use o mesmo nome que você definiu para o Adaptador 2 do pfSense).
  * **Por quê?:** Isso garante que os desktops e a interface LAN do pfSense (que agora faz parte da ponte) estejam no mesmo segmento de rede do VirtualBox.

Resumo da topologia

1. **Internet > Adaptador de Rede 1 (`wan_network`) da VM do pfSense**.
2. **VM do pfSense (na ponte) > Adaptador de Rede 2 (`lan_network`) da VM do pfSense**.
3. **Adaptador de Rede 1 (`lan_network`) das VMs dos desktops**.

Com esta configuração, o tráfego da internet entra pela rede interna `wan_network` do pfSense, passa pela ponte, e sai para a rede interna `lan_network`, onde as outras VMs estão conectadas. Você pode gerenciar o tráfego entre essas duas redes internas usando as regras de firewall na interface `OPT1` do pfSense.



----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Configuração BRIDGE

Essa parte é o ponto mais importante e o que geralmente causa confusão. Para que a aba

`BRIDGE0` apareça em **Firewall > Rules**, você precisa atribuir a ponte a uma nova interface.

A listagem `AVALIABLE NETWORK PORTS : BRIDGE0(NO NAT 192.168.0.0)` na sua tela de **Interface Assignments** mostra que a ponte foi criada, mas ainda não foi designada a uma interface funcional no pfSense. Ela é apenas um "bloco de construção" que você criou, mas que precisa ser associado a uma interface lógica para que o pfSense possa aplicar regras de firewall nela. 

Aqui estão os passos detalhados para resolver isso:

1. Desatribuir as interfaces-membro

Primeiro, remova a atribuição das interfaces físicas (`em0` e `em1`) que estão atualmente como WAN e LAN. Elas não devem ter funções de roteamento separadas depois de se tornarem parte da ponte. 

1. Vá em **Interfaces > Assignments**.

2. Na seção **Interface Assignments**, localize as linhas para **WAN** e **LAN**.

3. Defina a **WAN (em0)** e a **LAN (em1)** para **`None`** ou para um status de desativado, se a opção estiver disponível.

4. Clique em **Save**.

5. Atribuir a ponte a uma nova interface

Agora, você precisa atribuir a `BRIDGE0` a uma interface para que ela se torne funcional e apareça na lista de regras.

1. Ainda em **Interfaces > Assignments**, na seção **Available network ports**, você verá a **`BRIDGE0`**.

2. Selecione-a na lista suspensa e clique no botão **`+ Add`**. Isso irá criar uma nova interface, provavelmente chamada `OPT1`.

3. Clique em **Save**.

4. Configurar a nova interface (OPT1)

Após adicionar a interface `OPT1`, ela precisa ser ativada e configurada com o endereço IP que você pretende usar para o gerenciamento da sua ponte.

1. A nova aba **`OPT1`** aparecerá no menu **Interfaces**. Clique nela.

2. Marque a caixa **`Enable`**.

3. Mude a **Description** para algo mais fácil de identificar, como **`BRIDGE_LAN`**.

4. Em **IPv4 Configuration Type**, selecione **`Static IPv4`**.

5. Em **IPv4 Address**, insira o endereço IP que você usará para gerenciar o pfSense, como `192.168.0.1/24`. Este será o seu novo endereço de acesso ao firewall.

6. Clique em **Save** e depois em **Apply Changes**. 

7. Verificar as regras de firewall

Agora, ao navegar para **Firewall > Rules**, você verá a nova aba **`BRIDGE_LAN`** (ou o nome que você deu) ao lado de **Floating**, **WAN** e **LAN**. É aqui que você deve criar as regras para o tráfego que passa pela ponte. 

**Importante:** Se você não reatribuiu `em0` e `em1` para **`None`**, eles ainda podem estar filtrando o tráfego. O método mais limpo e controlado é remover as regras existentes da WAN e LAN e criar novas regras na aba da interface de ponte.

Quando o pfSense impede você de remover ou desatribuir uma interface (como a WAN ou LAN), significa que

**algo ainda está usando essa interface**. O sistema não permite que você remova uma interface ativa para evitar a perda de controle sobre o firewall.

A mensagem de erro "não pode ser deletado" ou o fato de a mudança não ser salva é a pista de que você precisa desabilitar os serviços que estão atrelados a essas interfaces antes de poder desvinculá-las.

O problema mais comum: DHCP Server e IPv4 Config Type

O motivo mais comum para isso acontecer é o **servidor DHCP** ainda estar ativo na interface LAN ou o tipo de configuração de IPv4 estar definido como `Static`.

O que você precisa fazer:

1. Remover o DHCP da LAN

Primeiro, desative o servidor DHCP na interface LAN.

1. Acesse o pfSense e vá em **Services > DHCP Server**.

2. Clique na aba **LAN**.

3. Desmarque a caixa **`Enable`** para desabilitar o serviço.

4. Clique em **Save**.

5. Alterar o tipo de configuração para None

Depois de desabilitar o DHCP, tente novamente definir a configuração da LAN para `None`.

1. Vá em **Interfaces > LAN**.

2. Em **IPv4 Configuration Type**, selecione **`None`**.

3. Em **IPv6 Configuration Type**, selecione **`None`**.

4. Clique em **Save**. Se a mudança for aceita, clique em **Apply Changes**.

5. Repetir para a WAN

A interface WAN também pode ter serviços atrelados. Embora geralmente receba um IP via DHCP, desativá-la pode ser o próximo passo se o problema persistir.

1. Vá em **Interfaces > WAN**.

2. Altere o **IPv4 Configuration Type** e **IPv6 Configuration Type** para **`None`**.

3. Clique em **Save** e em **Apply Changes**.

4. Atribuir a ponte

Agora que as interfaces `em0` e `em1` estão liberadas, você pode atribuir a ponte.

1. Vá para **Interfaces > Assignments**.

2. Na seção **Available network ports**, selecione a sua ponte (`BRIDGE0`).

3. Clique em **`+ Add`**. Isso criará uma nova interface (provavelmente `OPT1`).

4. Clique em **Save** e em **Apply Changes**.

5. Configurar a nova interface de ponte (OPT1)

Siga o passo a passo anterior para configurar a interface `OPT1` com um IP estático para gerenciamento e habilitá-la.

6. Verificar e Criar as Regras

Após configurar e habilitar a `OPT1`, a aba correspondente aparecerá em **Firewall > Rules**. Crie as regras de firewall na aba da **sua nova interface de ponte (`OPT1`)**.

**Importante:** Se o problema persistir mesmo após seguir esses passos, verifique se não há outras dependências:

* **Gateways**: Verifique se a WAN ou LAN não são o gateway padrão de outras rotas em **System > Routing**.
* **Interface Groups**: Verifique se as interfaces não foram adicionadas a nenhum grupo em **Interfaces > Assignments > Interface Groups**.

Sim, você

**deve desmarcar** a opção "Block bogon Networks" na interface WAN para o modo de ponte funcionar corretamente. 

Por que desmarcar a opção "Block bogon Networks"?

* **Evita bloqueios indevidos:** Em uma configuração de ponte, o pfSense está no meio do caminho entre a rede interna e a externa. A opção "Block bogon Networks" bloqueia endereços IP que não estão oficialmente alocados para uso na internet. No seu caso, o tráfego da sua rede interna (que usa IPs privados do Fortinet) passará pela WAN do pfSense. Se o pfSense bloquear bogons, ele pode erroneamente bloquear o tráfego legítimo que vem da sua própria rede interna.
* **Aplica a regra de bloqueio em outro lugar:** A inspeção de bogons deve ocorrer nas bordas da rede, e o seu Fortinet provavelmente já faz isso. O pfSense, operando na camada 2, não precisa duplicar essa funcionalidade.
* **Permite a passagem do tráfego interno:** Ao desabilitar essa opção, você garante que o tráfego interno, que pode incluir endereços bogons ou privados, possa passar de forma transparente pela ponte, conforme a sua intenção. 

Plano de ação revisado

Para garantir que você não perca a conexão novamente, siga esta ordem de passos, acessando o WebGUI com o console da VM pronto para intervir em caso de problemas:

1. **Acesse o WebGUI** pela LAN (`192.168.1.1`).
2. Vá em **Interfaces > WAN**.
3. Desmarque a opção **"Block bogon Networks"** (e também a opção **"Block private networks"**, se estiver marcada).
4. Defina o **IPv4 Configuration Type** e **IPv6 Configuration Type** para **`None`**.
5. Clique em **Salvar** e **Aplicar Alterações**.
6. Acesse **System > Advanced > System Tunables** e confirme que `net.link.bridge.pfil_member` é `0` e `net.link.bridge.pfil_bridge` é `1`.
7. Acesse **Firewall > Rules > OPT1** e confirme que a regra `Pass` para todo o tráfego está no **topo da lista**.
8. **Teste o acesso** pelo IP da `OPT1` (`192.168.0.1`) **antes** de desabilitar a LAN.
9. **Desative o DHCP da LAN** em **Services > DHCP Server > LAN**.
10. **Desative o IP da LAN** em **Interfaces > LAN**, definindo o IPv4 e IPv6 como **`None`**.
11. **Salve** e **Aplique as Alterações**.

Após seguir estes passos, o acesso via `192.168.0.1` deve permanecer ativo, e o pfSense operará como uma ponte transparente.

#-----------------------------------------------------------------------



Você levantou pontos cruciais na configuração de uma ponte no pfSense. Aqui estão as respostas detalhadas para cada uma de suas perguntas, considerando o cenário de um servidor com apenas duas interfaces de rede.

Sobre o passo 6: Desabilitar as interfaces originais

Sim, você precisa realizar o passo 6, mas de uma forma específica. No seu caso, com apenas duas placas de rede, você provavelmente as designou como WAN e LAN no início da configuração.

1. **Vá em `Interfaces > Assignments` e na aba `Interface Assignments`**.
2. **Certifique-se de que a sua nova interface de ponte (ex: `BRIDGE0`) está listada**. Se ela não aparecer, vá para a aba **`Bridges`** e certifique-se de que a ponte foi criada corretamente.
3. **Associe a ponte a uma interface virtual**: Na aba `Interface Assignments`, você deve ter a opção de atribuir a `BRIDGE0` a uma nova interface. Por exemplo, se a sua LAN anterior era `LAN` e a WAN `WAN`, você pode criar uma nova interface, atribuí-la à `BRIDGE0`, e então configurar as regras de firewall nela.
4. **Desative as interfaces originais da ponte**: As interfaces que foram incluídas na ponte (neste caso, as suas duas placas físicas) não devem ter mais configurações de IP. Vá em `Interfaces > [Nome da Interface]` (por exemplo, `Interfaces > LAN`) e defina o tipo de configuração de IPv4 e IPv6 para **`None`**. Se você usou a interface WAN, faça o mesmo.

**Importante**: Como o pfSense em modo ponte atua como uma camada 2 (como um switch), ele não precisa de um endereço IP nas interfaces que fazem parte da ponte. As configurações de IP, se necessárias, serão feitas na própria interface da ponte (`BRIDGE0`).

Sobre as regras de firewall (passos 7 e 8)

Ao criar uma ponte, o comportamento do firewall muda e você precisa ajustar as regras.

Como o firewall funciona em modo ponte

* **Padrão**: Por padrão, o pfSense filtra o tráfego nas interfaces _membro_ da ponte (as placas físicas), não na interface de ponte em si (`BRIDGE0`)..
* **Ajuste**: Para ter mais controle, você pode mudar essa configuração avançada para que as regras sejam aplicadas na interface de ponte. Isso é feito em `System > Advanced > System Tunables`. Procure por `net.link.bridge.pfil_member` e defina seu valor para `0` (desativa a filtragem nos membros), e `net.link.bridge.pfil_bridge` para `1` (ativa a filtragem na ponte). 

Regras de firewall essenciais para a ponte

Para que o tráfego flua de forma transparente, você precisará de uma regra básica que permita a passagem do tráfego.

1. **Vá em `Firewall > Rules`**.
2. Selecione a aba da sua nova **interface de ponte** (por exemplo, `BRIDGE0`).
3. Crie uma nova regra de passagem (**`Pass`**) com as seguintes configurações:
   * **Action**: `Pass`
   * **Interface**: `BRIDGE0`
   * **Address Family**: `IPv4+IPv6`
   * **Protocol**: `Any`
   * **Source**: `Any`
   * **Destination**: `Any`
   * **Description**: `Allow all traffic on bridge`
4. **Clique em `Save` e depois em `Apply Changes`**.

**Importante**: Esta regra permite que todo o tráfego passe pela ponte sem restrições. É um bom ponto de partida para garantir que a conectividade esteja funcionando. A partir daí, você pode refinar as regras para bloquear tráfego específico, se necessário.

Resumo dos passos:

1. **Criar a Ponte**: Conclua a criação da ponte em `Interfaces > Bridges`.
2. **Atribuir a Ponte**: Atribua a nova interface de ponte (`BRIDGE0`) em `Interfaces > Assignments`.
3. **Desativar IPs**: Nas interfaces físicas (WAN e LAN), defina o tipo de configuração de IPv4 e IPv6 para `None`.
4. **Ajustar Filtro (Opcional)**: Vá em `System > Advanced > System Tunables` e configure `net.link.bridge.pfil_member = 0` e `net.link.bridge.pfil_bridge = 1`.
5. **Criar Regra de Permissão**: Em `Firewall > Rules`, na aba da interface de ponte, crie uma regra `Pass` para permitir todo o tráfego.



Plano de ação revisado

Para garantir que você não perca a conexão novamente, siga esta ordem de passos, acessando o WebGUI com o console da VM pronto para intervir em caso de problemas:

1. **Acesse o WebGUI** pela LAN (`192.168.1.1`).
2. Vá em **Interfaces > WAN**.
3. Desmarque a opção **"Block bogon Networks"** (e também a opção **"Block private networks"**, se estiver marcada).
4. Defina o **IPv4 Configuration Type** e **IPv6 Configuration Type** para **`None`**.
5. Clique em **Salvar** e **Aplicar Alterações**.
6. Acesse **System > Advanced > System Tunables** e confirme que `net.link.bridge.pfil_member` é `0` e `net.link.bridge.pfil_bridge` é `1`.
7. Acesse **Firewall > Rules > OPT1** e confirme que a regra `Pass` para todo o tráfego está no **topo da lista**.
8. **Teste o acesso** pelo IP da `OPT1` (`192.168.0.1`) **antes** de desabilitar a LAN.
9. **Desative o DHCP da LAN** em **Services > DHCP Server > LAN**.
10. **Desative o IP da LAN** em **Interfaces > LAN**, definindo o IPv4 e IPv6 como **`None`**.
11. **Salve** e **Aplique as Alterações**.

Após seguir estes passos, o acesso via `192.168.0.1` deve permanecer ativo, e o pfSense operará como uma ponte transparente.



--------------------------------------------------------------------------------------------------------

# Configurações e Informações Extras

Depois de criar a ponte, as interfaces

**WAN** e **LAN** que você tinha anteriormente perdem sua função individual de roteamento. A filtragem do firewall agora pode ser aplicada de duas maneiras: 

1. **Nas interfaces-membro** (WAN e LAN), por padrão.
2. **Na interface da ponte** (`BRIDGE0`), que você pode habilitar com ajustes no sistema.

Considerando que você está fazendo a ponte para ter controle sobre o tráfego que passa por ela, o método mais indicado é criar as regras diretamente na interface de ponte (`BRIDGE0`).

Como atuar nas regras de firewall

1. **Habilite a filtragem na interface da ponte**:
   * Vá em **System > Advanced** e clique na aba **System Tunables**.
   * Procure por `net.link.bridge.pfil_member` e defina seu valor para **`0`**. Isso desativa a filtragem nas interfaces-membro.
   * Procure por `net.link.bridge.pfil_bridge` e defina seu valor para **`1`**. Isso ativa a filtragem na interface da ponte (`BRIDGE0`).
   * Clique em **Save** e depois em **Apply Changes**.
2. **Crie as regras na aba da interface de ponte**:
   * Volte para **Firewall > Rules**.
   * Agora, uma nova aba, correspondente à sua interface de ponte (por exemplo, `BRIDGE0`), deve ter aparecido. **É nela que você deve criar as regras**.
   * As regras existentes nas abas **WAN** e **LAN** que eram para controle de tráfego entre redes não serão mais aplicadas na ponte e podem ser removidas para evitar confusão. 

Regra de exemplo para liberar todo o tráfego (para teste)

Para garantir que a ponte está funcionando, crie uma regra simples que permite todo o tráfego:

1. Na aba da interface da ponte (ex: **`BRIDGE0`**), clique em **Add** (seta para cima ou para baixo).
2. **Action**: `Pass`
3. **Interface**: `BRIDGE0`
4. **Address Family**: `IPv4+IPv6`
5. **Protocol**: `Any`
6. **Source**: `Any`
7. **Destination**: `Any`
8. **Description**: `Permitir todo tráfego na ponte`
9. Clique em **Save** e depois em **Apply Changes**.

**Observação**: Após os testes, você pode refinar as regras para garantir a segurança, como criar regras específicas para o tráfego que você deseja inspecionar ou bloquear.



---------------------------------------------------------------------

Em uma máquina virtual do VirtualBox, o console é acessado diretamente pela janela da VM, como se você estivesse usando um monitor e teclado conectados ao servidor físico

. Você não perdeu o acesso ao console, apenas ao WebGUI, que depende do endereço IP. 

Como acessar o console da VM no VirtualBox

1. Abra o VirtualBox e selecione a máquina virtual do pfSense.
2. Clique em **"Start"** ou dê um duplo clique na VM para iniciá-la.
3. A janela da VM será aberta, mostrando a inicialização do pfSense.
4. Quando o pfSense terminar de carregar, o menu de console aparecerá na tela.

Próximos passos para recuperar o acesso

1. **Acesse a máquina virtual do pfSense** conforme descrito acima.
2. No menu do console, digite **`1`** para "Assign Interfaces" e pressione `Enter`.
3. O pfSense fará a detecção das interfaces de rede. Confirme a atribuição das interfaces WAN (`em0`) e LAN (`em1`).
4. Após a confirmação, o pfSense pedirá para configurar as interfaces. Configure o IP da LAN (`em1`) para o IP que você precisa para gerenciar o sistema, como `192.168.1.1`.
5. Quando o processo for concluído, você deve ter acesso ao WebGUI novamente através do navegador do seu computador host (se a configuração de rede da VM permitir).

Continue a configuração da ponte

Agora que você tem acesso ao WebGUI novamente, pode prosseguir com a configuração da ponte seguindo os passos originais, mas com cuidado extra.

1. **Desabilite o DHCP na LAN**: Vá em **Services > DHCP Server** e desmarque a opção "Enable" na aba LAN.
2. **Atribua a ponte a uma nova interface**: Em **Interfaces > Assignments**, atribua a `BRIDGE0` a uma nova interface, por exemplo, `OPT1`.
3. **Configure o IP de gerenciamento**: Acesse **Interfaces > OPT1**, habilite a interface e defina um IP estático para gerenciamento, como `192.168.0.1/24`.
4. **Remova o IP da LAN**: Somente depois de ter acesso pela nova interface (`OPT1`), vá em **Interfaces > LAN** e defina o tipo de configuração de IPv4 para **`None`**.
5. **Ajuste as regras**: Crie as regras de firewall na nova aba `OPT1` (em **Firewall > Rules**) para gerenciar o tráfego da ponte.



Seu problema é compreensível e muito comum ao configurar uma ponte transparente no pfSense. A confusão surge porque a forma como o pfSense gerencia as interfaces muda completamente quando a ponte é ativada.

O motivo da perda de conexão

O pfSense, ao ter a LAN e a WAN unidas em uma ponte, começa a tratar essas interfaces físicas como "membros" (camada 2) e a ponte (`BRIDGE0`) como a interface principal de rede (camada 3). Quando você define a LAN para **`None`**:

1. **A interface LAN (`em1`) perde seu endereço IP**.
2. **O acesso de gerenciamento (WebGUI)**, que dependia do IP da LAN, **é interrompido**.
3. **A nova interface de gerenciamento**, `OPT1` (associada à `BRIDGE0`), **não possui regras de firewall** para permitir o acesso.

Mesmo que você esteja acessando via `192.168.0.1`, o tráfego ainda precisa passar por regras de firewall que não existem para a `OPT1`. O pfSense, por padrão, bloqueia qualquer acesso a interfaces que não sejam a LAN tradicional, até que você crie regras específicas para a nova interface.

A solução: Permissão na nova interface de gerenciamento

Para evitar perder o acesso, você precisa criar a regra de firewall **antes** de desativar a interface LAN antiga.

**Passo a passo para corrigir e continuar:**

1. **Conecte-se via console** (na janela da VM) e restaure o IP da LAN (`em1`) para `192.168.1.1` usando a opção **`2`** do menu.
2. **Volte ao navegador** e acesse o pfSense em `192.168.1.1`.
3. **Habilite a filtragem na interface da ponte**:
   * Vá em **System > Advanced > System Tunables**.
   * Defina `net.link.bridge.pfil_member` para **`0`**.
   * Defina `net.link.bridge.pfil_bridge` para **`1`**.
   * Salve e aplique as mudanças.
4. **Crie a regra de permissão na nova interface (`OPT1`)**:
   * Vá em **Firewall > Rules**.
   * Clique na aba da sua nova interface (`OPT1`).
   * Adicione uma nova regra com as seguintes configurações:
     * **Action**: `Pass`
     * **Interface**: `OPT1`
     * **Address Family**: `IPv4+IPv6`
     * **Protocol**: `Any`
     * **Source**: `Any`
     * **Destination**: `Any`
     * **Description**: `Permitir acesso total na ponte`
   * Salve e aplique as mudanças.
5. **Teste o acesso pela nova interface**:
   * Abra uma nova aba do navegador e tente acessar o pfSense pelo novo IP (`192.168.0.1`).
   * Se funcionar, você está pronto para desativar a LAN antiga.
6. **Desative o IP da interface LAN (`em1`)**:
   * Vá em **Interfaces > LAN**.
   * Em **IPv4 Configuration Type**, selecione **`None`**.
   * Em **IPv6 Configuration Type**, selecione **`None`**.
   * Salve e aplique as mudanças.

**Importante**: O ajuste `net.link.bridge.pfil_bridge = 1` é crucial, pois ele direciona o pfSense a aplicar as regras na interface de ponte (`OPT1`) em vez das interfaces-membro (LAN e WAN). Se você não fizer isso, as regras que você criar não terão efeito.

Após seguir estes passos, o acesso via `192.168.0.1` funcionará, e o pfSense agirá como uma ponte transparente entre a LAN e a WAN, inspecionando o tráfego de acordo com as regras que você definiu para a interface `OPT1`.



Parece que o problema persiste porque o acesso ao WebGUI pela interface

`OPT1` está sendo bloqueado pelas regras de firewall, mesmo com o IP configurado. Quando você remove o IP da LAN, o pfSense perde o caminho de gerenciamento padrão e a nova interface `OPT1` não está configurada para permitir esse acesso.

Aqui estão os pontos cruciais que você precisa verificar, pois um deles pode estar causando o problema:

1. Ordem das regras de firewall

A ordem das regras é fundamental no pfSense. A regra que permite o acesso ao gerenciamento deve ser a primeira na lista da aba `OPT1`.

* **Verifique se a regra `Pass` para a `OPT1` é a primeira da lista.** Se houver regras de bloqueio antes dela, elas podem estar impedindo a conexão.

* **Posicione a regra no topo:** Use a seta para cima para mover a regra que permite o acesso ao topo da lista.

* **A regra de `anti-lockout` da LAN não se aplica à OPT1:** A regra de anti-lockout padrão só funciona na interface LAN. Ao remover o IP da LAN, essa proteção é desativada, e você precisa criar uma regra equivalente na OPT1.
2. Configurações avançadas do firewall (`System Tunables`)

Já mencionamos isso, mas é o ponto mais crítico para a filtragem em bridges.

* **Acesse `System > Advanced > System Tunables`.**

* **`net.link.bridge.pfil_member`:** deve ser `0`. (Desativa a filtragem nas interfaces-membro, como `em1`).

* **`net.link.bridge.pfil_bridge`:** deve ser `1`. (Ativa a filtragem na interface da bridge, `BRIDGE0`, que é a `OPT1`).

* **Reinicie o pfSense** após alterar essas configurações para garantir que elas sejam aplicadas corretamente.
3. Falha no processo de aplicação das configurações

Pode haver um problema na forma como o pfSense aplica as alterações em cascata.

**Sequência de passos para tentar novamente (com acesso via console garantido):**

1. **Acesse o console da VM** para garantir que você tenha um ponto de recuperação.
2. **Restaure o IP da LAN** (`em1`) para o valor original (`192.168.1.1`).
3. **Acesse o pfSense via navegador** pelo IP da LAN.
4. **Verifique os `System Tunables`** (`net.link.bridge.pfil_member = 0` e `net.link.bridge.pfil_bridge = 1`).
5. **Crie a regra de permissão** na aba `OPT1` (`Pass`, `Interface = OPT1`, `Protocol = Any`, `Source = Any`, `Destination = Any`). **Coloque-a no topo da lista.**
6. **Teste o acesso** via `192.168.0.1` (o IP da OPT1) **sem desabilitar a LAN ainda**. Se o acesso funcionar, você pode continuar.
7. **Desabilite o DHCP** na interface LAN (Services > DHCP Server > LAN).
8. **Vá em `Interfaces > LAN`**, selecione `None` para IPv4 e IPv6, e clique em **`Save`** e **`Apply Changes`**.

Onde a WAN se encaixa

Sua observação sobre o IP da WAN é normal.

* A WAN (`em0`) está pegando um IP via DHCP do Fortinet, o que é o comportamento esperado.
* Como ela está na bridge, você não precisará fazer nada nela.
* A perda de acesso quando você mexe na LAN acontece porque você está removendo a única rota de gerenciamento que o pfSense tinha certeza de que existia.

Cenário de teste (para isolar o problema)

Se as etapas acima não resolverem, tente um cenário mais simples para isolar a causa:

1. **Conecte-se via console** e restaure a LAN para o IP original.
2. **Vá em `System > Advanced > Admin Access`** e **desabilite temporariamente a regra de `anti-lockout`**. Isso pode ajudar a identificar se é a regra de firewall que está causando o problema. **Lembre-se de reabilitá-la depois**.
3. **Repita os passos** para desativar o IP da LAN.

Se o acesso funcionar com a regra de `anti-lockout` desabilitada, a questão é definitivamente a regra de firewall da `OPT1`. A regra `Pass` genérica deve funcionar, mas há casos complexos de bridging onde isso não ocorre.

Comunique o resultado da sua próxima tentativa para podermos refinar o diagnóstico.

-----------------------------------------------------------------------------------------------------------------------------------------------------------

Há um detalhe crítico que parece estar faltando no processo e que está causando a perda de acesso: as diferentes sub-redes para gerenciamento e tráfego de rede

. 

A causa da perda de conexão

O pfSense, quando em modo de ponte, permite que o tráfego passe de forma transparente entre a WAN e a LAN (em uma "ponte") e o firewall de gerenciamento (`OPT1`) atua em uma camada diferente. No entanto, o seu acesso ao WebGUI pelo IP da `OPT1` (192.168.0.1) só funciona enquanto a LAN (192.168.1.1) ainda está ativa.

Quando você desativa a LAN, o pfSense perde a rota de acesso padrão. O acesso pela `OPT1` falha porque:

* **Regra de firewall:** A regra que você criou na `OPT1` para permitir o acesso ao WebGUI pode estar incorreta ou ser insuficiente. Uma regra `Pass` genérica pode ser bloqueada por outras regras implícitas.
* **Tráfego de gerenciamento:** O tráfego para a `OPT1` pode não estar sendo encaminhado corretamente pelo Fortinet ou por suas regras de VM.
* **Loop de rede:** Em cenários de bridge complexos, loops podem ocorrer e causar interrupções.

Respostas às suas perguntas

Vai funcionar com os ranges de IP?

Sim, a sua configuração de ranges de IP é totalmente viável.

* **Fortinet (172.16.x.x):** O Fortinet continuará a distribuir IPs nesta faixa. Como o pfSense está em modo de ponte, os dispositivos conectados à rede interna (através da `lan_network`) receberão IPs do Fortinet.
* **pfSense (`OPT1` em 192.168.0.x):** O IP de gerenciamento do pfSense (`192.168.0.1`) está em uma sub-rede diferente. Isso é uma boa prática para isolar o acesso administrativo do tráfego principal da rede.
* **Mascara `255.255.0.0` (opcional):** A máscara `255.255.0.0` permite uma rede maior (`/16`). No entanto, para o cenário de ponte, ela não é estritamente necessária e você pode usar `255.255.255.0` se preferir manter as sub-redes separadas.

Qual o gateway?

Essa é a parte crucial e que costuma causar a falha:

* **O gateway dos dispositivos não é o pfSense.** Os dispositivos, como os desktops, receberão um endereço IP e o gateway do Fortinet. Eles usarão o Fortinet para sair para a internet.
* **O pfSense não é o gateway da rede.** No modo de ponte, o pfSense inspeciona o tráfego que passa, mas os dispositivos ainda usam o Fortinet para rotear o tráfego.

O que pode estar faltando

O problema pode ser o **acesso de gerenciamento** à `OPT1` após desativar a LAN. A regra que você criou pode não estar permitindo que o tráfego do Fortinet chegue até o pfSense para fins de gerenciamento.

**Plano de ação revisado e mais detalhado:**

1. **Acesse o pfSense pelo console da VM** e restaure a configuração da LAN (`192.168.1.1`).
2. **No navegador, acesse o pfSense por `192.168.1.1`**.
3. **Vá em `System > Advanced > System Tunables`** e defina `net.link.bridge.pfil_member` como `0` e `net.link.bridge.pfil_bridge` como `1`. Salve e aplique.
4. **Vá em `Firewall > Rules > OPT1`**.
   * **Crie uma regra `Pass` no topo:** `Interface: OPT1`, `Address Family: IPv4`, `Source: any`, `Destination: any`. Esta regra é para teste e deve garantir a conectividade.
   * **Crie uma regra para gerenciar o acesso:** `Interface: OPT1`, `Protocol: TCP`, `Source: [IP da sua máquina]`, `Destination: OPT1 address`, `Destination Port: 443 (HTTPS)`. Essa regra garante que apenas a sua máquina tenha acesso.
5. **Salve e aplique as regras.**
6. **Teste o acesso pela `OPT1`** (`192.168.0.1`). Se funcionar, você pode continuar.
7. **Desabilite o DHCP da LAN** em `Services > DHCP Server`.
8. **Vá em `Interfaces > LAN`**, defina o **IPv4 e IPv6** para **`None`**.
9. **Vá em `Interfaces > WAN`**, desmarque a opção "Block bogon networks", e defina **IPv4 e IPv6** para **`None`**.
10. **Salve e aplique as alterações.**

Se o problema persistir após seguir estes passos, pode haver alguma regra de firewall no Fortinet que esteja bloqueando o acesso do `172.16.x.x` para o `192.168.0.1`. Se esse for o caso, você precisará adicionar uma regra de permissão no Fortinet.

-------------------------------

# COnfiguração no Virtual Box 7.2.1 (Caso precise)

Entendido. Na versão 7.1.2 do VirtualBox, o acesso ao gerenciador de redes é diferente do caminho mais antigo (

`Arquivo > Preferências`).

O novo caminho para o Gerenciador de Redes na versão 7.1.2

1. Abra o VirtualBox e vá no menu **`Ferramentas`** (Tools).
2. Clique em **`Gerenciador de Redes`** (ou use o atalho `Ctrl + H`).

Como configurar a rede host-only

1. No **`Gerenciador de Redes`**, vá na aba **`Redes somente de Hospedeiro`**.
2. Clique em **`Criar`** para adicionar uma nova rede host-only (ela deve ser nomeada como `vboxnet0` por padrão).
3. Com a nova rede selecionada, clique no ícone **`Configurar`** (uma chave de fenda) para editar as propriedades.
4. Na aba **`Adaptador`**, defina:
   * **Endereço IPv4 do Adaptador:** `192.168.0.10`
   * **Máscara de Rede IPv4:** `255.255.255.0`
5. Na aba **`Servidor DHCP`**, **desmarque a opção `Habilitar Servidor`**.
6. Clique em **`Aplicar`** ou **`OK`** para salvar as alterações.

Reconfigurar os adaptadores da VM do pfSense

Agora, você pode associar a nova rede host-only ao Adaptador 2 da sua VM do pfSense.

1. **Acesse as configurações da VM do pfSense** (`Configurações > Rede > Adaptador 2`).
2. **Conectado a:** Selecione **`Placa de rede exclusiva de hospedeiro`**.
3. **Nome:** Selecione a rede que você acabou de configurar (`vboxnet0`).

Com esta configuração, o pfSense terá a interface `OPT1` na rede `192.168.0.x`, que se comunicará diretamente com sua máquina física, permitindo que você acesse a interface de gerenciamento (`192.168.0.1`) a partir da sua máquina física (`172.16.32.3`).



Sim, a sua compreensão está correta.

Configuração de rede da VM do pfSense

* **Adaptador 1 (WAN):** Conectado à **Rede Interna** chamada `wan_network`. Isso simula a conexão com a rede "externa" que vem do Fortinet.
* **Adaptador 2 (LAN/OPT1):** Conectado à **Placa de rede exclusiva de hospedeiro** (`vboxnet0`, ou o nome que você configurou). Isso permite que sua máquina física se comunique diretamente com o pfSense para fins de gerenciamento.

Como a sua rede funcionará

1. **Tráfego de rede transparente:** A ponte que você está configurando no pfSense conecta a `wan_network` e a `lan_network` (onde os outros clientes estão). O tráfego passará por essa ponte para que o pfSense possa inspecioná-lo, mas sem fazer NAT.
2. **Acesso de gerenciamento isolado:** A interface `OPT1` (associada à `vboxnet0`) servirá exclusivamente para o seu gerenciamento. A comunicação entre a sua máquina física e o pfSense passará por essa rede host-only, permitindo que você acesse a interface WebGUI do pfSense.
3. **Comunicação entre VMs:** As VMs clientes (Linux Mint, Windows 7) também devem estar na **Rede Interna** (`lan_network`), para que possam se comunicar com o pfSense e, através da ponte, com o Fortinet e a internet.

Próximos passos

Agora que a configuração da rede no VirtualBox está alinhada, você pode seguir o plano de ação passo a passo para:

1. **Restaurar o acesso:** Acessar o pfSense via console e reconfigurar a LAN temporariamente.
2. **Criar a regra de firewall:** Na interface `OPT1`, criar a regra para permitir o acesso de gerenciamento a partir do IP da sua máquina física (`172.16.32.3`).
3. **Desativar o IP da LAN e WAN:** Após confirmar que o acesso de gerenciamento funciona pela `OPT1`, desativar os IPs das interfaces LAN e WAN.
4. **Excluir as regras antigas:** Remover as regras de firewall das abas LAN e WAN para evitar confusão.
5. **Testar a conexão:** Verificar se os clientes (outras VMs) recebem IPs do Fortinet e têm acesso à internet através da ponte.

--------------------------------------------------------------------------------------

# SOlução 02:

Essa é uma questão excelente e mostra uma falha na nossa abordagem anterior. Se o seu IP muda, a rede host-only não é a solução ideal porque depende de uma sub-rede fixa. Eliminar a rede host-only não é o caminho, mas sim ajustar a configuração para que a comunicação seja flexível.

Aqui está a abordagem mais robusta para o seu cenário: usar o modo NAT no VirtualBox para o adaptador de gerenciamento, que irá mapear uma porta específica para a interface WebGUI do pfSense, independentemente do IP da sua máquina host.

O novo plano de ação

1. Reconfigure o Adaptador 2 da VM do pfSense

2. **Desligue a VM do pfSense**.

3. **Acesse `Configurações > Rede > Adaptador 2`**.

4. **Conectado a:** Selecione **"NAT"**. Isso criará uma rede NAT interna para a VM, que permitirá que o seu computador físico acesse a VM de forma segura.

5. Configure o Port Forwarding para a NAT

6. Com a opção "NAT" selecionada, clique no botão **"Avançado"**.

7. Clique em **"Port Forwarding"**.

8. Adicione uma nova regra:
   
   * **Nome:** `pfSense_HTTPS`
   * **Protocolo:** `TCP`
   * **Endereço IP Hospedeiro:** Deixe em branco (permite acesso de qualquer interface da sua máquina).
   * **Porta Hospedeiro:** `8443` (ou qualquer porta alta que não esteja em uso, para evitar conflitos).
   * **Endereço IP Convidado:** `192.168.0.1` (o IP de gerenciamento que você usará no pfSense).
   * **Porta Convidado:** `443` (a porta padrão do WebGUI HTTPS).

9. Clique em **OK** para salvar a regra de Port Forwarding.

10. Inicie a VM do pfSense e faça login

11. Inicie a VM do pfSense.

12. No console, configure o IP da interface `OPT1` para `192.168.0.1`.

13. No seu navegador, você poderá acessar o WebGUI do pfSense digitando: `https://localhost:8443` ou `https://127.0.0.1:8443`.

14. Configure o pfSense

Agora você pode seguir o restante do plano de ação dentro do pfSense:

1. **Ajuste os `System Tunables`**.
2. **Exclua as regras de LAN e WAN**.
3. **Desabilite o DHCP da LAN**.
4. **Defina os IPs das interfaces LAN e WAN como `None`**.

Vantagens desta abordagem

* **Não depende do seu IP DHCP:** Como o acesso é feito pelo `localhost` e mapeia uma porta para o pfSense, o IP dinâmico da sua máquina física não causará problemas de acesso.
* **Segurança:** O acesso é feito através de um túnel seguro (Port Forwarding), sem expor diretamente o pfSense à sua rede física.

Com esta configuração, você terá uma solução robusta para acessar o pfSense a partir do seu computador físico, mesmo que o seu IP mude.



-------------------------------------------------------------------------------------------------------------------------------------------------------------------

Abordagem alternativa: Modo Promíscuo para Adaptadores de Rede

No VirtualBox, o modo promíscuo em adaptadores de rede `bridge` ou `internal` pode ser necessário para que o tráfego seja inspecionado corretamente pelo pfSense. 

1. **Desligue a VM do pfSense**.
2. Acesse **Configurações > Rede** para cada adaptador de rede (`Adaptador 1` e `Adaptador 2`).
3. Clique em **"Avançado"**.
4. No campo **"Modo Promíscuo"**, selecione **"Permitir tudo"** (ou "Allow all").
5. Repita para o outro adaptador de rede do pfSense.

Outros possíveis problemas a verificar

1. **Bloqueio de redes privadas**:
   * No console do pfSense, use a opção para **`Shell`**.
   * Digite `pfctl -d` para desativar temporariamente o filtro de pacotes.
   * No WebGUI, vá em **Interfaces > WAN** e **desmarque** a opção "Block private networks and loopback addresses".
2. **Verifique as rotas**:
   * Acesse o console do pfSense.
   * Use a opção `Shell` e digite `netstat -rn`.
   * Verifique se as rotas da interface de gerenciamento (OPT1) estão corretas após a desativação da LAN. 

Nova tentativa de plano de ação (com console pronto)

1. **Restaure a configuração da LAN:** Use o console para reatribuir o IP da LAN (`192.168.1.1`).
2. **Ajuste as configurações no WebGUI**:
   * Ajuste os **`System Tunables`** para `pfil_member = 0` e `pfil_bridge = 1`.
   * Vá em **`Interfaces > WAN`**: desmarque "Block bogon Networks" e "Block private networks".
   * Vá em **`Firewall > Rules > OPT1`**: crie uma regra `Pass` genérica (qualquer origem, qualquer protocolo, qualquer destino) no **topo** da lista para testes.
3. **Teste o acesso pela OPT1**: Antes de desativar a LAN, tente acessar `192.168.0.1` do seu navegador. Se funcionar, avance.
4. **Desative a LAN e a WAN**:
   * Em **`Services > DHCP Server > LAN`**, desmarque "Enable".
   * Em **`Interfaces > LAN`**, defina o IP para `None`.
   * Em **`Interfaces > WAN`**, defina o IP para `None`.
5. **Verifique a conectividade**:
   * No console da VM, verifique se as interfaces estão sem IP.
   * Tente acessar o pfSense pelo `192.168.0.1` do seu navegador.

Se o problema persistir: Acesso à internet do Fortinet

Se a sua máquina física recebe o IP do Fortinet (172.16.x.x), é possível que o VirtualBox precise de ajuda para rotear o tráfego da rede interna (`lan_network`) para a rede externa.

1. Verifique a **interface de rede do VirtualBox**.
2. Considere usar um terceiro adaptador de rede no pfSense, configurado com NAT, apenas para o gerenciamento. Essa seria uma solução temporária para você poder configurar o restante.

--------------------------------------------------------------------------------------------------------------------------------------------

# Caso minha WAN seja DHCP do FORTINET



Ótima observação. Se o seu IP muda (DHCP), você não pode usar um IP fixo na regra de firewall do pfSense para permitir o acesso de gerenciamento. Isso requer uma pequena mudança na abordagem.

Em vez de usar um IP fixo (`172.16.32.3`) como origem na regra de firewall da `OPT1`, você deve usar a sub-rede completa da sua máquina física.

Plano de ação ajustado

1. **Acesse o console da VM** e restaure a configuração da LAN (`192.168.1.1`).
2. **No navegador, acesse o pfSense por `192.168.1.1`**.
3. **Verifique os `System Tunables`** (`net.link.bridge.pfil_member = 0`, `net.link.bridge.pfil_bridge = 1`). Salve e aplique.
4. **Vá em `Firewall > Rules > OPT1`**.
   * **Crie uma regra `Pass` no topo:**
     * **Action:** `Pass`
     * **Interface:** `OPT1`
     * **Protocol:** `TCP`
     * **Source:** `Network`. Digite a sub-rede da sua máquina física (`172.16.32.0/23`). O prefixo `/23` é necessário porque sua máscara de sub-rede (`255.255.254.0`) abrange as redes `172.16.32.x` e `172.16.33.x`.
     * **Destination:** `OPT1 address` (`192.168.0.1`).
     * **Destination Port Range:** `HTTP` (80) e `HTTPS` (443).
   * **Salve e aplique as mudanças.**
5. **Desabilite o DHCP da LAN** em `Services > DHCP Server > LAN`.
6. **Desative o IP da LAN** em `Interfaces > LAN` (`None`).
7. **Desative o IP da WAN** em `Interfaces > WAN` (`None`) e desmarque "Block bogon networks" e "Block private networks".
8. **Salve e aplique as alterações.**

Por que usar a sub-rede (`172.16.32.0/23`)?

* Ao usar a sub-rede como origem, a regra permite o acesso de qualquer endereço IP dentro dessa faixa, incluindo o seu.
* Quando o seu IP DHCP muda (dentro da faixa `172.16.32.x` e `172.16.33.x`), a regra continuará permitindo o acesso, pois ela abrange a sub-rede completa.

Considerações adicionais

* **Acesso de gerenciamento:** Agora, o acesso de gerenciamento ao pfSense será permitido de qualquer máquina que esteja na mesma sub-rede que a sua máquina física.
* **Segurança:** Se a sub-rede `172.16.32.0/23` tiver muitos dispositivos, considere usar um IP estático para sua máquina física ou restringir ainda mais o acesso (se possível).
* **Regras de firewall:** A regra de gerenciamento na `OPT1` deve estar acima de qualquer regra de bloqueio genérico para garantir que o acesso não seja negado.

-------------------------------------------------------------------------------------------------

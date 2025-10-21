Se você tem apenas duas placas de rede e precisa que o Fortinet gerencie a autenticação, a topologia de ponte transparente é o caminho correto. O fato de você perder a conexão ao desabilitar o IP da LAN, mesmo com a interface de gerenciamento (

`OPT1`) configurada, indica que a comunicação entre sua máquina de acesso e o pfSense está sendo interrompida. 

Vamos rever a configuração, focando no ambiente do VirtualBox para garantir que o acesso funcione de forma consistente.

1. Reconfigure o acesso de gerenciamento via rede exclusiva de hospedeiro

Usar uma rede host-only para o gerenciamento é a forma mais confiável de garantir que seu acesso ao pfSense não seja perdido, independentemente do que aconteça nas redes WAN e LAN.

1. **Desligue a VM do pfSense**.

2. **Abra o VirtualBox** e vá em `Ferramentas > Gerenciador de Redes`.

3. Vá em **`Redes somente de Hospedeiro`**. Certifique-se de que a rede `vboxnet0` está configurada com um IP estático (por exemplo, `192.168.0.10`) e que o **servidor DHCP está desabilitado**.

4. **Acesse as configurações da VM do pfSense** (`Configurações > Rede`).

5. **Adaptador 2 (LAN/OPT1):** Mude a configuração para **`Placa de rede exclusiva de hospedeiro`** e selecione `vboxnet0`.

6. Configure a interface de gerenciamento (OPT1) no pfSense

7. **Inicie a VM do pfSense** e, pelo console, redefina a interface LAN para o IP original (`10.0.0.1`) para ter acesso ao WebGUI.

8. **No WebGUI, vá em `Interfaces > Assignments`**.

9. Associe a **`vboxnet0`** à interface `OPT1`.

10. Vá em **`Interfaces > OPT1`**, habilite-a e defina o **`IPv4 Configuration Type`** para `Static IPv4` com o endereço **`192.168.0.1/24`**.

11. Crie a regra de firewall para o acesso de gerenciamento

Para garantir que a comunicação do seu computador host (com o IP `172.16.32.3`, que pode mudar) chegue ao pfSense, crie uma regra de permissão.

1. Vá em **`Firewall > Rules > OPT1`**.

2. Crie uma regra `Pass` no topo da lista:
   
   * **Action:** `Pass`
   * **Interface:** `OPT1`
   * **Protocol:** `TCP`
   * **Source:** **`Network`** com o endereço **`172.16.32.0/23`**. Isso permite acesso de qualquer IP dentro dessa sub-rede.
   * **Destination:** `OPT1 address`.
   * **Destination Port Range:** `HTTP` e `HTTPS`.

3. Desative as interfaces LAN e WAN para a ponte transparente

Agora que o acesso de gerenciamento está seguro, você pode desativar as interfaces WAN e LAN.

1. Vá em **`Services > DHCP Server > LAN`** e desabilite o DHCP.

2. **Em `Interfaces > WAN`:**
   
   * Defina **IPv4 e IPv6** como **`None`**.
   * Desmarque **`Block bogon Networks`** e **`Block private networks`**.

3. **Em `Interfaces > LAN`:**
   
   * Defina **IPv4 e IPv6** como **`None`**.

4. **Vá em `Interfaces > Bridges`** e crie a ponte `BRIDGE0` unindo as interfaces `em0` (WAN) e `em1` (LAN).

5. Finalize a configuração e teste

6. **Salve e aplique** todas as mudanças.

7. Se a sua máquina host e as VMs clientes estiverem na mesma sub-rede, o acesso ao WebGUI deverá ser feito pelo IP da `vboxnet0` (por exemplo, `https://192.168.0.1`).

8. **Verifique nas VMs clientes** se o DHCP do Fortinet está funcionando e se o acesso à internet redireciona para a página de login do Fortinet.

Por que a abordagem de ponte + captive portal não é ideal neste caso

Você mencionou que o Fortinet já controla a página de login, e a documentação do pfSense explica que não é possível combinar uma ponte transparente com o portal cativo na mesma interface. O pfSense precisaria ter o endereço IP da interface de bridge para servir a página de login, o que anula o propósito da ponte transparente. 

Sua melhor estratégia é usar a ponte transparente para que os dispositivos recebam IPs do Fortinet e, consequentemente, sejam gerenciados por ele.

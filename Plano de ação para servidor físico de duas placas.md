Plano de ação para servidor físico de duas placas

    Acesse o console físico: Conecte um monitor e teclado diretamente ao servidor pfSense. Essa é a sua única garantia de acesso ao WebGUI, caso perca a conexão.
    Restaure a LAN temporariamente: No console, redefina a LAN para 10.0.0.1 para restaurar o acesso ao WebGUI via navegador.
    Habilite a filtragem na ponte: No WebGUI, vá para System > Advanced > System Tunables e ajuste os valores:
        net.link.bridge.pfil_member para 0.
        net.link.bridge.pfil_bridge para 1.
    Crie a interface de ponte (OPT1):
        Vá para Interfaces > Assignments > Bridges e crie a ponte (BRIDGE0) com as interfaces em0 (WAN) e em1 (LAN).
        Em Interfaces > Assignments, atribua a BRIDGE0 a uma nova interface, por exemplo, OPT1.
    Configure o IP da OPT1:
        Vá para Interfaces > OPT1, habilite-a e atribua o endereço 10.0.0.1/24.
        Observação: O IP de gerenciamento deve estar na mesma sub-rede da sua rede interna, 10.0.0.0/24, e não deve ter conflito com nenhum outro dispositivo.
    Crie a regra de acesso de gerenciamento:
        Vá para Firewall > Rules > OPT1.
        Crie uma regra Pass para a porta 443 (e 80, se necessário). O Destination deve ser OPT1 address.
    Desative o DHCP da LAN: Em Services > DHCP Server > LAN, desmarque a opção "Enable".
    Desative os IPs das interfaces físicas:
        Vá para Interfaces > LAN e Interfaces > WAN.
        Defina o IPv4 Configuration Type e IPv6 Configuration Type para None em ambas.
        Salve e aplique as mudanças.
    Remova as regras de LAN e WAN: Vá para Firewall > Rules e exclua as regras nas abas LAN e WAN.
    Teste o acesso: Agora, você deve conseguir acessar o WebGUI pelo IP da interface de ponte (10.0.0.1). 

Importante: A conexão com o pfSense é feita através da rede interna 10.0.0.0. O tráfego dos clientes passa pela ponte, e você se conecta ao pfSense usando o endereço de gerenciamento da interface OPT1. A autenticação no Fortinet permanece inalterada, pois o pfSense age de forma transparente.

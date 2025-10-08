Com certeza! É totalmente possível organizar as informações do arquivo em uma documentação coesa com um guia passo a passo.

O problema central identificado é o **NAT (Network Address Translation)** do **pfSense** que está "escondendo" os IPs reais dos usuários, fazendo com que o **Fortinet** veja apenas o IP da interface WAN do pfSense. Isso faz com que a autenticação individual no captive portal do Fortinet libere a internet para todos.

A documentação a seguir organiza as soluções e detalha o passo a passo para a solução mais indicada ao seu objetivo (manter o login individual no Fortinet).

* * *

📄 Documentação: Configuração pfSense para Autenticação Fortinet Individual
---------------------------------------------------------------------------

### 📌 1. Análise da Situação e Soluções (Organização de Informação)

#### Problema Fundamental

O **pfSense** está configurado para fazer NAT de Saída (Outbound NAT ou masquerade) para as redes internas (ex: `10.4.x.x`).

* **Comportamento:** Todos os PCs da LAN saem para a WAN (onde está o Fortinet) com o **mesmo IP** (o IP WAN do pfSense).

* **Consequência:** Quando um usuário se autentica no portal do Fortinet, o Fortinet associa esse login ao **IP único do pfSense**, liberando o acesso para todos os demais usuários.

#### Soluções Viáveis (Consolidadas)

| Opção                                           | Descrição                                                                                                                                                         | Vantagens                                                                                                 | Desvantagens                                                                       |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **1. Desabilitar NAT no pfSense (Recomendado)** | Configurar o pfSense para agir como um roteador puro, enviando os IPs reais dos clientes (`10.x.x.x`) diretamente ao Fortinet.                                    | Solução **mais correta** para forçar o login individual no Fortinet. O Fortinet enxerga cada dispositivo. | Requer configuração de **Rotas Estáticas** no Fortinet para as redes internas.     |
| **2. Fortinet Single Sign-On (FSSO)**           | Integração direta do Fortinet com o Active Directory (AD) para autenticação transparente (quando o usuário loga no Windows, o acesso é liberado automaticamente). | É a solução mais limpa e transparente para o usuário.                                                     | Não atende ao seu objetivo de **manter o portal de login individual do Fortinet**. |
| **3. Remover pfSense da Autenticação**          | Deixar o Fortinet como firewall principal e o pfSense apenas para roteamento/serviços internos.                                                                   | Evita o problema do NAT e simplifica a camada de segurança.                                               | Exige uma reestruturação da rede e da função de cada firewall.                     |

* * *

🛠️ 2. Passo a Passo de Implementação: Desabilitar NAT no pfSense
-----------------------------------------------------------------

O objetivo é criar regras de **"Não fazer NAT" (No NAT)** no pfSense para as suas redes internas, garantindo que o Fortinet veja o IP real de cada cliente.

### A. Preparação no pfSense

1. **Acesso:** Acesse a interface web do pfSense.

2. **Mudar o Modo NAT:**
   
   * Vá em **Firewall > NAT**.
   
   * Clique na aba **Saída (Outbound)**.
   
   * Selecione a opção **Manual Outbound NAT rule generation (AON - Advanced Outbound NAT)**.
   
   * Clique em **Save** e depois em **Apply Changes**.

3. **Análise das Regras:** Ao mudar para o modo manual, as regras automáticas serão copiadas. Você deve **apagar ou desativar** as regras que fazem "masquerade" (tradução) da sua rede LAN para o **WAN address**.

### B. Criação da Regra "No NAT"

Você deve criar uma regra de **"Não faça NAT"** para cada uma de suas faixas de rede interna (ex: `10.4.0.0/23`, `10.0.0.0/24`, `10.3.0.0/24`, etc.).

1. Na aba **Outbound**, clique em **Adicionar** (ícone `+` ou seta verde).

2. Preencha os campos da seguinte forma:

| Campo                            | Valor                            | Observações                                                                |
| -------------------------------- | -------------------------------- | -------------------------------------------------------------------------- |
| **Não faça NAT**                 | **✅ Marcado**                    | **Crucial.** Impede que o pfSense altere o IP de origem.                   |
| **Interface**                    | **WAN**                          | A interface de saída para o Fortinet.                                      |
| **Protocolo**                    | **Any**                          | Para todo o tráfego.                                                       |
| **Fonte (Source) / Tipo**        | Rede ou LAN subnets              | Idealmente **Network or Alias** ou **LAN subnets**.                        |
| **Fonte (Source) / Rede**        | Ex: **`10.4.0.0/23`**            | Repita a regra para cada sub-rede interna que precisa de login individual. |
| **Destino (Destination) / Tipo** | Network or Alias                 | Resolve o erro de sintaxe.                                                 |
| **Destino (Destination) / Rede** | **`0.0.0.0`** e **Máscara: `0`** | Representa **"Qualquer Rede" (`0.0.0.0/0`)** e evita erros de validação.   |
| **Tradução / Endereço**          | **None**                         | Confirma a ação "Não fazer NAT".                                           |

3. Clique em **Save** e depois em **Apply Changes**.

### C. Ordem das Regras (Extremamente Importante)

No modo Manual, a ordem é processada de cima para baixo.

1. Vá para a lista principal de regras **Outbound NAT**.

2. **Arraste sua nova regra "No NAT"** para o **topo da lista** (acima de qualquer regra que ainda faça NAT para o "WAN Address").

3. Clique em **Save** e **Apply Changes** novamente para garantir que a ordem seja aplicada.

* * *

⚠️ 3. Solução de Problemas e Pós-Configuração
---------------------------------------------

### A. Fortinet (Rotas Estáticas)

Após o pfSense parar de mascarar os IPs, o Fortinet começará a receber pacotes com IPs de origem privados (Ex: `10.4.0.33`).

* **Ação:** O Fortinet precisa saber para onde enviar a resposta para esses IPs.

* **Configuração:** No Fortinet, crie **Rotas Estáticas** para cada uma das suas redes internas.
  
  * **Exemplo de Rota:** Rota para a rede `10.4.0.0/23` via o **IP da interface LAN/interna do pfSense** (que é o gateway para essas redes).

### B. Logon de Rede (Active Directory)

A alteração do NAT no pfSense não deve afetar diretamente o servidor de autenticação do Windows (AD, geralmente `10.4.0.9`).

* **Verificação:** Certifique-se de que os clientes de rede continuem a conseguir se comunicar com o servidor AD e que o **DNS** dos clientes aponte para o servidor AD.

### C. Se o Login no Fortinet ainda falhar

Se o problema persistir:

* **Verifique a Ordem:** Certifique-se de que sua regra **"Não faça NAT"** esteja no **topo da lista** na aba Outbound NAT do pfSense.

* **Verifique as Rotas:** Use o Fortinet para garantir que ele tenha a rota correta para as redes `10.x.x.x` (o Fortinet deve ver o pfSense como o gateway para estas redes).

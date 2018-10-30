---
title: Usar o Kerberos no gateway local para SSO (logon único) do Power BI para fontes de dados locais
description: Configure seu gateway com o Kerberos para habilitar o SSO do Power BI para fontes de dados locais
author: mgblythe
ms.author: mblythe
manager: kfile
ms.reviewer: ''
ms.service: powerbi
ms.component: powerbi-gateways
ms.topic: conceptual
ms.date: 10/10/2018
LocalizationGroup: Gateways
ms.openlocfilehash: 0055994ee883fbdb508dfa304d063bc359dd5beb
ms.sourcegitcommit: a764e4b9d06b50d9b6173d0fbb7555e3babe6351
ms.translationtype: HT
ms.contentlocale: pt-BR
ms.lasthandoff: 10/22/2018
ms.locfileid: "49641610"
---
# <a name="use-kerberos-for-single-sign-on-sso-from-power-bi-to-on-premises-data-sources"></a>Use o Kerberos para logon único (SSO) do Power BI para fontes de dados locais

Use [delegação restrita de Kerberos](https://technet.microsoft.com/library/jj553400.aspx) para habilitar a conectividade ininterrupta de logon único. Habilitar SSO torna mais fácil para relatórios e dashboards do Power BI atualizar os dados de fontes locais.

## <a name="supported-data-sources"></a>Fontes de dados para as quais há suporte

No momento, damos suporte para as seguintes fontes de dados:

* SQL Server
* SAP HANA
* Teradata
* Spark

Também damos suporte a SAP HANA com [SAML (Security Assertion Markup Language)](service-gateway-sso-saml.md).

### <a name="sap-hana"></a>SAP HANA

Para habilitar SSO para o SAP HANA, siga estas etapas primeiro:

* Verifique se o servidor do SAP HANA está executando a versão mínima necessária, que depende do nível de plataforma de servidor do SAP HANA:
  * [HANA 2 SPS 01 Rev 012.03](https://launchpad.support.sap.com/#/notes/2557386)
  * [HANA 2 SPS 02 Rev 22](https://launchpad.support.sap.com/#/notes/2547324)
  * [HANA 1 SP 12 Rev 122.13](https://launchpad.support.sap.com/#/notes/2528439)
* No computador do gateway, instale o driver ODBC do HANA mais recente do SAP.  A versão mínima é o HANA ODBC versão 2.00.020.00 de agosto de 2017.

Para saber mais sobre como configurar o logon único para SAP HANA usando o Kerberos, veja o tópico [Logon único usando Kerberos](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/1885fad82df943c2a1974f5da0eed66d.html) no Guia de Segurança do SAP HANA e os links da página, particularmente a Nota SAP 1837331 – COMO HANA DBSSO Kerberos/Active Directory].

## <a name="preparing-for-kerberos-constrained-delegation"></a>Preparando-se para a delegação restrita de Kerberos

Vários itens devem ser configurados para que a delegação restrita de Kerberos funcione corretamente, incluindo os *SPNs* (nomes das entidades de serviço) e as configurações de delegação nas contas de serviço.

### <a name="prerequisite-1-install--configure-the-on-premises-data-gateway"></a>Pré-requisito 1: instalar e configurar o gateway de dados local

Essa versão de gateway de dados local é compatível com atualização in-loco, bem como com o controle das configurações de gateway existentes.

### <a name="prerequisite-2-run-the-gateway-windows-service-as-a-domain-account"></a>Pré-requisito 2: executar o serviço Windows do gateway como uma conta de domínio

Em uma instalação padrão, o gateway é executado como uma conta de serviço de computador local (especificamente, *NT Service\PBIEgwService*), conforme mostrado na imagem a seguir:

![Conta de serviço](media/service-gateway-sso-kerberos/service-account.png)

Para habilitar a **delegação restrita de Kerberos**, o gateway deve ser executado como uma conta de domínio, a menos que o Azure AD já esteja sincronizado com o Active Directory local (usando o DirSync/Connect do Azure AD). Se você precisar mudar a conta para uma conta de domínio, confira [mudando o gateway para uma conta de domínio](#switching-the-gateway-to-a-domain-account) mais adiante neste artigo.

> [!NOTE]
> Se o Azure AD DirSync/Connect estiver configurado e as contas de usuário estiverem sincronizadas, o serviço do gateway não precisará executar pesquisas no AD local no tempo de execução e você poderá usar o SID de Serviço local (em vez de uma conta de domínio) para o serviço do gateway. As etapas de configuração da delegação restrita de Kerberos descritas neste artigo são as mesmas da configuração (elas são simplesmente aplicadas com base na SID do serviço, em vez da conta de domínio).

### <a name="prerequisite-3-have-domain-admin-rights-to-configure-spns-setspn-and-kerberos-constrained-delegation-settings"></a>Pré-requisito 3: ter direitos de administrador de domínio para configurar definições de SPNs (SetSPN) e da delegação restrita de Kerberos

Embora seja tecnicamente possível que um administrador de domínio conceda direitos temporários ou permanentes para alguém configurar a delegação de Kerberos e os SPNs sem exigir direitos de administrador de domínio, essa não é a abordagem recomendada. Na seção a seguir, cobrimos as etapas de configuração necessárias para o **Pré-requisito 3** em detalhes.

## <a name="configuring-kerberos-constrained-delegation-for-the-gateway-and-data-source"></a>Configurando a delegação restrita de Kerberos para o gateway e a fonte de dados

Para configurar o sistema corretamente, é preciso configurar ou validar os dois itens a seguir:

1. Se necessário, configure um SPN para a conta de domínio do serviço de gateway.

2. Defina as configurações de delegação na conta de domínio do serviço do gateway.

Observe que é preciso ser administrador de domínio para executar as duas etapas de configuração.

As seções a seguir descrevem essas etapas individualmente.

### <a name="configure-an-spn-for-the-gateway-service-account"></a>Configurar um SPN para a conta de serviço do gateway

Primeiro, determine se um SPN já foi criado para a conta de domínio usada como a conta de serviço do gateway, mas seguindo estas etapas:

1. Como administrador de domínio, inicialize **Usuários e Computadores do Active Directory**.

2. Clique com o botão direito do mouse no domínio, selecione **Localizar** e digite o nome da conta de serviço do gateway

3. No resultado da pesquisa, clique com o botão direito do mouse na conta de serviço do gateway e selecione **Propriedades**.

4. Se a guia **Delegação** estiver visível na caixa de diálogo **Propriedades**, será a indicação que um SPN já foi criado e que você poderá pular para a próxima subseção sobre como definir as configurações de Delegação.

    Se não houver nenhuma guia **Delegação** na caixa de diálogo **Propriedades**, você poderá criar manualmente um SPN nessa conta para adicionar a guia **Delegação** (que é a maneira mais fácil de definir as configurações de delegação). A criação de um SPN pode ser executada usando a [ferramenta setspn](https://technet.microsoft.com/library/cc731241.aspx) fornecida com o Windows (você precisa de direitos de administrador de domínio para criar o SPN).

    Por exemplo, imagine que a conta de serviço do gateway seja “PBIEgwTest\GatewaySvc” e o nome do computador com o serviço do gateway em execução tenha o nome de **Machine1**. Para definir o SPN para a conta de serviço do gateway desse computador do exemplo, você precisará executar o seguinte comando:

    ![Definir SPN](media/service-gateway-sso-kerberos/set-spn.png)

    Com a etapa concluída, podemos prosseguir para as configurações de delegação.

### <a name="configure-delegation-settings-on-the-gateway-service-account"></a>Definir as configurações de delegação na conta de serviço do gateway

O segundo requisito de configuração são as configurações de delegação na conta de serviço do gateway. Há diversas ferramentas que podem ser usadas para realizar essas etapas. Neste artigo, usaremos os **Usuários e Computadores do Active Directory**, que é um snap-in do MMC (Console de Gerenciamento Microsoft) que você pode usar para administrar e publicar informações no diretório. Ele está disponível nos controladores de domínio por padrão. Você também pode habilitá-lo pela configuração do **Recurso do Windows** em outros computadores.

Precisamos configurar a **delegação restrita de Kerberos** com o trânsito de protocolos. Com a delegação restrita, é preciso ser explícito sobre para quais serviços você deseja delegar. Por exemplo, somente seu SQL Server ou seu servidor SAP HANA aceitará chamadas de delegação da conta de serviço de gateway.

Esta seção presume que você já configurou SPNs para as fontes de dados subjacentes (como SQL Server, SAP HANA, Teradata, Spark e assim por diante). Para saber como configurar os SPNs do servidor de fonte de dados, confira a documentação técnica do respectivo servidor de banco de dados. Você também pode conferir a postagem no blog que descreve [*Qual SPN seu aplicativo exige?*](https://blogs.msdn.microsoft.com/psssql/2010/06/23/my-kerberos-checklist/)

Nas etapas a seguir, presumimos um ambiente local com dois computadores: um computador do gateway e um servidor de banco de dados executando o SQL Server. Para este exemplo, vamos também supor as configurações e os nomes a seguir:

* Nome do computador do gateway: **PBIEgwTestGW**
* Conta de serviço do gateway: **PBIEgwTest\GatewaySvc** (nome de exibição da conta: Conector do Gateway)
* Nome do computador da fonte de dados do SQL Server: **PBIEgwTestSQL**
* Conta de serviço da fonte de dados do SQL Server: **PBIEgwTest\SQLService**

Considerando os nomes e as configurações do exemplo, as etapas de configuração serão as seguintes:

1. Com direitos de administrador de domínio, inicie os **Usuários e Computadores do Active Directory**.

2. Clique com o botão direito do mouse na conta de serviço do gateway (**PBIEgwTest\GatewaySvc**) e selecione **Propriedades**.

3. Selecione a guia **Delegação**.

4. Selecione **Confiar no computador para delegação apenas a serviços especificados.**

5. Selecione **Usar qualquer protocolo de autenticação.**

6. Em **Serviços aos quais esta conta pode apresentar credenciais delegadas**, selecione **Adicionar**.

7. Na nova caixa de diálogo, selecione **Usuários ou Computadores**.

8. Insira a conta de serviço do Banco de Dados do SQL Server (**PBIEgwTest\SQLService**) e selecione **OK**.

9. Selecione o SPN que você criou para o servidor de banco de dados. Em nosso exemplo, o SPN começará com **MSSQLSvc**. Se você tiver adicionado o SPN NetBIOS e o FQDN para o serviço de banco de dados, selecione ambos. Você pode ver apenas um.

10. Selecione **OK**. Agora você verá o SPN na lista.

11. Como opção, é possível selecionar **Expandido** para mostrar o SPN do NetBIOS e o FQDN.

12. A caixa de diálogo será semelhante à seguinte se você selecionar **Expandido**. Selecione **OK**.

    ![Propriedades do conector de gateway](media/service-gateway-sso-kerberos/gateway-connector-properties.png)

Por fim, no computador que executa o serviço do gateway (**PBIEgwTestGW** em nosso exemplo), a conta de serviço do gateway deverá receber a política local “Representar um cliente após autenticação”. Você poderá executar/verificar isso com o Editor de Política de Grupo Local (**gpedit**).

1. No computador do gateway, execute: *gpedit.msc*.

1. Navegue até **Política de Computador Local > Configuração do Computador > Configurações do Windows > Configurações de Segurança > Políticas Locais > Atribuição de Direitos de Usuário**, conforme mostrado na imagem a seguir.

    ![Atribuição de direitos do usuário](media/service-gateway-sso-kerberos/user-rights-assignment.png)

1. Na lista de políticas em **Atribuição de Direitos de Usuário**, selecione **Representar um cliente após autenticação**.

    ![Representar um cliente](media/service-gateway-sso-kerberos/impersonate-client.png)

    Clique com o botão direito do mouse e abra as **Propriedades** para **Representar um cliente após autenticação** e verifique a lista de contas. A conta de serviço do gateway (**PBIEgwTest\GatewaySvc**) deverá estar incluída nela.

1. Na lista de políticas em **Atribuição de Direitos de Usuário**, selecione **Atuar como parte do sistema operacional (SeTcbPrivilege)**. Verifique se a conta de serviço do gateway também está incluída na lista de contas.

18. Reinicie o processo do serviço do **gateway de dados local**.

Se você está usando o SAP HANA, recomendamos seguir estas etapas adicionais, que podem produzir uma pequena melhoria de desempenho.

1. No diretório de instalação do gateway, localize e abra este arquivo de configuração: *Microsoft.PowerBI.DataMovement.Pipeline.GatewayCore.dll.config*.

1. Localize a propriedade *FullDomainResolutionEnabled* e altere seu valor para *True*.

    ```xml
    <setting name=" FullDomainResolutionEnabled " serializeAs="String">
          <value>True</value>
    </setting>
    ```

## <a name="running-a-power-bi-report"></a>Executando um relatório do Power BI

Depois de concluir todas as etapas de configuração descritas anteriormente neste artigo, você pode usar a página **Gerenciar Gateway** no Power BI para configurar a fonte de dados. Em seguida, em **Configurações Avançadas**, habilite SSO e publique relatórios e conjuntos de dados associados a essa fonte de dados.

![Configurações avançadas](media/service-gateway-sso-kerberos/advanced-settings.png)

Essa configuração funcionará na maioria dos casos. No entanto, dependendo do ambiente, pode haver configurações diferentes com o Kerberos. Se ainda não for possível carregar o relatório, você precisará contatar o administrador de domínio para investigar o caso.

## <a name="switching-the-gateway-to-a-domain-account"></a>Mudando o gateway para uma conta de domínio

Neste artigo, discutimos como mudar o gateway de uma conta de serviço local para ser executada como uma conta de domínio usando a interface do usuário do **Gateway de dados local**. Veja abaixo as etapas necessárias para fazer isso.

1. Inicie a ferramenta de configuração do **gateway de dados local**.

   ![Aplicativo da área de trabalho do gateway](media/service-gateway-sso-kerberos/gateway-desktop-app.png)

2. Selecione o botão **Entrar** na página principal e entre usando sua conta do Power BI.

3. Em seguida, selecione a guia **Configurações do Serviço**.

4. Selecione em **Alterar conta** para iniciar as instruções passo a passo, conforme mostrado na imagem a seguir.

   ![Alterar conta](media/service-gateway-sso-kerberos/change-account.png)

## <a name="configuring-sap-bw-for-sso"></a>Configurar SAP BW para SSO

Agora que você entende como o Kerberos funciona com um gateway, pode configurar o SSO para seu SAP BW (SAP Business Warehouse). As etapas a seguir pressupõem que você já [se preparou para a delegação restrita do Kerberos](#preparing-for-kerberos-constrained-delegation), conforme descrito anteriormente neste artigo.

### <a name="install-sap-bw-components"></a>Instalar componentes do SAP BW

Se você não tiver configurado o SAP gsskrb5 e gx64krb5 em seus computadores cliente e o Servidor de Aplicativos SAP BW, conclua esta seção. Se você já tiver concluído essa configuração (você criou um usuário de serviço para seu servidor BW e mapeou um SPN para ele), você poderá ignorar algumas partes desta seção.

1. Baixe gsskrb5/gx64krb5 da [Nota da SAP 2115486](https://launchpad.support.sap.com/) (usuário s da SAP necessário). Verifique se você tem pelo menos a versão 1.0.11.x de gsskrb5.dll e gx64krb5.dll.

1. Coloque a biblioteca em um local no computador do gateway que seja acessível por sua instância do gateway (e também pela SAP GUI, se você quiser testar a conexão SSO usando SAP GUI/Logon).

1. Coloque outra cópia em seu computador de servidor BW em um local acessível pelo servidor BW.

1. Nos computadores cliente e servidor, defina as variáveis de ambiente SNC\_LIB e SNC\_LIB\_64 para apontarem para os locais de gsskrb5 e gx64krb5.dll, respectivamente.

### <a name="complete-the-gateway-configuration-for-sap-bw"></a>Concluir a configuração de gateway para SAP BW

Além da configuração de gateway que você já fez, há algumas etapas adicionais específicas do SAP BW. A seção [**Definir configurações de delegação na conta de serviço do gateway**](#configure-delegation-settings-on-the-gateway-service-account) da documentação pressupõe que você já configurou SPNs para as fontes de dados subjacentes. Para concluir essa configuração para SAP BW:

1. Em um Controlador de Domínio do Active Directory, crie um Usuário de Serviço (inicialmente, apenas um usuário do Active Directory simples) para o Servidor de Aplicativos do BW no ambiente do Active Directory. Em seguida, atribua um SPN a ele.

    O SPN atribuído **deve** iniciar com SAP/. O que vem depois do SAP/cabe a você; uma opção é usar o nome de usuário do Usuário de Serviço do servidor do BW. Por exemplo, se você criar BWServiceUser@\<DOMAIN\> como o Usuário de Serviço, poderá usar o SPN SAP/BWServiceUser. Uma maneira de definir o mapeamento de SPN é o comando setspn. Por exemplo, para definir o SPN no usuário de serviço que acabamos de criar, você executaria o seguinte comando em uma janela cmd em um computador do Controlador de Domínio: `setspn -s SAP/ BWServiceUser DOMAIN\ BWServiceUser`.

1. Conceda acesso de Usuário doe Serviço para sua instância de Servidor de Aplicativos do BW:

    1. No computador do servidor do BW, adicione o Usuário do Serviço ao grupo Administrador Local para seu servidor do BW: abra o programa de Gerenciamento de Computador e clique duas vezes no grupo Administrador Local para seu servidor.

        ![Gerenciamento do computador](media/service-gateway-sso-kerberos/computer-management.png)

    1. Clique duas vezes no grupo Administrador Local e, em seguida, selecione **Adicionar** para adicionar o Usuário de Serviço do BW ao grupo. Use o botão **Verificar Nomes** para garantir que você tenha digitado o nome corretamente. Selecione **OK**.

1. Defina o Usuário de Serviço do Servidor do BW como o usuário que inicia o serviço do Servidor do BW no computador do servidor do BW.

    1. Abra o programa "Executar" e digite "Services.msc". Procure o serviço correspondente à instância do Servidor de Aplicativos do BW. Clique com o botão direito do mouse e selecione **Propriedades**.

        ![Propriedades do servidor](media/service-gateway-sso-kerberos/server-properties.png)

    1. Alterne para a guia **Logon** e mude o usuário para o Usuário de Serviço do BW, conforme especificado acima. Insira a senha do usuário e selecione **OK**.

1. Entre no seu servidor em SAP GUI/Logon e defina os seguintes parâmetros de perfil usando a transação RZ10:

    1. Defina o parâmetro de perfil snc/identity/as como p:\<o usuário de serviço do BW que você criou\>, como p:BWServiceUser@MYDOMAIN.COM. Observe o p: que precede o UPN do usuário do serviço.

    1. Defina o parâmetro de perfil snc/gssapi\_lib como \<caminho para gsskrb5.dll/gx64krb5.dll no computador do servidor (a biblioteca que você usará depende do número de bits do SO)\>. Lembre-se de colocar a biblioteca em um local que o Servidor de Aplicativos do BW possa acessar.

    1. Também defina os seguintes parâmetros de perfil adicionais, alterando os valores conforme necessário para atender às suas necessidades. Observe que as últimas cinco opções de permitem que os clientes conectem-se ao servidor do BW usando SAP Logon/GUI sem necessidade de ter SNC configurado.

        | **Configuração** | **Valor** |
        | --- | --- |
        | snc/data\_protection/max | 3 |
        | snc/data\_protection/min | 1 |
        | snc/data\_protection/use | 9 |
        | snc/accept\_insecure\_cpic | 1 |
        | snc/accept\_insecure\_gui | 1 |
        | snc/accept\_insecure\_r3int\_rfc | 1 |
        | snc/accept\_insecure\_rfc | 1 |
        | snc/permit\_insecure\_start | 1 |

    1. Defina a propriedade snc/enable como 1.

1. Depois de definir esses parâmetros de perfil, abra o Console de Gerenciamento SAP no computador do servidor e reinicie a instância do BW. Se o servidor não iniciar, verifique se você definiu corretamente os parâmetros de perfil. Para obter mais informações sobre configurações de perfil de parâmetro, confira a [documentação da SAP](https://help.sap.com/saphelp_nw70ehp1/helpdata/en/e6/56f466e99a11d1a5b00000e835363f/frameset.htm). Você também poderá consultar a seção sobre informações de solução de problemas mais adiante nesta seção se encontrar problemas.

### <a name="map-azure-ad-users-to-sap-bw-users"></a>Mapear os usuários do Azure AD para usuários do SAP BW

Mapeie um usuário do Active Directory para um usuário do servidor de aplicativos do SAP BW e teste a conexão de SSO em SAP GUI/Logon.

1. Entre no seu servidor BW usando SAP GUI/Logon. Execute a transação SU01.

1. Para **Usuário**, insira o usuário do BW para o qual você deseja habilitar conexões por SSO (na captura de tela acima, estamos definindo permissões para BIUSER). Selecione o ícone **Editar** perto do canto superior esquerdo da janela Logon do SAP (imagem de uma caneta).

    ![Manutenção do usuário](media/service-gateway-sso-kerberos/user-maintenance.png)

1. Selecione a guia **SNC**. Na caixa de entrada de nome SNC, insira p:\<seu usuário do Active Directory\>@\<seu domínio\>. Observe o p: obrigatório que deve preceder o UPN do usuário do Active Directory. O usuário do Active Directory que você especificar deverá pertencer à pessoa ou à organização para a qual você deseja habilitar o acesso SSO para o servidor de aplicativos do BW. Por exemplo, se você quiser habilitar o acesso SSO para o usuário [testuser@TESTDOMAIN.COM](mailto:testuser@TESTDOMAIN.COM), insira p:testuser@TESTDOMAIN.COM.

    ![Manter usuários](media/service-gateway-sso-kerberos/maintain-users.png)

1. Selecione o ícone Salvar (o disquete perto do canto superior esquerdo da tela).

### <a name="verify-sign-in-using-sso"></a>Verifique a conexão usando SSO

Verifique se você pode entrar no servidor usando o Logon da SAP/SAP GUI por meio de SSO como o usuário do Active Directory para o qual você acaba de habilitar acesso por SSO.

1. Entre um computador em que o Logon da SAP esteja instalado *como o usuário do Active Directory para o qual você acabou de habilitar o acesso por SSO* e inicialize SAP GUI/Logon. Criar uma nova conexão.

1. Na janela **Criar Nova Entrada do Sistema**, selecione **Sistema Especificado pelo Usuário** e selecione **Avançar**.

    ![Nova entrada do sistema](media/service-gateway-sso-kerberos/new-system-entry.png)

1. Preencha os detalhes apropriados na próxima página, incluindo o servidor de aplicativos, o número de instância e a ID do sistema, então selecione **Concluir**.

1. Clique com o botão direito do mouse na nova conexão e selecione **Propriedades**. Selecione a guia **Rede**. Na janela **Nome SNC**, insira p:\<o UPN do usuário de serviço do BW\>, como p:BWServiceUser@MYDOMAIN.COM.

    ![Propriedades de entrada do sistema](media/service-gateway-sso-kerberos/system-entry-properties.png)

1. Selecione **OK**. Agora, clique duas vezes a conexão que você acabou de criar para tentar uma conexão por SSO com o serviço. Se essa conexão for bem-sucedida, vá para a próxima etapa. Caso contrário, examine as etapas anteriores neste documento para garantir que foram concluídas corretamente, ou examine a seção de solução de problemas abaixo. Observe que, se você não conseguir se conectar ao servidor BW por meio de SSO nesse contexto, não poderão se conectar ao servidor BW usando SSO no contexto do gateway.

### <a name="troubleshoot-installation-and-connections"></a>Solucionar problemas de instalação e conexões

Se você tiver algum problema, siga estas etapas para solucionar problemas de instalação gsskrb5 e conexões de SSO do SAP GUI/Logon.

1. Exibir logs do servidor (...work\dev\_w0 no computador do servidor) poderá ser útil para solucionar problemas de erros encontrados ao concluir as etapas de configuração de gsskrb5, especialmente se o servidor BW não iniciar depois da alteração dos parâmetros de perfil.

1. Se você não puder iniciar o serviço BW devido a uma "falha de logon", poderá ter informado a senha incorreta ao definir o usuário "Iniciar como" do BW. Verifique a senha fazendo logon em um computador no ambiente do Active Directory como o usuário do serviço BW.

1. Se você receber erros sobre credenciais SQL impedindo que o servidor inicie, verifique se você concedeu acesso de Usuário do Serviço ao banco de dados do BW.

1. "(GSS-API)O destino especificado é desconhecido ou está inacessível": isso geralmente significa que você tem o Nome SNC errado especificado. Use "p", não "p:CN =" ou qualquer outro valor no aplicativo cliente que não seja o UPN do Usuário do Serviço.

1. "(GSS-API) Um nome inválido foi fornecido": verifique se "p:" está no valor do parâmetro de perfil de identidade SNC do servidor.

1. "(Erro SNC) O módulo especificado não pôde ser encontrado": isso geralmente é causado colocando gsskrb5.dll/gx64krb5.dll em algum lugar que exija privilégios elevados (direitos de administrador) para acesso.

### <a name="add-registry-entries"></a>Adicionar entradas de Registro

Adicione entradas de Registro necessárias para o Registro do computador em que o gateway está instalado. Em seguida, defina os parâmetros de configuração de gateway necessários.

1. Execute os seguintes comandos em uma janela cmd:

    1. REG ADD HKLM\SOFTWARE\Wow6432Node\SAP\gsskrb5 /v ForceIniCredOK /t REG\_DWORD /d 1 /f

    1. REG ADD HKLM\SOFTWARE\SAP\gsskrb5 /v ForceIniCredOK /t REG\_DWORD /d 1 /f

1. Abra o arquivo de configuração de gateway principal, Microsoft.PowerBI.DataMovement.Pipeline.GatewayCore.dll. Por padrão, esse arquivo é armazenado em C:\Arquivos de Programas\On-premises data gateway.

1. Defina **ADUserNameLookupProperty** como msDS-cloudExtensionAttribute1 e **ADUserNameReplacementProperty** como SAMAccountName. Salve o arquivo de configuração.

1. Reinicie o serviço de Gateway por meio da guia **Serviços** do Gerenciador de Tarefas (clique com o botão direito do mouse em **Reiniciar**).

    ![Reiniciar gateway](media/service-gateway-sso-kerberos/restart-gateway.png)

### <a name="set-azure-ad-properties"></a>Propriedades do conjunto do Azure AD

Defina a propriedade msDS-cloudExtensionAttribute1 do usuário do Active Directory mapeado para um usuário do BW (na etapa "Mapear usuários do Azure AD para usuários do SAP BW") para o usuário de serviço do Power BI para o qual você deseja habilitar SSO do Kerberos. Uma maneira de definir a propriedade msDS-cloudExtensionAttribute1 é usando o snap-in MMC de Computadores e Usuários do Active Directory (observe que outros métodos também podem ser usados).

1. Entre em um computador do Controlador de Domínio como um usuário administrador.

1. Abra a pasta **Usuários** na janela do snap-in e clique duas vezes no usuário do Active Directory mapeado para um usuário do BW.

1. Selecione a guia **Editor de Atributo**. Se você não vir esta guia, precisará pesquisar para obter instruções sobre como habilitá-la ou usar outro método para definir a propriedade msDS-cloudExtensionAttribute1. Selecione um dos atributos e, em seguida, a tecla "m" para navegar até as propriedades do Active Directory que começam com "m". Localize a propriedade msDS-cloudExtensionAttribute1 e clique duas vezes nela. Defina o valor para o nome de usuário usado para entrar e, em seguida, o Serviço do Power BI. Selecione **OK**.

    ![Editar atributo](media/service-gateway-sso-kerberos/edit-attribute.png)

1. Selecione **Aplicar**. Verifique se que o valor correto foi definido na coluna Valor.

### <a name="add-a-new-bw-application-server-data-source-to-the-power-bi-service"></a>Adicione uma nova fonte de dados do Servidor de Aplicativos do BW ao Serviço do Power BI

Adicione a fonte de dados do BW ao seu gateway: siga as instruções neste artigo sobre [como executar um relatório](#running-a-power-bi-report).

1. Na janela de configuração de fonte de dados, insira o **Nome de Host**, o **Número do Sistema** e a **ID do Cliente** do Servidor de Aplicativos como faria para entrar no seu servidor do BW no do Power BI Desktop. Como o **Método de Autenticação**, selecione **Windows**.

1. No campo **Nome do Parceiro SNC**, insira o valor armazenado no parâmetro de perfil snc/identity/as do servidor *com SAP/ adicionado entre o p: e o restante da identidade.* Por exemplo, se a identidade snc do servidor for p:BWServiceUser@MYDOMAIN.COM, você deverá inserir p:SAP/BWServiceUser@MYDOMAIN.COM. na caixa de entrada Nome do Parceiro SNC.

1. Para a Biblioteca SNC, selecione SNC\_LIB ou SNC\_LIB\_64.

1. O **Nome de usuário** e a **Senha** devem ser o nome de usuário e a senha de um usuário do Active Directory com permissão para entrar no servidor BW via SSO (um usuário do Active Directory mapeado para um usuário BW por meio a transação SU01). Essas credenciais serão usadas apenas se a caixa **Usar SSO via Kerberos para consultas DirectQuery** *não* estiver marcada.

1. Marque a caixa **Usar SSO via Kerberos para consultas do DirectQuery** e selecione **Aplicar**. Se a conexão de teste não for bem-sucedida, verifique se as etapas de instalação e de configuração anteriores foram concluídas corretamente.

### <a name="test-your-setup"></a>Testar a configuração

Publica um relatório do DirectQuery no Power BI Desktop para o serviço do Power BI para testar sua configuração. Verifique se que você está registrado no serviço do Power BI como o usuário para o qual você definir a propriedade msDS-cloudExtensionAttribute1. Se a instalação for concluída com êxito, você poderá criar um relatório com base no conjunto de dados publicado no serviço do Power BI e efetuar pull dos dados por meio de visuais no relatório.

### <a name="troubleshooting-gateway-connectivity-issues"></a>Solução de Problemas de Conectividade de Gateway

1. Verifique os logs de gateway. Abra o aplicativo de Configuração de Gateway, selecione **Diagnósticos** e selecione **Exportar Logs**. Os erros mais recentes estarão na parte inferior de qualquer arquivo de log que você examinar.

    ![Diagnóstico do gateway](media/service-gateway-sso-kerberos/gateway-diagnostics.png)

1. Ative o rastreamento de BW e examine os arquivos de log gerados. Há vários tipos diferentes de rastreamento BW disponíveis. Veja a documentação da SAP para obter mais informações.

## <a name="errors-from-an-insufficient-kerberos-configuration"></a>Erros de configuração insuficiente do Kerberos

Se o gateway e o servidor de banco de dados subjacente não forem configurados corretamente para a **delegação restrita de Kerberos**, você receberá a seguinte mensagem de erro:

![Erro ao carregar dados](media/service-gateway-sso-kerberos/load-data-error.png)

Os detalhes técnicos associados com a mensagem de erro (DM_GWPipeline_Gateway_ServerUnreachable) podem ser semelhante ao seguinte:

![Servidor inacessível](media/service-gateway-sso-kerberos/server-unreachable.png)

O resultado é que o gateway não conseguiu representar o usuário de origem corretamente e a tentativa de conexão de banco de dados falhou.

## <a name="next-steps"></a>Próximas etapas

Para obter mais informações sobre o **Gateway de dados local** e o **DirectQuery**, confira os seguintes recursos:

* [Gateway de dados local](service-gateway-onprem.md)
* [DirectQuery no Power BI](desktop-directquery-about.md)
* [Fontes de dados com suporte do DirectQuery](desktop-directquery-data-sources.md)
* [DirectQuery e SAP BW](desktop-directquery-sap-bw.md)
* [DirectQuery e SAP HANA](desktop-directquery-sap-hana.md)
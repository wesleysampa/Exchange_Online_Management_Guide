
# Como Resolver a Situação de um Grupo de Distribuição Sincronizado do Exchange On-Premises para o Exchange Online

Este artigo descreve um passo a passo detalhado sobre como gerenciar grupos de distribuição sincronizados do **Exchange On-Premises** (Local) para o **Exchange Online**, incluindo como verificar se o grupo é sincronizado, remover a sincronização e recriar o grupo diretamente no Exchange Online.

## Contexto  
Em muitas organizações que migraram para o Exchange Online, pode ocorrer uma situação em que um **grupo de distribuição** foi originalmente criado e sincronizado com o **Active Directory local (AD)**, mas o ambiente local de Exchange foi perdido ou não está mais disponível. Nesses casos, não é possível editar ou gerenciar o grupo diretamente via Exchange Online, pois ele ainda é gerido localmente.

Este tutorial ajudará a resolver esse problema, permitindo que você transfira o gerenciamento do grupo de distribuição para o Exchange Online (cloud-managed).

## Pré-requisitos  
- **Exchange Online PowerShell**: O módulo **ExchangeOnlineManagement** deve estar instalado.
- **Azure AD PowerShell** (opcional): Para validar se o grupo é sincronizado.
- **Acesso Administrativo**: Você deve ter permissões administrativas para realizar as alterações.

---

## Passo 1: Conectar-se ao Exchange Online via PowerShell  
Antes de iniciar, é necessário se conectar ao Exchange Online via PowerShell.

1. **Instalar o módulo (caso não tenha)**:
    ```powershell
    Install-Module -Name ExchangeOnlineManagement -Scope CurrentUser
    ```

2. **Conectar ao Exchange Online**:
    ```powershell
    Connect-ExchangeOnline -UserPrincipalName seu_usuario@seudominio.com
    ```

---

## Passo 2: Verificar se o grupo é sincronizado com o Exchange On-Premises  
Use o seguinte comando para verificar as propriedades do grupo de distribuição. Substitua o nome ou endereço de e-mail do grupo.

```powershell
# Substitua pelo nome ou e-mail do grupo
$Grupo = "<INSIRA_AQUI_O_NOME_OU_O_PRIMARYSMTPADDRESS_DO_GRUPO>"

Get-DistributionGroup -Identity $Grupo | Format-List Name, DisplayName, Identity, ExternalDirectoryObjectId, RecipientTypeDetails, PrimarySmtpAddress, ManagedBy, WhenCreated
```

**O que observar:**
- Se o `RecipientTypeDetails` for **MailUniversalDistributionGroup** e o campo `ExternalDirectoryObjectId` estiver presente, significa que o grupo foi sincronizado do AD local e não pode ser gerido diretamente no Exchange Online.

---

## Passo 3: Tentar adicionar um membro (opcional)  
Se você não conseguir editar o grupo, isso pode confirmar que ele é gerido pelo AD local.

```powershell
Add-DistributionGroupMember -Identity $Grupo -Member "usuario@seudominio.com"
```

Se der erro, significa que o gerenciamento do grupo não pode ser feito diretamente pelo Exchange Online.

---

## Passo 4: Verificar o grupo no Azure AD (opcional)  
Para confirmar se o grupo ainda está sincronizado, você pode verificar as propriedades no Azure AD.

1. **Instalar o módulo AzureAD (se necessário)**:
    ```powershell
    Install-Module -Name AzureAD
    ```

2. **Conectar ao Azure AD**:
    ```powershell
    Connect-AzureAD
    ```

3. **Consultar o grupo no Azure AD**:
    ```powershell
    Get-AzureADGroup -SearchString "<INSIRA_O_NOME_DO_GRUPO>"
    ```

Se o grupo estiver marcado como **DirSyncEnabled**, significa que ainda está sendo sincronizado.

---

## Passo 5: Excluir o grupo no Active Directory (se necessário)  
Se o grupo ainda existir no AD local, você precisará excluí-lo para forçar a remoção do Exchange Online.

1. **Excluir o grupo no Active Directory**:
   - Abra o **Active Directory Users and Computers (ADUC)**.
   - Encontre o grupo de distribuição e clique com o botão direito em **Excluir**.

---

## Passo 6: Forçar a sincronização com o Azure AD Connect  
Após excluir o grupo no AD, você precisa forçar a sincronização com o Azure AD Connect para refletir a exclusão no Exchange Online.

1. **No servidor do Azure AD Connect**, abra o PowerShell e execute o seguinte comando para iniciar a sincronização:

    ```powershell
    Start-ADSyncSyncCycle -PolicyType Delta
    ```

---

## Passo 7: Verificar a remoção no Exchange Online  
Após a sincronização, confirme se o grupo foi removido do Exchange Online.

```powershell
Get-DistributionGroup -Identity $Grupo
```

Se o grupo não aparecer, significa que ele foi removido com sucesso.

---

## Passo 8: Recriar o grupo de distribuição no Exchange Online  
Agora, você pode recriar o grupo de distribuição diretamente no Exchange Online.

```powershell
New-DistributionGroup -Name "Nome do Grupo" `
-DisplayName "Nome do Grupo" `
-Alias "aliasdogrupo" `
-PrimarySmtpAddress "grupo@seudominio.com" `
-ManagedBy "usuario_responsavel@seudominio.com"
```

**Adicionar membros**:
```powershell
Add-DistributionGroupMember -Identity "grupo@seudominio.com" -Member "usuario1@seudominio.com"
Add-DistributionGroupMember -Identity "grupo@seudominio.com" -Member "usuario2@seudominio.com"
```

---

## Passo 9: Validar o grupo no Exchange Online  
Agora, verifique se o grupo foi criado corretamente e se ele é gerido na nuvem.

```powershell
Get-DistributionGroup -Identity "grupo@seudominio.com" | Format-List Name, RecipientTypeDetails, ManagedBy
```

Se o `RecipientTypeDetails` for **MailUniversalDistributionGroup** e o campo **ManagedBy** mostrar o usuário correto, o grupo foi recriado e agora é gerido pelo Exchange Online.

---

## Conclusão  
Com este processo, você foi capaz de resolver a situação de um grupo de distribuição sincronizado, removendo a dependência do Exchange On-Premises e permitindo que o grupo seja gerido diretamente no Exchange Online. Agora, você pode adicionar e remover membros, editar as permissões e usar todas as funcionalidades do Exchange Online sem limitações.

---

## Referências  
- [Documentação do Exchange Online PowerShell](https://docs.microsoft.com/en-us/powershell/module/exchange/)
- [Documentação do Azure AD PowerShell](https://docs.microsoft.com/en-us/powershell/module/azuread/)

---

## Contribuições  
Se você encontrou algum erro ou tem sugestões de melhoria, sinta-se à vontade para abrir uma issue ou enviar um pull request.

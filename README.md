# active-directory-lab

# 🛡️ Active Directory Home Lab — Blue Team Edition

Laboratório prático de Active Directory construído sobre o projeto original do [Josh Madakor](https://www.youtube.com/@JoshMadakor), estendido com atividades voltadas para o dia a dia de um profissional de Blue Team.

---

## 🖥️ Ambiente

| Componente | Detalhe |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller | Windows Server 2022 |
| Clientes | Windows 10 (Client1, Client2) |
| Domínio | mydomain.com |
| Usuários criados | ~1000 (via PowerShell script) |

**Topologia de rede:**
- DC com dois adaptadores de rede (NAT + Internal Network)
- Clientes na Internal Network, com acesso à internet roteado pelo DC via RAS/NAT
- DHCP configurado no DC

---

## 📁 Estrutura de OUs

```
mydomain.com
├── _ADMINS
├── _USERS
├── _TI
├── _RH
├── _FINANCEIRO
└── _COMPUTADORES
    ├── _DESKTOPS   ← Client1 e Client2
    └── _LAPTOPS
```

---

## ✅ Atividades Realizadas

### Nível 1 — Gerenciamento Básico
- Criação manual de usuários com cargos e departamentos
- Criação de grupos de segurança por departamento
- Criação de hierarquia de OUs simulando ambiente corporativo
- Movimentação de objetos de computador para OUs corretas
- Distinção prática entre **Containers** padrão e **Organizational Units**

### Nível 2 — Group Policy Objects (GPO)
- Criação e vinculação de GPO à OU `_RH`
- Bloqueio de acesso ao Painel de Controle via GPO
- Teste e validação com `gpupdate /force` no cliente
- Entendimento do uso de GPOs como ferramenta defensiva de hardening

### Nível 3 — Simulação de Helpdesk
- Configuração de Account Lockout Policy (3 tentativas, 30 min)
- Simulação de incidente: bloqueio de conta por senha errada
- Desbloqueio de conta via **ADUC** (interface gráfica)
- Desbloqueio de conta via **PowerShell**:
```powershell
Import-Module ActiveDirectory
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName
Unlock-ADAccount -Identity nomeusuario
```
- Resolução de problema de relação de confiança quebrada entre cliente e domínio após clonagem de VM (`Reset-ComputerMachinePassword`)
- Instalação do **RSAT** no cliente para habilitar módulo ActiveDirectory

### Nível 4 — Segurança e Monitoramento
- Configuração de Advanced Audit Policy via GPO:
  - Audit Logon — Success and Failure
  - Audit Account Lockout — Success and Failure
- Análise de eventos no **Event Viewer** (Security Log):

| Event ID | Descrição |
|---|---|
| 4624 | Logon bem-sucedido |
| 4625 | Falha de logon |
| 4740 | Conta bloqueada |
| 4769 | Solicitação de ticket Kerberos (Kerberoasting) |

- Filtragem de eventos via PowerShell:
```powershell
Get-WinEvent -LogName Security |
Where-Object {$_.Id -eq 4625} |
Select-Object TimeCreated, Message -First 10
```

- Criação de conta de serviço vulnerável com SPN para simular vetor de Kerberoasting:
```powershell
New-ADUser -Name "svc_sqlserver" -SamAccountName "svc_sqlserver" `
           -AccountPassword (ConvertTo-SecureString "Password123" -AsPlainText -Force) `
           -Enabled $true

Set-ADUser -Identity "svc_sqlserver" `
           -ServicePrincipalNames @{Add="MSSQLSvc/mydomain.com:1433"}
```

- Enumeração de SPNs do domínio a partir de conta comum (perspectiva do atacante):
```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} `
           -Properties ServicePrincipalName |
           Select-Object Name, ServicePrincipalName
```

- Solicitação de ticket Kerberos para geração do Event ID 4769:
```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken `
           -ArgumentList "MSSQLSvc/mydomain.com:1433"
```

---

## 🔍 Principais Aprendizados

- GPOs são uma das ferramentas mais eficazes de **prevenção proativa** — reduzem a superfície de ataque antes mesmo de qualquer incidente
- Containers padrão do AD (Computers, Users) não aceitam GPOs — apenas OUs aceitam
- O módulo ActiveDirectory do PowerShell exige `Import-Module ActiveDirectory` e execução como Administrador
- Em máquinas cliente, o módulo ActiveDirectory requer instalação do **RSAT**
- Kerberoasting explora SPNs registrados em contas de usuário com senhas fracas — a contramedida é usar **gMSA (Group Managed Service Accounts)**
- O Event ID `4769` é o indicador primário de tentativa de Kerberoasting em um SIEM

---

## 📸 Evidências

| Atividade | Print |
|---|---|
| Estrutura de OUs e usuários | ![OUs](evidencias/NOVAS_OUs_E_USUARIOS.png) |
| GPO de bloqueio do Painel de Controle | ![GPO](evidencias/GPO_DE_BLOQUEIO_DE_ACESSO_AO_PAINEL_DE_CONTROLE.png) |
| Tentativa de acesso bloqueada | ![Bloqueio](evidencias/TENTATIVA_DE_ACESSO_AO_PAINEL_DE_CONTROLE_BLOQUEADA.png) |
| Configuração de Account Lockout | ![Lockout](evidencias/CONFIGURAÇÃO_DE_LOCKOUT.png) |
| Conta bloqueada no login | ![Conta bloqueada](evidencias/TENTATIVAS_DE_ACESSO_LIMITE_ATINGIDA_E_USUSARIO_BLOQUEADO.png) |
| Desbloqueio via ADUC | ![ADUC](evidencias/DESBLOQUEIO_DO_UUSARIO_PELO_ADUC.png) |
| Configuração de auditoria | ![Auditoria](evidencias/CONFIGURAÇÃO_DE_AUDITORIA_DE_EVENTOS.png) |
| Logs Kerberos (Event ID 4769) | ![Kerberos](evidencias/LOGS_DO_KERBEROS.png) |

---

## 📚 Referências

- [Josh Madakor — Active Directory Lab](https://www.youtube.com/watch?v=MHsI8hJmggI)
- [Microsoft Docs — Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [MITRE ATT&CK — Kerberoasting T1558.003](https://attack.mitre.org/techniques/T1558/003/)

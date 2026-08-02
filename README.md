# AWS Storage Gateway — Implantação Completa (Windows / Hyper-V)

## Visão Geral

Este projeto documenta a implantação completa do **AWS Storage Gateway** em ambiente Windows com Hyper-V, cobrindo dois tipos de gateway:

- **Volume Gateway (Cached Volumes)** — apresenta volumes iSCSI baseados em nuvem como discos locais
- **File Gateway (SMB)** — expõe buckets S3 como compartilhamentos de rede SMB

O fluxo de implantação é dividido em 4 etapas sequenciais:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Etapa 1       │    │   Etapa 2       │    │   Etapa 3       │    │   Etapa 4       │
│   Criar VM no   │───▶│   Ativar GW +   │───▶│   File Gateway  │───▶│   Montar SMB    │
│   Hyper-V       │    │   Conexão iSCSI │    │   + SMB Share   │    │   Localmente    │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Pré-Requisitos

### Hypervisor

| Requisito | Mínimo |
|-----------|--------|
| Sistema Operacional | Windows Server 2016+ ou Windows 10/11 Pro com Hyper-V |
| Hyper-V | Habilitado e funcional |
| vCPUs disponíveis | 4 |
| RAM disponível | 16 GB |
| Disco disponível | 80 GB (sistema) + 150 GB (cache) + 150 GB (upload buffer) |
| Rede | Virtual Switch com acesso à internet |

### Conta AWS

| Permissão / Política | Finalidade |
|----------------------|------------|
| `AWSStorageGatewayFullAccess` | Criar e gerenciar gateways |
| `AmazonS3FullAccess` | Criar buckets e gerenciar objetos (File Gateway) |
| `IAMFullAccess` | Criar roles IAM para o File Gateway |

### Conectividade de Rede

| Porta | Protocolo | Direção | Finalidade |
|-------|-----------|---------|------------|
| 80 | TCP | VM → Admin | Ativação do gateway no Console AWS |
| 443 | TCP | VM → Internet | Comunicação com endpoints AWS |
| 3260 | TCP | Cliente → VM | Conexão iSCSI (Volume Gateway) |
| 445 | TCP | Cliente → VM | Compartilhamento SMB (File Gateway) |

## Estrutura do Projeto

```
aws-storage-gateway/
├── README.md                          ← Este arquivo
├── config/
│   └── variaveis.env                  ← Parâmetros do ambiente
├── docs/
│   ├── arquitetura.md                 ← Topologia e fluxo de dados
│   └── implantacao-windows.md         ← Guia passo a passo (Windows/Hyper-V)
├── etapa-1-volume-gateway/            ← Criação da VM
├── etapa-2-iscsi/                     ← Ativação do gateway + iSCSI
├── etapa-3-file-gateway/              ← File Gateway + SMB
├── etapa-4-montagem-smb/              ← Montagem local do SMB
├── lib/                               ← Módulos auxiliares
└── logs/                              ← Logs de execução
```

## Instruções Rápidas por Etapa

### Etapa 1 — Criar VM no Hyper-V

1. Console AWS → Storage Gateway → **Create gateway** → Baixar imagem `.vhdx`
2. Hyper-V Manager → Nova VM → 4 vCPU, 16 GB RAM, disco `.vhdx` baixado
3. Adicionar 2 discos VHD: Cache (150 GB) e Upload Buffer (150 GB)
4. Iniciar a VM e anotar o IP

**Verificação:**
```powershell
Test-NetConnection -ComputerName <IP_DA_VM> -Port 80
# Esperado: TcpTestSucceeded : True
```

### Etapa 2 — Ativar Volume Gateway + Conexão iSCSI

1. Console AWS → Ativar gateway com IP da VM → Configurar discos (Cache + Upload Buffer)
2. Criar volume (Cached Volume) → Anotar o Target IQN
3. Na máquina destino: iSCSI Initiator → Descobrir portal → Conectar com CHAP
4. Disk Management → Inicializar → Formatar NTFS → Atribuir letra

**Verificação:**
```powershell
Get-Disk | Where-Object { $_.BusType -eq "iSCSI" }
# Esperado: disco iSCSI listado com status Online
```

### Etapa 3 — File Gateway + Compartilhamento SMB

1. Console AWS → Criar File Gateway (S3 File Gateway) com IP da VM
2. Criar File Share: bucket S3 + protocolo SMB + autenticação Guest
3. Aguardar status Available

**Verificação:**
```powershell
Test-NetConnection -ComputerName <IP_DA_VM> -Port 445
# Esperado: TcpTestSucceeded : True
```

### Etapa 4 — Montar SMB Localmente

```powershell
net use Z: \\<IP_DA_VM>\<nome-do-bucket> /user:sgw\smbguest <senha> /persistent:yes
```

**Verificação:**
```powershell
# Criar, ler e excluir arquivo de teste
"teste" | Out-File Z:\health-check.txt
Get-Content Z:\health-check.txt
Remove-Item Z:\health-check.txt
```

## Troubleshooting

### VM não obtém IP no Hyper-V

**Causa:** Virtual switch sem conexão externa.
**Solução:** No Hyper-V Manager → Virtual Switch Manager → Criar switch do tipo **External** vinculado à placa de rede física.

### Porta 80 não responde após iniciar a VM

**Causa:** A VM ainda está inicializando ou o firewall do host bloqueia.
**Solução:**
- Aguardar até 5 minutos para inicialização completa
- Verificar regras do Windows Firewall no host
- Conectar ao console da VM para verificar se o serviço está ativo

### iSCSI target não aparece na descoberta

**Causa:** Porta 3260 bloqueada entre a máquina destino e a VM.
**Solução:**
```powershell
# Testar conectividade
Test-NetConnection -ComputerName <IP_DA_VM> -Port 3260

# Se falhar, verificar firewall da VM e do host
New-NetFirewallRule -DisplayName "iSCSI" -Direction Inbound -LocalPort 3260 -Protocol TCP -Action Allow
```

### SMB "Access Denied" ao montar

**Causa:** Senha do guest incorreta ou serviço SMB não disponível.
**Solução:**
- Verificar a senha definida no Console AWS (File Share → SMB settings)
- Remover conexão antiga: `net use Z: /delete`
- Reconectar com credenciais corretas

### Arquivo não aparece no bucket S3

**Causa:** Latência de sincronização do File Gateway (até 60 segundos).
**Solução:** Aguardar 60 segundos e atualizar a visualização do bucket no Console AWS. Para forçar, use **Refresh cache** no Console do File Share.

## Referências

- [Documentação oficial AWS Storage Gateway](https://docs.aws.amazon.com/storagegateway/)
- [Requisitos de hardware para VM](https://docs.aws.amazon.com/storagegateway/latest/vgw/Requirements.html)
- [Configuração iSCSI para Windows](https://docs.aws.amazon.com/storagegateway/latest/vgw/initiator-connection-common.html)

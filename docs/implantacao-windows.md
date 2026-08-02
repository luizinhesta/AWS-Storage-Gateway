# Guia de Implantação — Windows / Hyper-V

## Introdução

Este guia descreve o procedimento completo de implantação do AWS Storage Gateway em ambiente Windows com Hyper-V, cobrindo desde a criação da VM até a montagem final dos volumes e compartilhamentos.

**Tempo estimado:** 30-45 minutos (sem contar downloads)

---

## Etapa 1 — Criar a VM do Storage Gateway no Hyper-V

### 1.1 Baixar a Imagem da VM

1. Acesse o **Console AWS** → serviço **Storage Gateway**
2. Clique em **Create gateway**
3. Em **Gateway type**, selecione **Volume Gateway**
4. Em **Host platform**, selecione **Microsoft Hyper-V**
5. Clique em **Download image**
6. Extraia o arquivo `.zip` para uma pasta local (ex: `C:\VMs\AWS-SGW\`)

> O arquivo extraído será um `.vhdx` — o disco virtual do appliance.

### 1.2 Criar a Máquina Virtual

1. Abra o **Hyper-V Manager**
2. No painel Actions → **New** → **Virtual Machine**
3. Wizard de criação:

| Campo | Valor |
|-------|-------|
| Name | `AWS-StorageGateway` |
| Generation | Generation 1 (recomendado para compatibilidade) |
| Startup Memory | `16384` MB (desmarcar memória dinâmica) |
| Networking | Selecionar Virtual Switch com acesso externo |
| Hard Disk | **Use an existing virtual hard disk** → selecionar o `.vhdx` |

4. Clique **Finish**

### 1.3 Configurar Processadores

1. Clique com botão direito na VM → **Settings**
2. **Processor** → Number of virtual processors: `4`
3. Clique **Apply**

### 1.4 Adicionar Disco Cache

1. Em Settings → **IDE Controller 1** (ou SCSI Controller) → **Add** → **Hard Drive**
2. Clique **New**
3. Selecione **Fixed size** (recomendado para performance)
4. Tamanho: `150` GB
5. Local: `C:\VMs\AWS-SGW\cache.vhdx`
6. Clique **Finish**

### 1.5 Adicionar Disco Upload Buffer

1. Mesmo processo: **Add** → **Hard Drive** → **New**
2. Tamanho: `150` GB
3. Local: `C:\VMs\AWS-SGW\upload-buffer.vhdx`
4. Clique **Finish** → **Apply** → **OK**

### 1.6 Iniciar e Verificar

1. Clique com botão direito na VM → **Start**
2. Clique em **Connect** para ver o console
3. Aguarde 2-3 minutos para inicialização completa
4. A VM exibirá o IP obtido por DHCP no console
5. Anote o IP (ex: `192.168.1.50`)

**Verificação no PowerShell:**

```powershell
Test-NetConnection -ComputerName 192.168.1.50 -Port 80
```

Resultado esperado:
```
ComputerName     : 192.168.1.50
RemotePort       : 80
TcpTestSucceeded : True
```

> Se `TcpTestSucceeded : False`, aguarde mais tempo ou verifique o Virtual Switch.

---

## Etapa 2 — Ativar o Volume Gateway e Configurar iSCSI

### 2.1 Ativar o Gateway no Console AWS

1. Console AWS → **Storage Gateway** → Continue o wizard de criação (ou **Create gateway**)
2. Em **Gateway connection options**:
   - **IP address**: insira o IP da VM (ex: `192.168.1.50`)
3. Clique **Next**
4. Configure:
   - **Gateway name**: `VolumeGateway-Local`
   - **Gateway time zone**: `(UTC-03:00) South America (Sao Paulo)`
   - **CloudWatch log group**: opcional (pode criar para monitoramento)
5. Clique **Activate gateway**
6. Aguarde o status mudar para **Running** (1-2 minutos)

### 2.2 Configurar Discos Locais

Na tela de configuração de discos (aparece automaticamente após ativação):

| Disco | Tamanho | Função a atribuir |
|-------|---------|-------------------|
| `/dev/sdb` ou similar | 150 GB | **Cache** |
| `/dev/sdc` ou similar | 150 GB | **Upload Buffer** |

Clique **Configure logging** → **Save and continue**

### 2.3 Criar Volume iSCSI

1. No menu lateral → **Volumes** → **Create volume**
2. Configure:

| Campo | Valor |
|-------|-------|
| Gateway | `VolumeGateway-Local` |
| Volume type | Cached volume |
| Capacity (GiB) | `500` (ou conforme necessidade, 1-32768) |
| Snapshot ID | Em branco (volume novo) |
| Target name | `vol1` |

3. Clique **Create volume**
4. Anote:
   - **Target IQN**: `iqn.1997-05.com.amazon:sgw-XXXXXXXX-vol1`
   - **Network Interface**: IP da VM

### 2.4 Configurar CHAP (Autenticação)

1. Selecione o volume criado → aba **Actions** → **Configure CHAP authentication**
2. Defina:
   - **Initiator name**: `iqn.1991-05.com.microsoft:SEU-PC`
   - **Initiator secret**: senha com 12+ caracteres (ex: `ChapSecret12345`)
   - **Target secret**: outra senha com 12+ caracteres (ex: `TargetSecret1234`)
3. Clique **Save**

### 2.5 Configurar iSCSI Initiator no Windows

1. Pressione `Win + S` → digite **iSCSI Initiator** → abra
2. Se pedir para iniciar o serviço Microsoft iSCSI, clique **Yes**

**Descobrir o Target:**
1. Aba **Discovery** → **Discover Portal**
2. IP address: `192.168.1.50`
3. Port: `3260`
4. Clique **OK**

**Conectar ao Target:**
1. Aba **Targets** → selecione o target (status: Inactive)
2. Clique **Connect**
3. Marque **Enable CHAP log on**
4. Clique **Advanced**:
   - **Target secret**: a senha do target definida no passo 2.4
   - **Initiator secret**: a senha do initiator definida no passo 2.4
5. Marque **Add this connection to the list of Favorite Targets**
6. Clique **OK** → status muda para **Connected**

### 2.6 Formatar e Montar o Disco

1. Pressione `Win + R` → `diskmgmt.msc` → Enter
2. O novo disco aparece como **Not Initialized**
3. Clique com botão direito no disco → **Initialize Disk** → GPT → **OK**
4. Clique com botão direito no espaço não alocado → **New Simple Volume**
5. Wizard:

| Campo | Valor |
|-------|-------|
| Volume size | Máximo disponível |
| Drive letter | `E:` (ou outra livre) |
| File system | NTFS |
| Allocation unit size | Default |
| Volume label | `iSCSI-Volume` |
| Perform quick format | Marcado |

6. Clique **Finish**

**Verificação:**

```powershell
# Listar discos iSCSI
Get-Disk | Where-Object { $_.BusType -eq "iSCSI" }

# Testar integridade
$testFile = "E:\teste-integridade.bin"
$data = [byte[]](1..1024) * 1024  # 1 MB
[System.IO.File]::WriteAllBytes($testFile, $data)
$hash1 = (Get-FileHash $testFile -Algorithm SHA256).Hash
$readBack = [System.IO.File]::ReadAllBytes($testFile)
[System.IO.File]::WriteAllBytes("$testFile.verify", $readBack)
$hash2 = (Get-FileHash "$testFile.verify" -Algorithm SHA256).Hash

if ($hash1 -eq $hash2) {
    Write-Host "Health Check OK: integridade confirmada" -ForegroundColor Green
} else {
    Write-Host "FALHA: hashes nao coincidem" -ForegroundColor Red
}

Remove-Item $testFile, "$testFile.verify" -ErrorAction SilentlyContinue
```

---

## Etapa 3 — File Gateway com Compartilhamento SMB

### 3.1 Criar/Ativar o File Gateway

> Você pode usar a mesma VM do Volume Gateway ou criar uma VM separada (recomendado para produção).

1. Console AWS → **Storage Gateway** → **Create gateway**
2. Gateway type: **Amazon S3 File Gateway**
3. Host platform: **Microsoft Hyper-V**
4. Se estiver usando nova VM: repetir passos da Etapa 1
5. Se estiver usando a mesma VM: pular para conexão
6. Em **Gateway connection options**: inserir IP da VM
7. Nome: `FileGateway-Local`
8. Fuso horário: `(UTC-03:00) South America (Sao Paulo)`
9. Clique **Activate gateway**
10. Configurar disco como **Cache** (150 GB)

### 3.2 Criar o Bucket S3 (se não existir)

1. Console AWS → **S3** → **Create bucket**
2. Nome: `meu-bucket-file-gateway` (deve ser globalmente único)
3. Região: `us-east-1` (ou a desejada)
4. Manter configurações padrão → **Create bucket**

### 3.3 Criar Compartilhamento SMB (File Share)

1. Console AWS → Storage Gateway → **File shares** → **Create file share**
2. Configure:

| Campo | Valor |
|-------|-------|
| Gateway | `FileGateway-Local` |
| Amazon S3 bucket name | `meu-bucket-file-gateway` |
| AWS Region | `us-east-1` |
| Access objects using | **Server Message Block (SMB)** |
| IAM role | Create a new IAM role (automático) |

3. Clique **Next**
4. Em **Amazon S3 storage settings**: manter padrão
5. Em **Access settings**:
   - **Authentication method**: Guest access
   - **Set guest password**: definir senha (ex: `GuestPass123!`)
6. Clique **Create file share**
7. Aguardar status **Available**

### 3.4 Anotar Informações do Compartilhamento

Após criação, clique no file share para ver:
- **Path**: `\\192.168.1.50\meu-bucket-file-gateway`
- **Comando de montagem**: o Console mostra o comando pronto

**Verificação:**

```powershell
Test-NetConnection -ComputerName 192.168.1.50 -Port 445
```

Resultado esperado: `TcpTestSucceeded : True`

---

## Etapa 4 — Montar o Compartilhamento SMB no Windows

### 4.1 Montar o Drive

No PowerShell (como Administrador):

```powershell
net use Z: \\192.168.1.50\meu-bucket-file-gateway /user:sgw\smbguest GuestPass123! /persistent:yes
```

Ou pelo **File Explorer**:
1. Clique com botão direito em **This PC** → **Map network drive**
2. Drive: `Z:`
3. Folder: `\\192.168.1.50\meu-bucket-file-gateway`
4. Marcar **Reconnect at sign-in**
5. Clicar **Connect using different credentials**
6. Usuário: `sgw\smbguest` | Senha: `GuestPass123!`

### 4.2 Health Check Completo

```powershell
$mountPoint = "Z:"
$testFile = "$mountPoint\health-check-test.txt"
$timeout = 30
$stopwatch = [System.Diagnostics.Stopwatch]::StartNew()

try {
    # Criar
    "Teste de integridade - $(Get-Date -Format 'yyyy-MM-ddTHH:mm:ss')" | Out-File $testFile
    Write-Host "[OK] Arquivo criado" -ForegroundColor Green

    # Ler
    $conteudo = Get-Content $testFile
    if ($conteudo) {
        Write-Host "[OK] Arquivo lido: $conteudo" -ForegroundColor Green
    } else {
        throw "Falha na leitura"
    }

    # Excluir
    Remove-Item $testFile -Force
    if (-not (Test-Path $testFile)) {
        Write-Host "[OK] Arquivo excluido" -ForegroundColor Green
    } else {
        throw "Falha na exclusao"
    }

    $stopwatch.Stop()
    if ($stopwatch.Elapsed.TotalSeconds -le $timeout) {
        Write-Host "`nHEALTH CHECK PASSED em $([math]::Round($stopwatch.Elapsed.TotalSeconds, 1))s" -ForegroundColor Green
    } else {
        Write-Host "`nAVISO: Operacao concluida mas excedeu timeout de ${timeout}s" -ForegroundColor Yellow
    }
} catch {
    $stopwatch.Stop()
    Write-Host "`nHEALTH CHECK FAILED: $_" -ForegroundColor Red
}
```

### 4.3 Verificar Sincronização com S3

1. Crie um arquivo no drive `Z:`
2. Aguarde 60 segundos
3. Acesse Console AWS → **S3** → seu bucket
4. O arquivo deve aparecer como objeto no bucket

---

## Resumo Final

Após concluir todas as etapas:

| Recurso | Endpoint |
|---------|----------|
| IP do Gateway | `192.168.1.50` |
| Volume iSCSI | Drive `E:` (IQN: `iqn.1997-05.com.amazon:sgw-XXXXX-vol1`) |
| Compartilhamento SMB | Drive `Z:` (`\\192.168.1.50\meu-bucket-file-gateway`) |
| Bucket S3 | `meu-bucket-file-gateway` |
| Console AWS | Storage Gateway → seus gateways ativos |

---

## Próximos Passos

- [ ] Configurar **CloudWatch Alarms** para monitorar o gateway
- [ ] Agendar **snapshots** automáticos do volume iSCSI
- [ ] Configurar **lifecycle policies** no bucket S3
- [ ] Testar **disaster recovery** restaurando snapshot em nova VM
- [ ] Avaliar migração de autenticação SMB para **Active Directory** em produção

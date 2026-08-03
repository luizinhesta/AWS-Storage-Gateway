# Implantação — File Gateway SMB (Passo a Passo)

## Ambiente

| Item | Valor |
|------|-------|
| VM | Windows Server no Hyper-V |
| IP da VM | 192.168.15.104 |
| Região AWS | us-east-1 (Norte da Virgínia) |
| Bucket S3 | `sgw-file-gateway-storage` |
| Nome do Gateway | `FileGateway-Storage` |
| Compartilhamento | `\\192.168.15.104\sgw-file-gateway-storage` |
| Drive no PC | `Z:` |

---

## Passo 1 — Criar o Bucket S3

1. Acesse o **Console AWS** → serviço **S3**
2. Clique em **Create bucket**
3. Configure:
   - **Bucket name**: `sgw-file-gateway-storage`
   - **AWS Region**: US East (N. Virginia) us-east-1
4. Mantenha as demais configurações padrão
5. Clique em **Create bucket**

**Resultado esperado:** Bucket `sgw-file-gateway-storage` criado e listado na página do S3.

---

## Passo 2 — Instalar o File Gateway na VM Windows Server

### 2.1 Baixar o instalador

1. Console AWS → serviço **Storage Gateway**
2. Clique em **Create gateway**
3. Em **Configurar gateway**:
   - **Nome do gateway**: `FileGateway-Storage`
   - **Tipo de gateway**: Amazon S3 File Gateway
   - **Plataforma do host**: **Amazon EC2** ou **Microsoft Hyper-V** (selecione conforme seu caso)

> **Nota:** Como você está usando Windows Server em VM, a opção mais prática pode ser instalar o software do gateway diretamente. Alternativamente, se a AWS não oferecer instalador direto para Windows Server, será necessário usar a imagem `.vhdx` como VM separada dentro do Windows Server (VM aninhada).

### 2.2 Opção alternativa — Usar o gateway via EC2

Se a instalação local não funcionar por limitação de recursos:

1. No wizard de criação, selecione **Amazon EC2** como plataforma
2. A AWS vai criar uma instância EC2 com o gateway pré-configurado
3. Use o IP público da instância EC2 no lugar de `192.168.15.104`
4. Ajuste o endpoint SMB para o IP da EC2

---

## Passo 3 — Ativar o Gateway no Console AWS

1. Console AWS → **Storage Gateway** → **Create gateway** (ou continue o wizard)
2. Na tela **Conectar-se à AWS**:
   - **Opções de conexão**: Endereço IP
   - **Endereço IP**: `192.168.15.104` (IP da VM onde o gateway está rodando)
   - **Endpoint de serviço**: Publicamente acessível
   - **FIPS**: Desmarcado
   - **Versão do IP**: IPv4 usando endpoint somente IPv4
3. Clique em **Próximo**
4. Na tela **Revisar e ativar**:
   - Confirme as informações
   - Clique em **Ativar gateway**
5. Aguarde o status mudar para **Running** (1-2 minutos)

**Resultado esperado:** Gateway `FileGateway-Storage` com status **Running** no Console AWS.

---

## Passo 4 — Configurar Disco de Cache

Após a ativação, o Console pede para configurar os discos locais:

1. Na tela **Configurar gateway** (Etapa 4):
   - Identifique o disco disponível na VM
   - Atribua como **Cache**
2. Clique em **Salvar e continuar**

> O disco de cache armazena temporariamente os arquivos acessados com frequência para leitura rápida.

---

## Passo 5 — Criar o Compartilhamento SMB (File Share)

1. Console AWS → **Storage Gateway** → **File shares** → **Create file share**
2. Configure:

| Campo | Valor |
|-------|-------|
| Gateway | `FileGateway-Storage` |
| Amazon S3 bucket name | `sgw-file-gateway-storage` |
| AWS Region | us-east-1 |
| Access objects using | **Server Message Block (SMB)** |
| IAM role | Create a new IAM role (automático) |

3. Clique **Next**
4. Em **Amazon S3 storage settings**: manter padrão
5. Em **Access settings**:
   - **Authentication method**: Guest access
   - **Set guest password**: definir senha (ex: `SmbPass2024!`)
6. Clique em **Create file share**
7. Aguarde o status ficar **Available**

**Resultado esperado:** File share `sgw-file-gateway-storage` com status **Available**.

---

## Passo 6 — Montar o Compartilhamento SMB no PC Host

### 6.1 Verificar conectividade

No PowerShell do **PC Host** (não da VM):

```powershell
Test-NetConnection -ComputerName 192.168.15.104 -Port 445
```

Resultado esperado: `TcpTestSucceeded : True`

### 6.2 Montar o drive

No PowerShell como Administrador:

```powershell
net use Z: \\192.168.15.104\sgw-file-gateway-storage /user:sgw\smbguest SmbPass2024! /persistent:yes
```

Ou pelo **File Explorer**:
1. Abra o Explorador de Arquivos
2. Clique com botão direito em **Este Computador** → **Mapear unidade de rede**
3. Drive: `Z:`
4. Pasta: `\\192.168.15.104\sgw-file-gateway-storage`
5. Marque **Reconectar ao entrar**
6. Clique em **Conectar usando credenciais diferentes**
7. Usuário: `sgw\smbguest`
8. Senha: `SmbPass2024!`
9. Clique **OK**

**Resultado esperado:** Drive `Z:` aparece no Explorador de Arquivos como unidade de rede.

---

## Passo 7 — Testar o Compartilhamento

### 7.1 Teste rápido

```powershell
# Criar arquivo
"Teste File Gateway" | Out-File Z:\teste-sgw.txt

# Ler arquivo
Get-Content Z:\teste-sgw.txt

# Excluir arquivo
Remove-Item Z:\teste-sgw.txt
```

### 7.2 Verificar no S3

1. Acesse Console AWS → **S3** → bucket `sgw-file-gateway-storage`
2. O arquivo `teste-sgw.txt` deve aparecer após ~60 segundos
3. Se não aparecer, use **Refresh cache** no file share (Console → Storage Gateway → File shares)

### 7.3 Health Check completo

```powershell
$drive = "Z:"
$testFile = "$drive\health-check.txt"

Write-Host "=== Health Check File Gateway ===" -ForegroundColor Cyan
Write-Host ""

# Teste 1: Criar
try {
    "Health Check - $(Get-Date)" | Out-File $testFile
    Write-Host "[PASS] Criar arquivo" -ForegroundColor Green
} catch {
    Write-Host "[FAIL] Criar arquivo: $_" -ForegroundColor Red
    exit 1
}

# Teste 2: Ler
try {
    $conteudo = Get-Content $testFile
    if ($conteudo) {
        Write-Host "[PASS] Ler arquivo: $conteudo" -ForegroundColor Green
    } else {
        throw "Conteudo vazio"
    }
} catch {
    Write-Host "[FAIL] Ler arquivo: $_" -ForegroundColor Red
}

# Teste 3: Excluir
try {
    Remove-Item $testFile -Force
    if (-not (Test-Path $testFile)) {
        Write-Host "[PASS] Excluir arquivo" -ForegroundColor Green
    } else {
        throw "Arquivo ainda existe"
    }
} catch {
    Write-Host "[FAIL] Excluir arquivo: $_" -ForegroundColor Red
}

Write-Host ""
Write-Host "=== Health Check concluido ===" -ForegroundColor Cyan
```

**Resultado esperado:**
```
=== Health Check File Gateway ===

[PASS] Criar arquivo
[PASS] Ler arquivo: Health Check - 08/02/2026 14:30:00
[PASS] Excluir arquivo

=== Health Check concluido ===
```

---

## Resumo Final

Após concluir todos os passos:

| Recurso | Valor |
|---------|-------|
| Gateway | `FileGateway-Storage` (status: Running) |
| Bucket S3 | `sgw-file-gateway-storage` |
| File Share | `\\192.168.15.104\sgw-file-gateway-storage` |
| Drive mapeado | `Z:` |
| Autenticação | Guest (`sgw\smbguest`) |

---

## Problemas Comuns

| Erro | Causa | Solução |
|------|-------|---------|
| "Ativação falhou" no Console AWS | VM não acessível na porta 80 | Verificar firewall da VM; testar `Test-NetConnection 192.168.15.104 -Port 80` |
| "Access Denied" ao montar Z: | Senha incorreta ou conexão antiga | `net use Z: /delete` e reconectar com senha correta |
| Arquivo não aparece no S3 | Sincronização em andamento | Aguardar 60s; usar "Refresh cache" no Console |
| "Network path not found" | Porta 445 bloqueada | Liberar porta 445 no firewall da VM |
| Gateway "Offline" no Console | VM sem internet | Verificar se VM acessa internet (porta 443) |

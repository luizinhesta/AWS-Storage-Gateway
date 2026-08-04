# AWS Storage Gateway — File Gateway (SMB)

![Objetivos](imagens/imagem(1).png)

## Visão Geral

Este projeto documenta a implantação do **AWS Storage Gateway** no modo **File Gateway**, expondo um bucket S3 como compartilhamento de rede SMB acessível no Windows.

O File Gateway permite que você acesse arquivos armazenados no Amazon S3 como se fossem uma pasta de rede local, usando o protocolo SMB (Server Message Block).

![Objetivos](imagens/imagem(2).png)

```
┌────────────────────┐         ┌──────────────────────┐         ┌───────────┐
│  Seu PC (Windows)  │──SMB───▶│  VM Windows Server   │──HTTPS──▶│ Amazon S3 │
│  Drive Z:          │  445    │  File Gateway        │  443    │  Bucket   │
└────────────────────┘         └──────────────────────┘         └───────────┘
```

## Ambiente

| Componente | Detalhes |
|------------|----------|
| PC Host | Windows com Hyper-V |
| VM | Windows Server (IP: `<IP_DA_VM>`) |
| Gateway | AWS Storage Gateway - File Gateway |
| Bucket S3 | `sgw-file-gateway-storage` |
| Compartilhamento | `\\<IP_DA_VM>\sgw-file-gateway-storage` |
| Drive mapeado | `Z:` |
| Região AWS | us-east-1 (Norte da Virgínia) |

## Pré-Requisitos

| Requisito | Detalhes |
|-----------|----------|
| VM Windows Server | Rodando no Hyper-V com acesso à internet |
| Conta AWS | Com permissões `AWSStorageGatewayFullAccess` e `AmazonS3FullAccess` |
| Portas liberadas | 80 (ativação), 443 (dados AWS), 445 (SMB) |
| Conectividade | VM com acesso à internet e acessível pelo PC host |

## Estrutura do Projeto

```
aws-storage-gateway/
├── README.md                       ← Este arquivo (visão geral)
├── docs/
│   ├── arquitetura.md              ← Diagrama e fluxo de dados
│   └── implantacao-windows.md      ← Guia passo a passo completo
├── imagens/                        ← Prints do processo
└── testes/                         ← Arquivos de teste
```

---

## Arquitetura do Projeto

> Integração híbrida entre ambiente local Windows/Hyper-V e Amazon S3

---

### 1. Acesso Local

![Objetivos](imagens/imagem(9).png)

O usuário acessa os arquivos pelo PC Host Windows, através do Explorador de Arquivos ou qualquer aplicação, usando um drive de rede mapeado.

| Item | Detalhe |
|------|---------|
| PC Host | Windows com Hyper-V |
| Acesso | Drive de Rede Z: |
| Caminho | `\\<IP_DA_VM>\sgw-file-gateway-storage` |
| Protocolo | SMB (porta 445) |

---

### 2. Ambiente Local

![Objetivos](imagens/imagem(10).png)

A VM Windows Server hospeda o AWS File Gateway, que atua como ponte entre o compartilhamento SMB e o Amazon S3.

| Item | Detalhe |
|------|---------|
| VM | Windows Server (IP: `<IP_DA_VM>`) |
| Software | AWS File Gateway |
| Disco de Cache | Arquivos acessados com frequência |
| Compartilhamento SMB | Guest access / `sgw\\smbguest` |
| Portas principais | 445 (SMB), 443 (HTTPS), 80 (ativação) |
| Status esperado | Running |

---

### 3. Conectividade AWS

![Objetivos](imagens/imagem(3).png)

O File Gateway se comunica com o Storage Gateway Service na AWS para ativação, transferência de dados e gerenciamento de permissões.

| Item | Detalhe |
|------|---------|
| Ativação do Gateway | HTTP 80 — Endpoint público |
| Transferência de Dados | HTTPS 443 — Criptografado |
| IAM Role | Permissões mínimas para o bucket S3 |
| Segurança | Comunicação HTTPS + IAM Role + Bucket S3 privado |

---

### 4. Armazenamento

![Objetivos](imagens/imagem(4).png)

O Amazon S3 é o destino final dos arquivos. Os objetos são armazenados no bucket privado e acessados como arquivos SMB pelo gateway.

| Item | Detalhe |
|------|---------|
| Bucket S3 | `sgw-file-gateway-storage` |
| Tipo | Privado |
| Objetos | Acessados como arquivos SMB |
| Operações | Upload para S3, Leitura via cache, Refresh cache, Sincronização (<60s) |

---

### 5. Fluxo do Processamento

![Objetivos](imagens/imagem(5).png)

Passo a passo de como os dados trafegam na solução:

| Etapa | Descrição |
|-------|-----------|
| 1 | Usuário acessa Z: |
| 2 | Arquivo é enviado via SMB |
| 3 | File Gateway recebe na VM |
| 4 | Cache local grava temporariamente |
| 5 | Upload seguro para o Amazon S3 |
| 6 | Arquivo disponível no bucket / leitura posterior |

> 💡 **Leitura:** se estiver no cache, resposta rápida; se não, o gateway busca no S3 e entrega ao usuário.

---

### 6. Recursos Utilizados

![Objetivos](imagens/imagem(6).png)

| Recurso | Tipo |
|---------|------|
| AWS Storage Gateway | Serviço AWS |
| Amazon S3 | Armazenamento |
| IAM Role | Segurança/Permissões |
| Windows Server | Sistema Operacional (VM) |
| Hyper-V | Virtualização |
| SMB | Protocolo de compartilhamento |
| HTTPS | Protocolo de transferência |
| Cache Disk | Armazenamento local temporário |

---

### 7. Resumo Final

![Objetivos](imagens/imagem(7).png)

| Item | Valor |
|------|-------|
| Gateway | FileGateway-Storage |
| Bucket | `sgw-file-gateway-storage` |
| Compartilhamento | `\\<IP_DA_VM>\sgw-file-gateway-storage` |
| Drive | Z: |

---

### 8. Benefícios da Arquitetura

![Objetivos](imagens/imagem(8).png)

| # | Benefício | Descrição |
|---|-----------|-----------|
| 1 | **Integração Híbrida** | Conecta ambiente local ao S3 sem mudar a forma de acesso do usuário |
| 2 | **Cache Local** | Melhora a performance de leitura dos arquivos mais acessados |
| 3 | **Escalabilidade** | Amazon S3 oferece armazenamento altamente escalável |
| 4 | **Segurança** | Comunicação criptografada e controle por IAM |
| 5 | **Simplicidade Operacional** | Compartilhamento SMB familiar para usuários Windows |

---

## Resumo dos Passos

1. **Criar bucket S3** — `sgw-file-gateway-storage`
2. **Instalar o File Gateway** na VM Windows Server
3. **Ativar o gateway** no Console AWS
4. **Criar o compartilhamento SMB** apontando para o bucket
5. **Montar o drive Z:** no PC host
6. **Testar** criando arquivos e verificando no S3

Consulte `docs/implantacao-windows.md` para o guia detalhado passo a passo.

## Troubleshooting

| Problema | Solução |
|----------|---------|
| Ativação falha (não conecta no IP) | Verificar se a porta 80 está acessível: `Test-NetConnection <IP_DA_VM> -Port 80` |
| SMB "Access Denied" | Verificar senha do guest; remover conexão antiga: `net use Z: /delete` |
| Arquivo não aparece no S3 | Aguardar 60 segundos; usar "Refresh cache" no Console AWS |
| Porta 445 bloqueada | Verificar firewall do Windows Server: liberar porta 445 inbound |
| Gateway offline no Console | Verificar se a VM tem acesso à internet (porta 443) |

## Referências

- [Documentação AWS - File Gateway](https://docs.aws.amazon.com/storagegateway/latest/userguide/WhatIsStorageGateway.html)
- [Configurar File Share SMB](https://docs.aws.amazon.com/storagegateway/latest/userguide/CreatingAnSMBFileShare.html)

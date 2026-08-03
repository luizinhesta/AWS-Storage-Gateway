# AWS Storage Gateway — File Gateway (SMB)

## Visão Geral

Este projeto documenta a implantação do **AWS Storage Gateway** no modo **File Gateway**, expondo um bucket S3 como compartilhamento de rede SMB acessível no Windows.

O File Gateway permite que você acesse arquivos armazenados no Amazon S3 como se fossem uma pasta de rede local, usando o protocolo SMB (Server Message Block).

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

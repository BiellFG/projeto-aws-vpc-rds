# Projeto AWS — Infraestrutura VPC + RDS (PostgreSQL)

> Documento de acompanhamento do projeto prático de AWS, construído **por etapas** via Console AWS.
> **Região utilizada:** `sa-east-1` (São Paulo)

---

## 1. Objetivo

Praticar os serviços fundamentais de **rede** e **banco de dados** da AWS montando, na mão, uma infraestrutura que isola um banco **PostgreSQL (RDS)** dentro de uma **subnet privada**, sem acesso direto pela internet.

A ideia central: o banco fica **protegido na subnet privada** e tudo que precisa de acesso externo fica na **subnet pública**.

---

## 2. Arquitetura

### 2.1 Modelo inicial proposto

```
VPC
│
├── Subnet Pública
│
└── Subnet Privada
       │
       └── RDS
```

### 2.2 Arquitetura implementada (atual)

Durante a Etapa 3, a AWS exigiu que o *DB subnet group* cobrisse **pelo menos 2 Availability Zones**. Por isso adicionamos uma **segunda subnet privada** (`sa-east-1a`), mantendo as duas subnets privadas isoladas da internet.

```
VPC: projeto-vpc (10.0.0.0/16)
│
├── Internet Gateway (projeto-igw)
│
├── Subnet Pública        projeto-subnet-publica    10.0.1.0/24  — sa-east-1a
│       └── Route table   projeto-rt-publica  (0.0.0.0/0 → IGW)
│
├── Subnet Privada        projeto-subnet-privada    10.0.2.0/24  — sa-east-1b
│       └── Route table   projeto-rt-privada  (só rota local)
│
└── Subnet Privada 2      projeto-subnet-privada-2  10.0.3.0/24  — sa-east-1a
        └── Route table   main/default (só rota local)

DB subnet group: projeto-db-subnet-group (privada + privada-2)
      └── RDS PostgreSQL: projeto-db (single-AZ, sem acesso público)
```

---

## 3. Inventário de recursos criados

### 3.1 Rede (VPC)

| Recurso | Nome | Detalhes |
|---|---|---|
| VPC | `projeto-vpc` | CIDR `10.0.0.0/16` · ID `vpc-0xxxxxxxxxxxxxxxxx` |
| Internet Gateway | `projeto-igw` | Anexado à `projeto-vpc` |

### 3.2 Subnets

| Nome | CIDR | AZ | Auto-assign IP público |
|---|---|---|---|
| `projeto-subnet-publica` | `10.0.1.0/24` | `sa-east-1a` | Sim |
| `projeto-subnet-privada` | `10.0.2.0/24` | `sa-east-1b` | Não |
| `projeto-subnet-privada-2` | `10.0.3.0/24` | `sa-east-1a` | Não |

### 3.3 Route tables

| Nome | Rotas | Subnets associadas |
|---|---|---|
| `projeto-rt-publica` | `local` + `0.0.0.0/0 → projeto-igw` | `projeto-subnet-publica` |
| `projeto-rt-privada` | `local` | `projeto-subnet-privada` |
| `main` (default) | `local` | `projeto-subnet-privada-2` |

> ℹ️ A route table `main` é criada automaticamente com a VPC. A `projeto-subnet-privada-2` ficou associada a ela (só rota local), portanto também é **privada**.

### 3.4 Security Groups

| Nome | Inbound | Outbound | Uso |
|---|---|---|---|
| `projeto-sg-rds` | PostgreSQL (5432) de `10.0.0.0/16` | All traffic | Anexado ao RDS |
| `projeto-sg-ec2` | SSH (22) de `My IP` | All traffic | Anexado ao EC2 bastion |

### 3.5 Banco de dados (RDS)

| Campo | Valor |
|---|---|
| DB instance identifier | `projeto-db` |
| Engine | PostgreSQL |
| Instance class | `db.t4g.micro` (free tier) |
| Storage | 20 GiB · General Purpose SSD (gp2) · *sem autoscaling* |
| DB subnet group | `projeto-db-subnet-group` |
| Public access | **No** |
| Security group | `projeto-sg-rds` |
| Initial database name | `projetodb` |
| Master username | `postgres` |
| Deletion protection | Desativado |
| Endpoint | `projeto-db.xxxxxx.sa-east-1.rds.amazonaws.com` |

> 🔒 **Credenciais:** a senha do usuário `postgres` **não** está documentada aqui por segurança. Guarde-a em um gerenciador de senhas — será necessária na etapa do EC2.

### 3.6 EC2 e Key pair

| Item | Valor |
|---|---|
| Key pair | `projeto-keypair` (RSA, `.pem` salvo localmente) |
| Instância EC2 | `projeto-bastion` (Amazon Linux 2023, `t2.micro`) |
| Subnet | `projeto-subnet-publica` (`10.0.1.0/24`) — `sa-east-1a` |
| IP Privado | `10.0.1.222` |
| IP Público | `18.230.xx.xxx (dinâmico)` |
| Security group | `projeto-sg-ec2` (SSH liberado) |

---

## 4. Etapas realizadas

### ✅ Etapa 1 — Rede
- Criação da VPC `projeto-vpc` (`10.0.0.0/16`).
- Criação das subnets pública e privada.
- Criação e anexação do Internet Gateway.
- Configuração das route tables (pública com rota à internet, privada sem).

**Conceitos:** CIDR, Availability Zone, Internet Gateway, roteamento (`0.0.0.0/0` vs rota `local`).

### ✅ Etapa 2 — Security Groups
- Criação do `projeto-sg-rds` liberando a porta 5432 apenas para recursos dentro da VPC.

**Conceitos:** firewall *stateful*, regras de permissão (não há "negar"), padrão inbound = negar tudo / outbound = permitir tudo.

### ✅ Etapa 3 — RDS PostgreSQL
- Criação do banco `projeto-db` (PostgreSQL, free tier, single-AZ) na subnet privada.
- Criação do `projeto-db-subnet-group` cobrindo 2 AZs (exigência da AWS).
- Configuração de acesso: **sem IP público**.

**Conceitos:** banco gerenciado (RDS), DB subnet group, exigência de cobertura de 2 AZs, Single-AZ vs Multi-AZ.

### ✅ Etapa 4 — EC2 (bastion) e Validação de Conectividade
- Criação do Security Group `projeto-sg-ec2` (SSH liberado para o IP de origem).
- Criação do key pair `projeto-keypair` (arquivo `.pem` salvo localmente e protegido com `icacls`).
- Provisionamento da instância EC2 `projeto-bastion` na `projeto-subnet-publica` com IP público.
- Conexão remota bem-sucedida via SSH a partir do Windows PowerShell.
- Instalação do cliente PostgreSQL (`postgresql15`) via gerenciador de pacotes `dnf`.
- **Validação de comunicação de rede privada:** Conexão autenticada com sucesso no RDS PostgreSQL através do endpoint interno da VPC (`psql -h projeto-db... -U postgres -d postgres`), confirmando o isolamento do banco e a funcionalidade do Bastion Host.

**Conceitos:** EC2, AMI, key pair (acesso SSH seguro), bastion/jump host, resolução de DNS interna da VPC, comunicação inter-subnets.

---

## 5. Conceitos-chave aprendidos

| Conceito | Explicação resumida |
|---|---|
| **CIDR** | Faixa de endereços IP (ex.: `/16` ≈ 65 mil IPs; `/24` = 256 IPs). |
| **Availability Zone (AZ)** | Data center isolado dentro da região. |
| **Internet Gateway (IGW)** | "Porta de saída" da VPC para a internet. |
| **Route table** | Define o caminho do tráfego. A rota `0.0.0.0/0` → IGW é o que torna uma subnet "pública". |
| **Security Group** | Firewall *stateful* anexado aos recursos; só permite, nunca nega. |
| **DB subnet group** | Conjunto de subnets onde o RDS pode ser criado; exige ≥ 2 AZs. |
| **Single-AZ vs Multi-AZ** | Single-AZ = 1 instância (free tier); Multi-AZ = redundância (pago). |
| **RDS** | Banco gerenciado (AWS cuida de patch, backup, failover). |
| **Bastion Host** | Servidor de salto seguro na rede pública usado como ponte para administrar recursos em redes privadas. |

---

## 6. Custos (free tier)

Dentro do free tier (12 meses, contas novas):

- ✅ VPC, subnets, route tables, Internet Gateway — **grátis**.
- ✅ Security Groups — **grátis**.
- ✅ RDS `db.t4g.micro`/`db.t3.micro` Single-AZ — **750h/mês grátis**.
- ✅ EC2 `t2.micro`/`t3.micro` — **750h/mês grátis**.
- ✅ 20 GB de storage + 20 GB de backup — **grátis**.

**Evitamos de propósito (cobram):**
- ❌ NAT Gateway (~US$0,045/hora)
- ❌ Multi-AZ (instâncias extras)
- ❌ Storage autoscaling
- ❌ RDS Proxy, Enhanced Monitoring, DevOps Guru, Database Insights Advanced
- ❌ Backup replication em outra região
- ❌ AWS Secrets Manager (US$0,40/segredo/mês)

---

## 7. Próximos passos

- [x] **Etapa 4.3 — Criar a instância EC2** `projeto-bastion` na subnet pública.
- [x] **Etapa 4.4 — Conectar via SSH** a partir do Windows usando o `.pem`.
- [x] **Etapa 4.5 — Instalar o cliente PostgreSQL** no EC2 e conectar no RDS.
- [ ] **Etapa 5 — Migração do RDS para EC2:** (planejado) hospedar o PostgreSQL em uma instância EC2.
- [ ] **Etapa final — Limpeza (cleanup):** apagar todos os recursos para não gerar custo.

---

## 8. Como limpar (cleanup)

Quando o projeto terminar, apague na seguinte ordem:

1. EC2 `projeto-bastion`: **Terminate instance** (ou **Stop** se for continuar em outro momento).
2. RDS `projeto-db` (confirme que *deletion protection* está desativado).
3. DB subnet group `projeto-db-subnet-group`.
4. Security Groups `projeto-sg-rds` e `projeto-sg-ec2`.
5. Subnets e route tables.
6. Internet Gateway `projeto-igw` (desanexar primeiro).
7. VPC `projeto-vpc`.
8. (Opcional) Apagar a key pair `projeto-keypair`.

> ⚠️ Sempre confira no **Billing/Cost Explorer** se não sobrou nenhum recurso cobrando.

---

*Última atualização: Etapa 4 concluída com sucesso — Conexão validada entre EC2 Bastion e RDS PostgreSQL.*

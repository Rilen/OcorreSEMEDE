# MEMORIA.md

## 🎯 Escopo do Sistema

Projeto: OcorreSEMEDE

Sistema digital do "Livro de Ocorrência Escolar" da SEMEDE.

Objetivo principal:
- Registrar ocorrências escolares de alunos da rede municipal.
- Consultar dados de alunos no banco PostgreSQL interno legado "e-cidade", em modo somente leitura.
- Autenticar Diretores e Coordenadores via servidor LDAP da prefeitura.
- Registrar ocorrências em banco PostgreSQL próprio da aplicação.
- Organizar registros com base em normativas educacionais brasileiras, incluindo MEC/LDB.
- Permitir assinatura dos responsáveis por dois fluxos:
  1. Assinatura digital via link público com token seguro.
  2. Impressão de PDF para assinatura física na secretaria, com posterior upload/captura fotográfica da folha assinada via Cloudinary.

Perfis previstos:
- Diretor
- Coordenador
- Administrador SEMEDE
- Responsável/Pai/Mãe via link público de assinatura

Módulos previstos:
- Autenticação institucional via LDAP
- Consulta de alunos no e-cidade
- Cadastro e gestão de ocorrências
- Geração de PDF da ocorrência
- Link público de assinatura digital
- Upload de comprovante físico assinado
- Histórico/auditoria de ações
- Painel administrativo SEMEDE

## 🛠️ Stack e Variáveis de Ambiente Necessárias

Stack base:
- Next.js 15 com App Router
- TypeScript
- Tailwind CSS
- tRPC
- Prisma
- PostgreSQL
- Zod
- React-PDF
- Cloudinary
- LDAP via ldapts
- Lucide React
- Auth.js/NextAuth ou autenticação customizada baseada em sessão, a decidir no setup de autenticação

Variáveis de ambiente necessárias, apenas chaves:

```env
# Banco principal da aplicação
DATABASE_URL=

# Banco legado e-cidade, somente leitura
ECIDADE_DATABASE_URL=

# LDAP Prefeitura
LDAP_URL=
LDAP_BIND_DN=
LDAP_BIND_PASSWORD=
LDAP_BASE_DN=
LDAP_USER_SEARCH_ATTRIBUTE=
LDAP_TLS_REJECT_UNAUTHORIZED=

# Sessão/autenticação
AUTH_SECRET=
AUTH_URL=

# Tokens públicos
PUBLIC_SIGNATURE_TOKEN_SECRET=

# Cloudinary
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
CLOUDINARY_UPLOAD_FOLDER=

# Aplicação
NEXT_PUBLIC_APP_URL=
NODE_ENV=
```

## 🏗️ Modelagem de Dados (Prisma Schema atualizado)

Versão inicial conceitual. O schema final será criado na etapa de setup do Prisma.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserRole {
  ADMIN_SEMEDE
  DIRETOR
  COORDENADOR
}

enum OccurrenceStatus {
  DRAFT
  PENDING_SIGNATURE
  SIGNED_DIGITALLY
  SIGNED_PHYSICALLY
  CLOSED
  CANCELED
}

enum SignatureMethod {
  DIGITAL
  PHYSICAL
}

enum OccurrenceSeverity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

model User {
  id        String   @id @default(cuid())
  name      String
  email     String?  @unique
  username  String   @unique
  role      UserRole
  schoolId  String?
  school    School?  @relation(fields: [schoolId], references: [id])
  active    Boolean  @default(true)

  occurrencesCreated Occurrence[] @relation("OccurrenceCreatedBy")
  auditLogs           AuditLog[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model School {
  id         String  @id @default(cuid())
  externalId String? @unique
  name       String
  ineCode    String?
  city       String?
  active     Boolean @default(true)

  users       User[]
  occurrences Occurrence[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model StudentSnapshot {
  id                String    @id @default(cuid())
  externalStudentId String
  name              String
  birthDate         DateTime?
  guardianName      String?
  guardianPhone     String?
  guardianEmail     String?
  schoolName        String?
  className         String?
  rawData           Json?

  occurrences Occurrence[]

  createdAt DateTime @default(now())
}

model Occurrence {
  id       String @id @default(cuid())
  protocol String @unique

  schoolId String
  school   School @relation(fields: [schoolId], references: [id])

  studentSnapshotId String
  studentSnapshot   StudentSnapshot @relation(fields: [studentSnapshotId], references: [id])

  createdById String
  createdBy   User @relation("OccurrenceCreatedBy", fields: [createdById], references: [id])

  title       String
  description String
  category    String
  legalBasis  String?
  severity    OccurrenceSeverity @default(LOW)
  status      OccurrenceStatus   @default(DRAFT)

  happenedAt DateTime
  closedAt   DateTime?

  signatures  Signature[]
  attachments Attachment[]
  auditLogs   AuditLog[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Signature {
  id           String     @id @default(cuid())
  occurrenceId String
  occurrence   Occurrence @relation(fields: [occurrenceId], references: [id])

  method         SignatureMethod
  signerName     String
  signerDocument String?
  signerIp       String?
  signedAt       DateTime?

  tokenHash      String?   @unique
  tokenExpiresAt DateTime?

  physicalAttachmentId String?
  physicalAttachment   Attachment? @relation(fields: [physicalAttachmentId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Attachment {
  id           String     @id @default(cuid())
  occurrenceId String
  occurrence   Occurrence @relation(fields: [occurrenceId], references: [id])

  fileName     String
  mimeType     String
  url          String
  cloudinaryId String?
  kind         String

  signaturesUsingAsPhysicalProof Signature[]

  createdAt DateTime @default(now())
}

model AuditLog {
  id     String @id @default(cuid())
  userId String?
  user   User?  @relation(fields: [userId], references: [id])

  occurrenceId String?
  occurrence   Occurrence? @relation(fields: [occurrenceId], references: [id])

  action    String
  metadata  Json?
  ipAddress String?

  createdAt DateTime @default(now())
}
```

Observação:
- O banco e-cidade será tratado como fonte externa somente leitura.
- A aplicação salvará snapshots dos dados do aluno para preservar o histórico da ocorrência mesmo que o cadastro legado mude depois.

## ✅ Etapas Concluídas

- Arquitetura macro definida.
- Nome oficial do projeto definido: OcorreSEMEDE.
- Stack principal definida.
- Fluxos de assinatura definidos:
  - Assinatura digital por token público.
  - Assinatura física via PDF + upload fotográfico.
- Estratégia inicial de memória do projeto definida via `MEMORIA.md`.
- Repositório Git local existente no workspace.

## 🚧 Etapa Atual em Desenvolvimento

Etapa inicial:
- Preparar projeto com o nome OcorreSEMEDE.
- Usar GitHub como repositório remoto assim que houver repositório disponível ou autorização de criação via integração/CLI.
- Ainda não implementar componentes.
- Próxima etapa após confirmação: setup do Prisma.

## 🔜 Próximos Passos / Backlog

1. Criar ou conectar repositório GitHub `OcorreSEMEDE`.
2. Inicializar projeto Next.js 15.
3. Instalar dependências base.
4. Configurar Prisma com PostgreSQL principal.
5. Definir schema Prisma real inicial.
6. Criar migrations.
7. Configurar acesso somente leitura ao banco e-cidade.
8. Definir estratégia de autenticação LDAP.
9. Criar layout autenticado básico.
10. Criar módulo de consulta de aluno.
11. Criar módulo de ocorrência.
12. Criar fluxo de geração de PDF.
13. Criar assinatura digital pública por token.
14. Criar upload de folha assinada via Cloudinary.
15. Criar auditoria de ações.
16. Criar testes mínimos de serviços críticos.

## ⚠️ Decisões de Arquitetura e Avisos

- O banco e-cidade deve ser somente leitura. Não executar migrations nem writes nele.
- Dados de aluno vindos do e-cidade devem ser copiados como snapshot no banco da aplicação no momento da ocorrência.
- Tokens públicos de assinatura devem ser armazenados como hash, nunca em texto puro.
- Links públicos de assinatura devem ter expiração.
- Uploads físicos devem usar Cloudinary com pasta específica por ambiente.
- React-PDF deve ser carregado com cuidado no Next.js App Router. Se houver uso em componente client-side, considerar import dinâmico com `ssr: false`.
- Para captura fotográfica em celular, usar input com `accept="image/*"` e `capture="environment"`.
- LDAP deve ser encapsulado em serviço próprio para evitar acoplamento direto com rotas e componentes.
- Validações de entrada devem usar Zod.
- tRPC deve ser usado para APIs internas autenticadas.
- Rotas públicas de assinatura devem evitar exposição de dados sensíveis além do necessário.
- Nenhum valor real de variável de ambiente deve ser salvo no `MEMORIA.md`.

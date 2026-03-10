# CLAUDE.md - Sistema RFID Industrial

> Documentação técnica completa do Sistema RFID para Embalagem e Conferência Industrial

---

## 📋 Índice

1. [Visão Geral do Sistema](#visão-geral-do-sistema)
2. [Domínio do Negócio](#domínio-do-negócio)
3. [Entidades Principais](#entidades-principais)
4. [Regras de Negócio](#regras-de-negócio)
5. [Fluxos de Trabalho](#fluxos-de-trabalho)
6. [Nomenclaturas e Convenções](#nomenclaturas-e-convenções)
7. [Arquitetura do Sistema](#arquitetura-do-sistema)
8. [APIs e Integrações](#apis-e-integrações)
9. [Modelo de Dados](#modelo-de-dados)
10. [Casos de Uso](#casos-de-uso)

---

## 1. Visão Geral do Sistema

### Propósito
Sistema de gestão de embalagem e conferência de pedidos industriais utilizando tecnologia RFID (Radio-Frequency Identification) para automatizar, rastrear e validar processos de expedição.

### Problema que Resolve
- **Erro humano**: Reduz erros de separação/conferência manual de 3-5% para <0.5%
- **Tempo de processo**: Reduz tempo de conferência de 5-10min para 30-60s por pedido
- **Rastreabilidade**: Cria trilha de auditoria completa de todas as operações
- **Integração**: Automatiza comunicação com ERP (Focco) para liberação de faturamento

### Escopo Inicial (Fase 1)
- ✅ Embalagem de volumes com RFID
- ✅ Conferência automatizada de pedidos
- ✅ Integração com ERP Focco
- ✅ Rastreabilidade básica
- ✅ Auditoria de operações

---

## 2. Domínio do Negócio

### Contexto Industrial
O sistema opera em ambiente de **armazém/centro de distribuição industrial**, onde:
- Produtos são **separados** de pedidos de clientes
- Produtos são **embalados** em volumes para transporte
- Volumes são **conferidos** antes da expedição
- Conferência aprovada **libera faturamento** no ERP

### Atores Principais

#### 1. **Operador de Embalagem (Packer)**
- **Função**: Montar volumes com produtos do pedido
- **Responsabilidades**:
  - Iniciar novo volume para um pedido
  - Ler tags RFID dos produtos
  - Adicionar produtos ao volume
  - Finalizar volume e emitir etiqueta RFID
- **Ferramentas**: Leitor RFID portátil ou fixo, impressora de etiquetas
- **Métricas**: Volumes embalados/hora, taxa de erro

#### 2. **Conferente (Checker)**
- **Função**: Validar que todos os volumes do pedido estão corretos
- **Responsabilidades**:
  - Iniciar sessão de conferência para um pedido
  - Ler tags RFID dos volumes
  - Validar correspondência com esperado
  - Identificar e reportar divergências
  - Finalizar conferência e enviar ao ERP
- **Ferramentas**: Portal/túnel RFID, coletor de dados
- **Métricas**: Pedidos conferidos/hora, divergências encontradas

#### 3. **Supervisor**
- **Função**: Gestão e controle de qualidade
- **Responsabilidades**:
  - Monitorar performance da equipe
  - Analisar divergências
  - Gerar relatórios
  - Aprovar exceções
- **Ferramentas**: Dashboard web, relatórios

#### 4. **Sistema ERP Focco**
- **Função**: Sistema de gestão empresarial integrado
- **Responsabilidades**:
  - Fornecer dados de pedidos
  - Receber confirmações de conferência
  - Liberar faturamento
  - Atualizar status de pedidos

### Glossário do Domínio

| Termo | Definição | Sinônimos |
|-------|-----------|-----------|
| **Pedido** | Solicitação de compra de um cliente contendo lista de produtos | Order, Purchase Order |
| **Volume** | Caixa/embalagem física contendo produtos de um pedido | Box, Package, Parcel |
| **Item** | Produto individual dentro de um volume | Product, SKU |
| **Tag RFID** | Chip de radiofrequência que armazena identificação única | RFID Tag, Smart Label |
| **Leitura** | Ato de capturar dados de uma tag RFID | Scan, Read |
| **Gravação** | Ato de escrever dados em uma tag RFID | Write, Encode |
| **Conferência** | Processo de validação de volumes vs. esperado | Checking, Verification |
| **Divergência** | Diferença entre esperado e conferido | Discrepancy, Variance |
| **Embalagem** | Processo de montagem de volumes | Packing, Packaging |
| **Expedição** | Envio de volumes para transporte | Shipping, Dispatch |
| **Sessão de Conferência** | Período de conferência de um pedido específico | Checking Session |

---

## 3. Entidades Principais

### 3.1 User (Usuário)
**Descrição**: Pessoa que utiliza o sistema

**Atributos**:
- `id` (UUID): Identificador único
- `username` (String): Login do usuário
- `passwordHash` (String): Senha criptografada
- `fullName` (String): Nome completo
- `role` (Enum): Perfil de acesso
  - `ADMIN`: Acesso total
  - `SUPERVISOR`: Gestão e relatórios
  - `PACKER`: Embalagem
  - `CHECKER`: Conferência
- `active` (Boolean): Se está ativo
- `createdAt` (DateTime): Data de criação
- `updatedAt` (DateTime): Última atualização

**Regras**:
- Username deve ser único
- Senha mínima de 8 caracteres (em produção)
- Apenas administradores podem criar usuários

---

### 3.2 Order (Pedido)
**Descrição**: Solicitação de compra de um cliente

**Atributos**:
- `id` (UUID): Identificador único
- `orderNumber` (String): Número do pedido (ex: "PED-45823")
- `customerName` (String): Nome do cliente
- `customerId` (String): Código do cliente no ERP
- `totalItems` (Integer): Quantidade total de produtos
- `status` (Enum): Estado atual
  - `PENDING`: Aguardando embalagem
  - `PACKING`: Em embalagem
  - `PACKED`: Embalado
  - `CHECKING`: Em conferência
  - `CHECKED`: Conferido
  - `SENT_TO_FOCCO`: Enviado ao ERP
  - `COMPLETED`: Completo
  - `CANCELLED`: Cancelado
- `syncedFromFocco` (Boolean): Se veio do Focco
- `foccoData` (JSON): Dados brutos do Focco
- `createdAt` (DateTime): Data de criação
- `updatedAt` (DateTime): Última atualização

**Relacionamentos**:
- Possui múltiplos `Volume`
- Possui múltiplas `CheckingSession`
- Referenciado em `IntegrationLog`

**Regras**:
- OrderNumber deve ser único
- Não pode ser deletado após conferência iniciada
- Status deve seguir fluxo sequencial

---

### 3.3 Volume (Volume/Caixa)
**Descrição**: Embalagem física contendo produtos

**Atributos**:
- `id` (UUID): Identificador único
- `rfidCode` (String): Código RFID único (ex: "RFID20260212001")
- `volumeNumber` (String): Número sequencial no pedido ("001", "002"...)
- `orderId` (UUID): Referência ao pedido
- `totalItems` (Integer): Quantidade de itens únicos
- `totalQuantity` (Integer): Quantidade total de peças
- `status` (Enum): Estado atual
  - `IN_PROGRESS`: Em montagem
  - `PACKED`: Finalizado/embalado
  - `CHECKED`: Conferido
  - `CANCELLED`: Cancelado
- `packedById` (UUID): Operador que embalou
- `packedAt` (DateTime): Quando foi embalado
- `checkedById` (UUID): Conferente que validou
- `checkedAt` (DateTime): Quando foi conferido
- `foccoSentAt` (DateTime): Quando foi enviado ao Focco
- `createdAt` (DateTime): Data de criação
- `updatedAt` (DateTime): Última atualização

**Relacionamentos**:
- Pertence a um `Order`
- Criado por um `User` (packer)
- Conferido por um `User` (checker)
- Contém múltiplos `VolumeItem`
- Pode ter `VolumeMetadata`

**Regras**:
- RfidCode deve ser único em todo o sistema
- Combinação (orderId + volumeNumber) deve ser única
- Não pode ser editado após status PACKED
- Deve ter ao menos 1 item para ser finalizado

---

### 3.4 VolumeItem (Item do Volume)
**Descrição**: Produto individual dentro de um volume

**Atributos**:
- `id` (UUID): Identificador único
- `volumeId` (UUID): Referência ao volume
- `productCode` (String): Código do produto (ex: "PROD-1001")
- `productName` (String): Descrição do produto
- `quantity` (Integer): Quantidade de unidades
- `rfidTag` (String): Tag RFID do produto individual (opcional)
- `createdAt` (DateTime): Quando foi adicionado

**Relacionamentos**:
- Pertence a um `Volume`

**Regras**:
- ProductCode deve existir no catálogo
- Quantity deve ser > 0
- Não pode haver itens duplicados (mesmo productCode) no mesmo volume

---

### 3.5 CheckingSession (Sessão de Conferência)
**Descrição**: Processo de validação de um pedido

**Atributos**:
- `id` (UUID): Identificador único
- `orderId` (UUID): Referência ao pedido
- `checkedById` (UUID): Conferente responsável
- `startedAt` (DateTime): Início da conferência
- `completedAt` (DateTime): Fim da conferência
- `status` (Enum): Estado atual
  - `IN_PROGRESS`: Em andamento
  - `COMPLETED`: Completa sem divergências
  - `COMPLETED_WITH_DIVERGENCE`: Completa com divergências
  - `CANCELLED`: Cancelada
- `totalExpected` (Integer): Total de volumes esperados
- `totalChecked` (Integer): Total de volumes conferidos
- `hasDivergence` (Boolean): Se há divergências
- `divergenceNotes` (String): Observações sobre divergências
- `foccoSent` (Boolean): Se foi enviado ao Focco
- `foccoResponseJson` (JSON): Resposta do Focco

**Relacionamentos**:
- Pertence a um `Order`
- Criada por um `User` (checker)
- Contém múltiplos `CheckingVolume`

**Regras**:
- Apenas uma sessão ativa por pedido
- Não pode ser completada com totalChecked < totalExpected (exceto com justificativa)
- Divergências devem ser documentadas

---

### 3.6 CheckingVolume (Volume Conferido)
**Descrição**: Registro de conferência de um volume específico

**Atributos**:
- `id` (UUID): Identificador único
- `sessionId` (UUID): Referência à sessão
- `volumeId` (UUID): Referência ao volume
- `checkedAt` (DateTime): Quando foi conferido
- `status` (Enum): Resultado da conferência
  - `MATCHED`: Conferido OK
  - `DIVERGENT`: Divergência encontrada
  - `EXTRA`: Volume extra (não esperado)
- `notes` (String): Observações

**Relacionamentos**:
- Pertence a uma `CheckingSession`
- Referencia um `Volume`

**Regras**:
- Não pode conferir mesmo volume duas vezes na mesma sessão
- Volume deve pertencer ao pedido da sessão

---

### 3.7 AuditLog (Log de Auditoria)
**Descrição**: Registro de todas as operações do sistema

**Atributos**:
- `id` (UUID): Identificador único
- `timestamp` (DateTime): Momento da operação
- `eventType` (Enum): Tipo de evento
  - `USER_LOGIN`, `USER_LOGOUT`
  - `VOLUME_CREATED`, `VOLUME_UPDATED`, `VOLUME_FINALIZED`, `VOLUME_CANCELLED`
  - `ITEM_ADDED`, `ITEM_REMOVED`
  - `CHECKING_STARTED`, `CHECKING_COMPLETED`
  - `FOCCO_SENT`, `SYSTEM_ERROR`
- `entityType` (String): Tipo de entidade (Volume, Order, etc)
- `entityId` (String): ID da entidade
- `userId` (UUID): Usuário que executou
- `action` (String): Descrição da ação
- `oldValuesJson` (JSON): Valores antes da mudança
- `newValuesJson` (JSON): Valores após a mudança
- `metadataJson` (JSON): Dados adicionais
- `ipAddress` (String): IP do cliente
- `userAgent` (String): Browser/dispositivo

**Regras**:
- Logs são imutáveis (não podem ser editados/deletados)
- Retenção mínima de 2 anos
- Todos os eventos críticos devem gerar log

---

### 3.8 IntegrationLog (Log de Integração)
**Descrição**: Registro de comunicações com sistemas externos

**Atributos**:
- `id` (UUID): Identificador único
- `timestamp` (DateTime): Momento da integração
- `integrationType` (Enum): Tipo de integração
  - `FOCCO_ORDER_FETCH`: Busca de pedido no Focco
  - `FOCCO_SEND_CHECKING`: Envio de conferência ao Focco
  - `RFID_READ`: Leitura de RFID
  - `RFID_WRITE`: Gravação de RFID
- `orderId` (UUID): Pedido relacionado (se aplicável)
- `requestPayload` (JSON): Dados enviados
- `responseStatus` (Integer): HTTP status code
- `responseBody` (JSON): Resposta recebida
- `errorMessage` (String): Mensagem de erro (se houver)
- `retryCount` (Integer): Tentativas realizadas
- `success` (Boolean): Se foi bem-sucedido

**Regras**:
- Registra todas as tentativas (sucesso e falha)
- Usado para troubleshooting e SLA

---

## 4. Regras de Negócio

### 4.1 Embalagem

#### RN-001: Iniciar Volume
- **Regra**: Apenas usuários com role PACKER ou ADMIN podem iniciar volumes
- **Validação**: Verificar se pedido existe e está em status PENDING ou PACKING
- **Ação**: Criar volume com status IN_PROGRESS e volumeNumber sequencial

#### RN-002: Adicionar Item ao Volume
- **Regra**: Item deve pertencer ao pedido sendo embalado
- **Validação**: 
  - ProductCode existe no pedido
  - Item não foi adicionado anteriormente no mesmo volume
  - Volume está em status IN_PROGRESS
- **Ação**: Inserir VolumeItem e atualizar contadores do Volume

#### RN-003: Remover Item do Volume
- **Regra**: Apenas possível se volume está IN_PROGRESS
- **Validação**: Item existe no volume
- **Ação**: Deletar VolumeItem, atualizar contadores, registrar auditoria

#### RN-004: Finalizar Volume
- **Regra**: Volume deve ter ao menos 1 item
- **Validação**: 
  - totalItems > 0
  - Status é IN_PROGRESS
- **Ação**:
  - Gerar rfidCode único
  - Gravar chip RFID
  - Validar gravação
  - Imprimir etiqueta
  - Atualizar status para PACKED
  - Registrar packedBy e packedAt

#### RN-005: Cancelar Volume
- **Regra**: Apenas volumes IN_PROGRESS podem ser cancelados
- **Validação**: Volume não foi conferido
- **Ação**: 
  - Atualizar status para CANCELLED
  - Registrar motivo (se fornecido)
  - Manter histórico (não deletar)

---

### 4.2 Conferência

#### RN-006: Iniciar Conferência
- **Regra**: Apenas usuários com role CHECKER, SUPERVISOR ou ADMIN
- **Validação**:
  - Pedido existe e tem volumes embalados
  - Não há sessão ativa para o pedido
- **Ação**:
  - Criar CheckingSession com status IN_PROGRESS
  - Registrar totalExpected (quantidade de volumes PACKED)

#### RN-007: Conferir Volume
- **Regra**: Volume deve pertencer ao pedido em conferência
- **Validação**:
  - RFID lido corresponde a um volume existente
  - Volume pertence ao orderId da sessão
  - Volume não foi conferido anteriormente nesta sessão
  - Volume está com status PACKED
- **Ação**:
  - Criar CheckingVolume com status MATCHED
  - Incrementar totalChecked
  - Atualizar volume.checkedAt e checkedBy

#### RN-008: Detectar Divergências
- **Regra**: Sistema detecta automaticamente divergências
- **Tipos de Divergência**:
  - **Volume Faltante**: Esperado mas não conferido
  - **Volume Extra**: Conferido mas não esperado
  - **Quantidade Diferente**: Total de itens não confere
- **Ação**:
  - Marcar hasDivergence = true
  - Criar CheckingVolume com status DIVERGENT ou EXTRA
  - Exigir divergenceNotes antes de completar

#### RN-009: Finalizar Conferência
- **Regra**: Conferência só pode ser finalizada se:
  - totalChecked = totalExpected (100% conferido) OU
  - Divergências foram documentadas e justificadas
- **Validação**: Status é IN_PROGRESS
- **Ação**:
  - Atualizar status (COMPLETED ou COMPLETED_WITH_DIVERGENCE)
  - Registrar completedAt
  - Enfileirar envio ao Focco

#### RN-010: Reiniciar Conferência
- **Regra**: Apenas conferências IN_PROGRESS podem ser reiniciadas
- **Validação**: Confirmação do usuário
- **Ação**:
  - Deletar todos CheckingVolume da sessão
  - Resetar totalChecked para 0
  - Manter sessionId e startedAt (para auditoria)

---

### 4.3 Integração com Focco

#### RN-011: Enviar Conferência ao Focco
- **Regra**: Apenas conferências COMPLETED podem ser enviadas
- **Payload Mínimo**:
  - orderNumber
  - conferente (nome e ID)
  - dataHora
  - volumes[] (rfid, numero, itens)
  - status (APROVADO ou DIVERGENTE)
- **Retry**: Até 3 tentativas com exponential backoff
- **Timeout**: 10 segundos máximo
- **Ação em Sucesso**:
  - Marcar foccoSent = true
  - Salvar foccoResponseJson
  - Atualizar order.status para SENT_TO_FOCCO
  - Registrar IntegrationLog com success = true
- **Ação em Falha**:
  - Adicionar à fila de retry
  - Registrar IntegrationLog com success = false
  - Notificar supervisor após 3 falhas

#### RN-012: Buscar Pedido no Focco
- **Regra**: Sincronização de pedidos do ERP
- **Frequência**: Sob demanda ou agendada
- **Validação**: OrderNumber existe no Focco
- **Ação**:
  - Criar ou atualizar Order local
  - Marcar syncedFromFocco = true
  - Salvar foccoData completo

---

### 4.4 RFID

#### RN-013: Gerar Código RFID
- **Formato**: `RFID + AAAAMMDD + Sequencial(3 dígitos)`
- **Exemplo**: `RFID20260212001`
- **Regra**: Deve ser único globalmente
- **Validação**: Verificar não-duplicação antes de gravar chip

#### RN-014: Gravar Chip RFID
- **Dados Gravados**:
  - rfidCode
  - volumeNumber
  - orderNumber
  - packedAt
  - operatorId
  - totalItems
- **Validação**: Leitura imediata após gravação para confirmar
- **Retry**: Até 3 tentativas em caso de falha
- **Ação em Falha**: Bloquear emissão de etiqueta, alertar operador

#### RN-015: Leitura de RFID
- **Timeout**: 1 segundo máximo
- **Validação**: Dados decodificados corretamente
- **Feedback**: Sonoro + visual imediato
- **Log**: Todas as leituras são registradas (auditoria)

---

### 4.5 Segurança e Auditoria

#### RN-016: Autenticação
- **Regra**: Todas as operações requerem usuário autenticado
- **Sessão**: Expira após 8 horas de inatividade
- **Token**: JWT com claims: userId, username, role
- **Refresh**: Permitido dentro de 24h

#### RN-017: Autorização
- **ADMIN**: Acesso total
- **SUPERVISOR**: Leitura completa + aprovação de exceções
- **PACKER**: Apenas embalagem
- **CHECKER**: Apenas conferência
- **Validação**: Antes de cada operação crítica

#### RN-018: Auditoria Obrigatória
- **Eventos que SEMPRE geram log**:
  - Login/Logout
  - Criação/Finalização de Volume
  - Adição/Remoção de Item
  - Início/Fim de Conferência
  - Envio ao Focco
  - Erros de sistema
- **Dados Capturados**: Who, What, When, Where, Why (se aplicável)

---

## 5. Fluxos de Trabalho

### 5.1 Fluxo de Embalagem Completo

```
1. Operador faz login (role: PACKER)
   ↓
2. Sistema exibe pedidos pendentes de embalagem
   ↓
3. Operador seleciona pedido PED-XXXXX
   ↓
4. Sistema cria Volume #001 (status: IN_PROGRESS)
   ↓
5. Operador aproxima produto do leitor RFID
   ↓
6. Sistema lê tag RFID do produto
   ↓
7. Sistema valida:
   - Produto pertence ao pedido? ✓
   - Produto já foi adicionado? ✗
   ↓
8. Sistema adiciona produto ao volume
   ↓
9. Sistema exibe confirmação visual/sonora
   ↓
10. Operador repete passos 5-9 até completar volume
   ↓
11. Operador clica "Finalizar Volume"
   ↓
12. Sistema valida:
   - Volume tem itens? ✓
   ↓
13. Sistema gera código RFID único
   ↓
14. Sistema grava dados no chip RFID
   ↓
15. Sistema valida gravação (re-leitura)
   ↓
16. Sistema imprime etiqueta física
   ↓
17. Sistema atualiza Volume:
   - status → PACKED
   - rfidCode → gerado
   - packedAt → timestamp
   - packedBy → usuário atual
   ↓
18. Sistema registra em AuditLog
   ↓
19. Sistema pergunta: "Novo volume ou finalizar pedido?"
   ↓
   [Novo Volume] → volta ao passo 4 (Volume #002)
   [Finalizar] → pedido marcado como PACKED
```

**Exceções**:
- Produto não pertence ao pedido → Alerta + não adiciona
- Produto duplicado → Alerta + não adiciona
- Falha na gravação RFID → Retry (3x) → Alerta supervisor
- Falha na impressão → Permitir reimpressão

---

### 5.2 Fluxo de Conferência Completo

```
1. Conferente faz login (role: CHECKER)
   ↓
2. Sistema exibe pedidos prontos para conferência
   ↓
3. Conferente seleciona pedido PED-XXXXX
   ↓
4. Sistema cria CheckingSession (status: IN_PROGRESS)
   ↓
5. Sistema busca volumes esperados (PACKED) do pedido
   ↓
6. Sistema exibe comparação:
   - Coluna "Esperado": 5 volumes
   - Coluna "Conferido": 0 volumes
   ↓
7. Conferente passa volume pelo portal/leitor RFID
   ↓
8. Sistema lê código RFID do volume
   ↓
9. Sistema valida:
   - Volume existe? ✓
   - Volume pertence a este pedido? ✓
   - Volume já foi conferido? ✗
   ↓
10. Sistema registra conferência:
   - Cria CheckingVolume (status: MATCHED)
   - Incrementa totalChecked
   - Move volume para coluna "Conferido"
   - Atualiza barra de progresso
   ↓
11. Sistema emite confirmação visual/sonora
   ↓
12. Conferente repete passos 7-11 para todos volumes
   ↓
13. Sistema detecta: totalChecked = totalExpected (100%)
   ↓
14. Sistema habilita botão "Enviar para Faturamento"
   ↓
15. Conferente clica "Enviar para Faturamento"
   ↓
16. Sistema valida:
   - Conferência completa? ✓
   - Sem divergências? ✓ (ou divergências documentadas)
   ↓
17. Sistema finaliza sessão:
   - status → COMPLETED
   - completedAt → timestamp
   ↓
18. Sistema enfileira job: envio-focco
   ↓
19. Worker processa job:
   - Formata payload conforme API Focco
   - Envia POST /api/focco/conferencias
   - Aguarda resposta (timeout 10s)
   ↓
20. [Sucesso] 
   - Marca foccoSent = true
   - Salva foccoResponseJson
   - Atualiza Order.status → SENT_TO_FOCCO
   - Notifica conferente (WebSocket)
   - Registra IntegrationLog (success: true)
   ↓
21. [Falha]
   - Incrementa retryCount
   - Se retry < 3: Re-enfileira (delay exponencial)
   - Se retry = 3: Dead Letter Queue + alerta supervisor
   - Registra IntegrationLog (success: false)
```

**Exceções**:
- Volume não pertence ao pedido → Alerta + não adiciona
- Volume duplicado → Alerta + não incrementa
- Volume extra (não esperado) → Marca como DIVERGENT
- Volume faltante → Exige justificativa antes de completar
- Falha comunicação Focco → Retry automático + fila

---

### 5.3 Fluxo de Tratamento de Divergências

```
1. Durante conferência, sistema detecta divergência
   ↓
2. Tipos possíveis:
   - Volume esperado não foi conferido
   - Volume conferido não era esperado
   - Quantidade total não confere
   ↓
3. Sistema:
   - Marca hasDivergence = true
   - Destaca visualmente divergências (cor vermelha)
   - Incrementa contador de divergências
   ↓
4. Conferente pode:
   [a] Continuar conferindo (divergência pendente)
   [b] Reportar divergência imediatamente
   ↓
5. Ao clicar "Reportar Divergência":
   - Modal solicita:
     * Tipo de divergência
     * Descrição detalhada
     * Foto (opcional)
     * Ação tomada
   ↓
6. Sistema salva em divergenceNotes
   ↓
7. Sistema notifica supervisor (email/push)
   ↓
8. Supervisor revisa:
   - Aprova e permite finalizar conferência
   - Rejeita e solicita recontagem
   ↓
9. Se aprovado:
   - Conferência pode ser finalizada
   - Status: COMPLETED_WITH_DIVERGENCE
   - Payload ao Focco inclui divergências
   ↓
10. Sistema registra em AuditLog:
   - Divergência reportada
   - Supervisor que aprovou
   - Ações corretivas
```

---

## 6. Nomenclaturas e Convenções

### 6.1 Nomenclatura de Código

#### Variáveis e Funções (JavaScript/TypeScript)
```javascript
// camelCase para variáveis e funções
const volumeNumber = '001';
const totalItems = 5;
function addItemToVolume(item) { }
function simulateRfidRead() { }

// PascalCase para classes e componentes React
class VolumeService { }
function PackingStation() { }

// UPPER_CASE para constantes
const MAX_RETRY_ATTEMPTS = 3;
const RFID_TIMEOUT_MS = 1000;
```

#### Banco de Dados (PostgreSQL)
```sql
-- snake_case para tabelas e colunas
CREATE TABLE volume_items (
  id UUID PRIMARY KEY,
  volume_id UUID NOT NULL,
  product_code VARCHAR(50),
  created_at TIMESTAMP
);

-- índices começam com idx_
CREATE INDEX idx_volumes_rfid_code ON volumes(rfid_code);
```

#### API Endpoints
```
// kebab-case para URLs
GET  /api/packing/volumes/:id
POST /api/checking/sessions/start
GET  /api/traceability/audit-logs
```

---

### 6.2 Padrões de Código RFID

#### Formato de Código RFID
```
Padrão: RFID + AAAAMMDD + NNN
Exemplo: RFID20260212001

Onde:
- RFID: Prefixo fixo
- AAAA: Ano (4 dígitos)
- MM: Mês (2 dígitos, 01-12)
- DD: Dia (2 dígitos, 01-31)
- NNN: Sequencial do dia (3 dígitos, 001-999)

Validação Regex: ^RFID\d{11}$
```

#### Formato de Número de Pedido
```
Padrão: PED-NNNNN
Exemplo: PED-45823

Onde:
- PED: Prefixo fixo
- NNNNN: Número sequencial (5 dígitos)

Validação Regex: ^PED-\d{5}$
```

#### Formato de Número de Volume
```
Padrão: NNN
Exemplo: 001, 002, 015

Onde:
- NNN: Sequencial do pedido (3 dígitos, 001-999)

Formatação: String.padStart(3, '0')
```

---

### 6.3 Códigos de Produto

```
Formato: PROD-NNNN
Exemplo: PROD-1001

Categorias (primeiro dígito):
- 1xxx: Fixadores (parafusos, porcas, arruelas)
- 2xxx: Componentes eletrônicos
- 3xxx: Ferramentas
- 4xxx: Materiais de construção
- 5xxx: Consumíveis
```

---

### 6.4 Mensagens de Sistema

#### Mensagens de Sucesso (✅)
```
"✅ Item adicionado com sucesso!"
"✅ Volume finalizado e etiqueta emitida!"
"✅ Conferência completa!"
"✅ Dados enviados ao Focco com sucesso!"
```

#### Mensagens de Alerta (⚠️)
```
"⚠️ Item já adicionado ao volume!"
"⚠️ Todos os volumes já foram conferidos!"
"⚠️ Divergência detectada: volume faltante"
```

#### Mensagens de Erro (❌)
```
"❌ Produto não pertence a este pedido"
"❌ Falha na gravação RFID - tente novamente"
"❌ Não é possível enviar: conferência incompleta"
"❌ Erro de comunicação com Focco"
```

#### Mensagens Informativas (ℹ️)
```
"📦 Novo volume iniciado"
"🔄 Conferência reiniciada"
"⏳ Enviando dados para o Focco..."
```

---

### 6.5 Cores do Sistema

```css
/* Cores Primárias */
--primary: #667eea;       /* Roxo - Ações principais */
--secondary: #11998e;     /* Verde água - Conferência */

/* Cores Semânticas */
--success: #28a745;       /* Verde - Sucesso */
--danger: #dc3545;        /* Vermelho - Erro/Perigo */
--warning: #ffc107;       /* Amarelo - Alerta */
--info: #17a2b8;          /* Azul - Informação */

/* Cores Neutras */
--light: #f8f9fa;         /* Fundo claro */
--dark: #343a40;          /* Texto escuro */
--border: #dee2e6;        /* Bordas */
```

---

## 7. Arquitetura do Sistema

### 7.1 Visão Geral

```
┌─────────────────────────────────────────────────┐
│              CAMADA DE APRESENTAÇÃO              │
├─────────────────────────────────────────────────┤
│  Web App (React) | Tablet | Coletor de Dados   │
└──────────────────┬──────────────────────────────┘
                   │ HTTPS / WebSocket
┌──────────────────▼──────────────────────────────┐
│              API GATEWAY (NGINX)                 │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│           CAMADA DE APLICAÇÃO                    │
├─────────────────────────────────────────────────┤
│  Auth API | Packing API | Checking API          │
│  Node.js + Express + TypeScript                  │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│           CAMADA DE SERVIÇOS                     │
├─────────────────────────────────────────────────┤
│  RFID Service | Focco Service | Queue Service   │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│           CAMADA DE DADOS                        │
├─────────────────────────────────────────────────┤
│  PostgreSQL | Redis | File Storage              │
└─────────────────────────────────────────────────┘
```

### 7.2 Stack Tecnológica

**Frontend**:
- React 18+ (Hooks, Suspense)
- TypeScript
- Zustand (state management)
- React Query (data fetching)
- Tailwind CSS (styling)
- Socket.io-client (real-time)

**Backend**:
- Node.js 20+ LTS
- Express.js
- TypeScript
- Prisma ORM
- Bull (job queue)
- Winston (logging)
- JWT (authentication)

**Database**:
- PostgreSQL 15+ (primary)
- Redis 7+ (cache, queue, pub/sub)

**Infrastructure**:
- Docker + Docker Compose
- Kubernetes (production)
- NGINX (reverse proxy)
- Prometheus + Grafana (monitoring)

---

## 8. APIs e Integrações

### 8.1 API REST Interna

Base URL: `https://api.rfid-system.com/v1`

**Autenticação**: Bearer Token (JWT)
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Endpoints Principais**:
```
Authentication:
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh

Packing:
POST   /packing/volumes/start
POST   /packing/volumes/:id/items
DELETE /packing/volumes/:id/items/:itemId
POST   /packing/volumes/:id/finalize
GET    /packing/volumes/:id

Checking:
POST   /checking/sessions/start
POST   /checking/sessions/:id/scan
POST   /checking/sessions/:id/complete
GET    /checking/sessions/:id

Traceability:
GET    /traceability/volumes/:rfidCode/history
GET    /traceability/orders/:orderNumber/volumes
```

**Formato de Resposta**:
```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-03-09T10:30:00Z"
}
```

**Formato de Erro**:
```json
{
  "success": false,
  "error": {
    "code": "PRODUCT_NOT_IN_ORDER",
    "message": "O produto não pertence a este pedido",
    "details": { ... }
  },
  "timestamp": "2026-03-09T10:30:00Z"
}
```

---

### 8.2 API Focco (Externa)

**Base URL**: `https://api.focco.com.br/v1`

**Autenticação**: Bearer Token
```
Authorization: Bearer {FOCCO_API_TOKEN}
```

**Endpoints Utilizados**:

#### Buscar Pedido
```
GET /pedidos/{orderNumber}

Response:
{
  "pedido": "PED-45823",
  "cliente": {
    "id": "CLI-001",
    "nome": "Indústria ABC Ltda"
  },
  "itens": [
    {
      "codigo": "PROD-1001",
      "descricao": "Parafuso M8",
      "quantidade": 100
    }
  ],
  "status": "SEPARADO"
}
```

#### Enviar Conferência
```
POST /conferencias

Payload:
{
  "pedido": "PED-45823",
  "conferente": {
    "id": "user-456",
    "nome": "Maria Santos"
  },
  "dataHoraConferencia": "2026-03-09T18:15:00-03:00",
  "volumes": [
    {
      "rfid": "RFID20260309001",
      "numero": "001",
      "quantidadeItens": 3,
      "quantidadeTotal": 450
    }
  ],
  "status": "APROVADO",
  "observacoes": null
}

Response:
{
  "sucesso": true,
  "conferencia": {
    "id": "CFR-123456",
    "pedido": "PED-45823",
    "status": "LIBERADO_FATURAMENTO"
  }
}
```

---

### 8.3 WebSocket Events

**Namespace**: `/rfid`

**Client → Server**:
```javascript
// Subscrever atualizações
socket.emit('subscribe:volume', { volumeId: 'uuid' });
socket.emit('subscribe:session', { sessionId: 'uuid' });
```

**Server → Client**:
```javascript
// Leitura RFID
socket.on('rfid:read-success', (data) => {
  // data: { volumeId, rfidCode, timestamp }
});

// Integração Focco
socket.on('focco:integration-complete', (data) => {
  // data: { sessionId, success, foccoId }
});

// Divergência detectada
socket.on('checking:divergence', (data) => {
  // data: { sessionId, type, details }
});
```

---

## 9. Modelo de Dados

### 9.1 Diagrama ER Simplificado

```
┌──────────┐         ┌──────────┐         ┌───────────────┐
│   User   │────────>│  Volume  │<────────│ VolumeItem    │
└──────────┘    1:N  └──────────┘    1:N  └───────────────┘
                          ^
                          │ N:1
                          │
                     ┌────┴─────┐
                     │  Order   │
                     └────┬─────┘
                          │ 1:N
                          v
                ┌─────────────────────┐
                │ CheckingSession     │
                └─────────┬───────────┘
                          │ 1:N
                          v
                ┌─────────────────────┐
                │ CheckingVolume      │
                └─────────────────────┘
```

### 9.2 Chaves Estrangeiras

```sql
-- Volume
FOREIGN KEY (order_id) REFERENCES orders(id)
FOREIGN KEY (packed_by_id) REFERENCES users(id)
FOREIGN KEY (checked_by_id) REFERENCES users(id)

-- VolumeItem
FOREIGN KEY (volume_id) REFERENCES volumes(id) ON DELETE CASCADE

-- CheckingSession
FOREIGN KEY (order_id) REFERENCES orders(id)
FOREIGN KEY (checked_by_id) REFERENCES users(id)

-- CheckingVolume
FOREIGN KEY (session_id) REFERENCES checking_sessions(id) ON DELETE CASCADE
FOREIGN KEY (volume_id) REFERENCES volumes(id)

-- AuditLog
FOREIGN KEY (user_id) REFERENCES users(id)

-- IntegrationLog
FOREIGN KEY (order_id) REFERENCES orders(id)
```

---

## 10. Casos de Uso

### Caso de Uso 1: Embalar Pedido Simples

**Ator**: Operador de Embalagem
**Pré-condição**: Operador autenticado, pedido disponível
**Fluxo**:
1. Operador seleciona pedido PED-12345
2. Sistema cria Volume #001
3. Operador lê 5 produtos diferentes
4. Sistema adiciona cada produto ao volume
5. Operador clica "Finalizar Volume"
6. Sistema gera RFID, grava chip, imprime etiqueta
7. Volume #001 completo
**Pós-condição**: Volume pronto para conferência

---

### Caso de Uso 2: Conferir Pedido com Divergência

**Ator**: Conferente
**Pré-condição**: Pedido embalado (5 volumes)
**Fluxo**:
1. Conferente inicia conferência de PED-12345
2. Sistema mostra: Esperado 5 volumes
3. Conferente lê 4 volumes via RFID
4. Sistema detecta: 1 volume faltante
5. Conferente clica "Reportar Divergência"
6. Conferente informa: "Volume extraviado no armazém"
7. Supervisor aprova exceção
8. Conferente finaliza conferência
9. Sistema envia ao Focco com observação
**Pós-condição**: Pedido liberado com divergência documentada

---

### Caso de Uso 3: Rastrear Volume

**Ator**: Supervisor
**Pré-condição**: Volume foi embalado e conferido
**Fluxo**:
1. Supervisor busca por RFID: RFID20260309001
2. Sistema exibe histórico completo:
   - Embalado por João em 09/03 14:30
   - Itens: PROD-1001 (100un), PROD-1002 (200un)
   - Conferido por Maria em 09/03 15:45
   - Enviado ao Focco em 09/03 15:46
   - Status atual: LIBERADO_FATURAMENTO
**Pós-condição**: Supervisor tem rastreabilidade completa

---

## Conclusão

Este documento serve como **fonte única de verdade** sobre o domínio, regras e funcionamento do Sistema RFID Industrial. Deve ser atualizado sempre que houver mudanças no negócio ou requisitos.

**Versão**: 1.0
**Última Atualização**: 09/03/2026
**Responsável**: Gerente de Produtos

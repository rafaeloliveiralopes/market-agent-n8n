# PRD - Sistema de Atendimento Automatizado para Mercado via WhatsApp

## 1. Visão Geral

### 1.1 Propósito

Sistema de atendimento automatizado baseado em IA para mercados/supermercados, que permite aos clientes fazer pedidos, consultar produtos e preços, gerenciar cadastro e receber suporte através do WhatsApp de forma natural e conversacional.

### 1.2 Objetivo

Automatizar 100% do processo de atendimento ao cliente via WhatsApp, desde a consulta inicial de produtos até a finalização do pedido, reduzindo custos operacionais e aumentando a eficiência no atendimento.

### 1.3 Data de Referência

Dezembro de 2025

---

## 2. Stakeholders

- **Proprietário**: Rafael Lopes
- **Usuários Finais**: Clientes do mercado que fazem pedidos via WhatsApp
- **Equipe Operacional**: Atendentes e gestores do mercado
- **Fornecedores de Serviços**: OpenAI, Supabase, Evolution API

---

## 3. Arquitetura do Sistema

### 3.1 Tecnologias Principais

| Componente         | Tecnologia                    | Versão | Finalidade                            |
| ------------------ | ----------------------------- | ------ | ------------------------------------- |
| **Automação**      | n8n                           | -      | Orquestração de workflows             |
| **IA/LLM**         | OpenAI GPT-4o-mini            | Latest | Processamento de linguagem natural    |
| **Banco de Dados** | Supabase (PostgreSQL)         | -      | Armazenamento de dados e vector store |
| **Cache/Fila**     | Redis                         | -      | Gerenciamento de filas de mensagens   |
| **WhatsApp**       | Evolution API                 | -      | Integração com WhatsApp               |
| **Armazenamento**  | Google Drive                  | -      | Documentos de treinamento             |
| **Embeddings**     | OpenAI text-embedding-3-small | -      | Geração de embeddings para RAG        |

### 3.2 Arquitetura Multi-Agente

O sistema implementa uma arquitetura baseada em múltiplos agentes especializados:

```text
┌─────────────────────────────────────────────────────┐
│              WEBHOOK EVOLUTION API                  │
│              (Recebe mensagens WhatsApp)            │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│          AGENTE PRINCIPAL - ANA (Supervisor)        │
│          - Orquestra todos os agentes               │
│          - Mantém contexto da conversa              │
│          - Valida fluxo de coleta de dados          │
└────┬────────────┬─────────────┬─────────────────────┘
     │            │             │
     ▼            ▼             ▼
┌─────────┐ ┌──────────┐ ┌─────────────┐
│ Catálogo│ │   RAG    │ │ Treinamento │
│ e Preços│ │  Agent   │ │    RAG      │
└─────────┘ └──────────┘ └─────────────┘
```

---

## 4. Agentes do Sistema

### 4.1 Agente Principal - Ana (Supervisor)

**Nome**: Ana  
**Tipo**: Agente Orquestrador  
**Modelo**: OpenAI GPT-4o-mini  
**Localização**: `workflows/Agente principal - Atendente mercado.json`

#### Responsabilidades

1. **Orquestração**: Coordena todos os agentes especializados
2. **Gerenciamento de Contexto**: Mantém histórico e contexto da conversa
3. **Validação de Fluxo**: Garante sequência correta de coleta de informações
4. **Resposta ao Cliente**: Formula respostas naturais e amigáveis

#### Fluxo de Atendimento Obrigatório

1. **Produtos** - "Quais produtos você quer pedir?"
2. **Validação** - Consulta catálogo automaticamente via agente
3. **Marca** - "Você tem preferência por alguma marca?"
4. **Quantidade** - "Agora me diga a quantos do item XXX você deseja
5. **Variações** - Apenas se necessário (tamanho, sabor, voltagem, volume, peso (gramas, kilos))
6. **Nome** - Confirmação do nome do cliente
7. **Endereço** - Validação do endereço completo
8. **Telefone** - "Se eu precisar falar com você, posso ligar para este mesmo número do WhatsApp?"
9. **Substituição** - Preferência caso falte item
10. **Pagamento** - "Qual será a forma de pagamento?"
11. **Troco** - Se pagamento em dinheiro: "Vai precisar de troco?"

#### Regras de Ouro

- **UMA pergunta por mensagem** (nunca múltiplas perguntas)
- **NUNCA** aceitar produtos sem validação via catálogo
- **NUNCA** inventar preços, estoque ou prazos
- **SEMPRE** consultar agente apropriado para informações específicas
- **PROIBIDO** usar palavras relacionadas a "felicidade"
- **EXCEÇÃO**: Forma de pagamento é tratada diretamente (não há agente)

#### Ferramentas Disponíveis

- `consultar_catalogo_precos` - Validação obrigatória de produtos/marcas
- `thinking` - Para raciocínio complexo
- `buscar_documentos` - RAG para documentação

#### Memória

- **Tipo**: PostgreSQL Chat Memory
- **Janela de Contexto**: 20 mensagens
- **Session Key**: Número de telefone do cliente

---

### 4.2 Agente de Catálogo e Preços

**Nome**: Agente de Catálogo e Preços  
**Tipo**: Agente Especializado  
**Localização**: `workflows/Agente de Catálogo e Preços.json`

#### Responsabilidades do Agente de Catálogo e Preços

1. Buscar produtos no banco de dados Supabase
2. Validar disponibilidade de marcas específicas
3. Calcular estatísticas de preços (mínimo, máximo, médio)
4. Sugerir alternativas quando produto não encontrado
5. Retornar informações estruturadas sobre estoque

#### Parâmetros de Entrada

- `query` (string, obrigatório) - Nome do produto ou marca
- `categoria` (string, opcional) - Categoria do produto

#### Formato de Resposta

```json
{
  "encontrou": boolean,
  "total": number,
  "produtos": [
    {
      "id": number,
      "nome": string,
      "categoria": string,
      "tipo": string,
      "marca": string,
      "preco": number,
      "preco_formatado": string,
      "tamanho": string,
      "estoque_disponivel": number,
      "em_estoque": boolean
    }
  ],
  "estatisticas": {
    "preco_minimo": number,
    "preco_maximo": number,
    "preco_medio": number,
    "categorias_encontradas": string[]
  },
  "mensagem": string
}
```

#### Lógica de Busca

1. Validar query de entrada
2. Executar busca no Supabase (tabela `produtos_catalogo`)
3. Se encontrado: formatar resultados com estatísticas
4. Se não encontrado: buscar alternativas (produtos ativos)
5. Marcar alternativas com flag `is_alternativa: true`

---

### 4.3 Agente RAG (Retrieval-Augmented Generation)

**Nome**: AgenteRag  
**Tipo**: Agente de Conhecimento  
**Modelo**: OpenAI GPT-4o-mini

#### Responsabilidades do Agente RAG

1. Buscar informações em documentos de treinamento
2. Responder dúvidas baseadas em conhecimento armazenado
3. Não inventar respostas fora do conhecimento base

#### Ferramentas

- `buscar_documentos` - Vector store search com Supabase

#### Configuração Vector Store

- **Tabela**: `documents`
- **Função**: `match_documents`
- **Embedding**: OpenAI text-embedding-3-small (1536 dimensões)
- **TopK**: 2 documentos mais relevantes

#### Regras

- Usar **SOMENTE** dados da ferramenta `buscar_documentos`
- Se não houver dados: retornar "Não foi possível encontrar informações"
- **NUNCA** inventar respostas
- Copiar texto original sem alterações

---

### 4.4 Agente de Treinamento RAG

**Nome**: AgenteAperfeicoamentoDados  
**Tipo**: Agente de Manutenção  
**Modelo**: OpenAI GPT-4o-mini

#### Responsabilidades do gente de Treinamento RAG

1. Receber conteúdo textual e aprimorá-lo
2. Verificar duplicatas no banco de dados
3. Inserir/atualizar questões e respostas refinadas
4. Manter base de conhecimento atualizada

#### Ferramentas do gente de Treinamento RAG

- `Verificar_conteudo_existente` - Examina material semelhante
- `Modificar_base_dados` - Atualiza tabela de documentos

#### Fluxo de Treinamento

1. Schedule diário às 4h da manhã
2. Buscar arquivos no Google Drive (pasta específica)
3. Processar documentos (PDF, XLSX, Google Docs, CSV)
4. Dividir texto em chunks (1200 caracteres, overlap 200)
5. Gerar embeddings com OpenAI
6. Inserir no Supabase Vector Store

---

## 5. Estrutura de Dados

### 5.1 Tabelas PostgreSQL/Supabase

#### 5.1.1 `clientes_supermercado`

```sql
CREATE TABLE IF NOT EXISTS clientes_supermercado (
  id BIGSERIAL PRIMARY KEY,
  nome TEXT,
  rua TEXT,
  bairro TEXT,
  quadra TEXT,
  lote TEXT,
  numero_da_casa TEXT,
  whatsapp TEXT UNIQUE,
  ponto_de_referencia TEXT,
  data_da_atualizacao TIMESTAMPTZ DEFAULT NOW(),
  horas TIME,
  atendimento_ia TEXT DEFAULT 'reativada'
);
```

**Valores de `atendimento_ia`**:

- `reativada` - IA está ativa e respondendo
- `pause` - IA pausada, aguardando atendimento humano

#### 5.1.2 `chats`

```sql
CREATE TABLE chats (
  id BIGSERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ,
  phone TEXT,
  updated_at TEXT
);
```

#### 5.1.3 `chat_messages`

```sql
CREATE TABLE chat_messages (
  id BIGSERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ,
  phone TEXT,
  nomewpp TEXT,
  bot_message TEXT,
  user_message TEXT,
  message_type TEXT,
  active BOOLEAN DEFAULT TRUE
);
```

#### 5.1.4 `documents` (Vector Store)

```sql
CREATE TABLE documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,
  metadata JSONB,
  embedding VECTOR(1536)
);

CREATE FUNCTION match_documents (
  query_embedding VECTOR(1536),
  match_count INT DEFAULT NULL,
  filter JSONB DEFAULT '{}'
) RETURNS TABLE (
  id BIGINT,
  content TEXT,
  metadata JSONB,
  similarity FLOAT
) LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY
  SELECT
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

#### 5.1.5 `n8n_chat_histories`

Tabela gerenciada automaticamente pelo n8n para PostgreSQL Chat Memory.

#### 5.1.6 `produtos_catalogo`

Estrutura inferida:

- `id` - Identificador único
- `produto` / `nome` - Nome do produto
- `categoria` - Categoria do produto
- `tipo` - Tipo/classificação
- `marca` - Marca do produto
- `preco_decimal` - Preço em formato decimal
- `massa_volume` - Tamanho/volume
- `estoque_disponivel` - Quantidade em estoque
- `ativo` - Flag booleana de produto ativo

---

## 6. Fluxos de Trabalho

### 6.1 Fluxo Principal de Atendimento

```text
┌─────────────────────────┐
│ Mensagem WhatsApp       │
│ (Evolution API Webhook) │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Processamento Dados     │
│ - Extrai telefone       │
│ - Identifica tipo msg   │
│ - Extrai conteúdo       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Filtro Treinamento      │
│ - Normal: fluxo comum   │
│ - "289090:": treinar IA │
└─────┬─────────────┬─────┘
      │             │
      │             └──────► [Agente Treinamento RAG]
      │
      ▼
┌─────────────────────────┐
│ Buscar/Criar Cliente    │
│ (Supabase)              │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Verificar Status IA     │
│ - Ativa: continua       │
│ - Pausada: aguarda      │
│ - Reativar: volta ativa │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Roteamento Mensagem     │
│ - Texto: direto         │
│ - Áudio: transcrição    │
│ - Imagem: análise visão │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Enfileirar (Redis)      │
│ - Aguarda 1 seg         │
│ - Agrupa mensagens      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Compara Memória         │
│ - Evita duplicatas      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ RAG Agent               │
│ - Busca documentos      │
│ - Enriquece contexto    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ SUPERVISOR (Ana)        │
│ - Analisa contexto      │
│ - Chama agentes         │
│ - Formula resposta      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Histórico Supabase      │
│ - Busca/Cria chat       │
│ - Atualiza timestamp    │
│ - Salva mensagens       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Split de Mensagens      │
│ - Divide mensagens      │
│   longas (300-500 char) │
│ - Formata markdown      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Loop Envio (1 seg/msg)  │
│ - Evolution API         │
│ - Delay 4100ms          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Delete Memory (Redis)   │
│ - Limpa fila            │
└─────────────────────────┘
```

---

### 6.2 Fluxo de Consulta de Catálogo

```text
┌─────────────────────────┐
│ Ana recebe produto      │
│ mencionado pelo cliente │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Chama                   │
│ consultar_catalogo_     │
│ precos(query, categoria)│
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Workflow Agente         │
│ Catálogo e Preços       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Processar Entrada       │
│ - Limpar query          │
│ - Validar campos        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Query Válida?           │
└─────┬─────────────┬─────┘
      │ Não         │ Sim
      │             │
      ▼             ▼
┌──────────┐  ┌────────────────┐
│ Erro     │  │ Buscar Produtos│
│ Response │  │ (Supabase)     │
└──────────┘  └────────┬───────┘
                       │
                       ▼
              ┌────────────────┐
              │ Tem Resultados?│
              └─────┬─────┬────┘
                Sim │     │ Não
                    │     │
                    │     ▼
                    │  ┌──────────────────┐
                    │  │ Buscar           │
                    │  │ Alternativas     │
                    │  │ (produtos ativos)│
                    │  └────────┬─────────┘
                    │           │
                    │           ▼
                    │  ┌──────────────────┐
                    │  │ Marcar como      │
                    │  │ is_alternativa   │
                    │  └────────┬─────────┘
                    │           │
                    ▼           ▼
              ┌────────────────────┐
              │ Formatar Resultados│
              │ - Estatísticas     │
              │ - Preços           │
              │ - Mensagem final   │
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ Retornar JSON      │
              │ para Ana           │
              └────────────────────┘
```

---

### 6.3 Fluxo de Treinamento RAG

```text
┌─────────────────────────┐
│ Schedule Trigger        │
│ (Diariamente às 4h)     │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Google Drive            │
│ - Listar arquivos pasta │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Download Files          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Switch Tipo Arquivo     │
└─────┬────┬────┬────┬────┘
      │    │    │    │
      ▼    ▼    ▼    ▼
    PDF  Excel CSV  Docs
      │    │    │    │
      └────┴────┴────┘
             │
             ▼
┌─────────────────────────┐
│ Extract Text            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Aggregate               │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Summarize               │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Character Text Splitter │
│ - Chunk: 1200 chars     │
│ - Overlap: 200 chars    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Embeddings OpenAI       │
│ (text-embedding-3-small)│
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Insert Supabase         │
│ Vector Store            │
│ (tabela documents)      │
└─────────────────────────┘
```

---

## 7. Integrações Externas

### 7.1 Evolution API (WhatsApp)

**Endpoint Webhook**: `/fluxo-mercado`  
**Método**: POST

#### Estrutura de Mensagem Recebida

```json
{
  "body": {
    "event": "messages.upsert",
    "instance": "instance_name",
    "data": {
      "key": {
        "remoteJid": "5527XXXXXXXXX@s.whatsapp.net",
        "fromMe": false
      },
      "pushName": "Nome do Cliente",
      "message": {
        "conversation": "texto da mensagem",
        "extendedTextMessage": {
          "text": "texto estendido"
        },
        "audioMessage": {
          "url": "url_do_audio"
        },
        "imageMessage": {
          "url": "url_da_imagem",
          "caption": "legenda da imagem"
        }
      },
      "messageTimestamp": 1234567890,
      "messageType": "conversation|extendedTextMessage|audioMessage|imageMessage"
    }
  }
}
```

#### Envio de Mensagens

- **Resource**: `messages-api`
- **Delay**: 4100ms entre mensagens
- **Formato**: Markdown WhatsApp (_negrito_, ~tachado~, _itálico_)
- **Links**: Formatados como `www.link.com.br`

---

### 7.2 OpenAI

#### Modelos Utilizados

1. **GPT-4o-mini** - Chat/Agentes
2. **text-embedding-3-small** - Embeddings (1536 dim)
3. **Whisper** - Transcrição de áudio
4. **GPT-4o-mini Vision** - Análise de imagens

#### Configurações

- Temperatura padrão
- Sem configurações especiais de saída
- OutputParser para split de mensagens

---

### 7.3 Supabase

#### Serviços Utilizados

1. **Database (PostgreSQL)** - Tabelas principais
2. **Vector Store** - Busca semântica com pgvector
3. **API REST** - CRUD operations

#### Funções Customizadas

- `match_documents` - Busca vetorial com similaridade

---

### 7.4 Redis

#### Uso

- **Enfileiramento de mensagens** - Agrupa mensagens do mesmo usuário
- **Keys**: Número de telefone do cliente
- **TTL**: Automático (deletado após processamento)

#### Operações

- `PUSH` - Adicionar mensagem na fila
- `GET` - Recuperar todas as mensagens
- `DELETE` - Limpar fila após processar

---

### 7.5 Google Drive

#### Integração

- **Conta**: Google Drive configurada via OAuth2
- **Pasta de Treinamento**: Pasta específica configurada no workflow
- **Conversão**: Google Docs → Text, Sheets → Excel

#### Tipos de Arquivo Suportados

- PDF
- XLSX (Excel)
- Google Docs
- Google Sheets (convertido para CSV)

---

## 8. Funcionalidades Principais

### 8.1 Gestão de Conversação

#### 8.1.1 Histórico de Chat

- **Storage**: PostgreSQL (n8n_chat_histories)
- **Janela**: 20 últimas mensagens
- **Formato**: JSON com type (human/ai)
- **Session**: Baseada em número de telefone

#### 8.1.2 Memória Redis

- Agrupamento de mensagens em tempo real
- Evita processar mensagens duplicadas
- Aguarda 1 segundo antes de processar

#### 8.1.3 Tracking Supabase

- Tabela `chat_messages` - Histórico completo
- Tabela `chats` - Sessões ativas
- Atualização de timestamps

---

### 8.2 Processamento de Mídia

#### 8.2.1 Áudio

1. Recebe base64 do áudio
2. Converte para arquivo .ogg
3. Transcreve com OpenAI Whisper
4. Enfileira texto transcrito

#### 8.2.2 Imagem

1. Recebe base64 da imagem
2. Converte para arquivo .png
3. Analisa com GPT-4o Vision
4. Se tem legenda: combina com análise
5. Se sem legenda: apenas análise visual
6. Enfileira descrição

#### 8.2.3 Texto

- Processamento direto
- Suporte a texto estendido
- Enfileiramento imediato

---

### 8.3 Sistema de Pausar IA

#### Estados

1. **reativada** (padrão) - IA ativa
2. **pause** - Atendimento humano
3. Transição via mensagem "Atendimento finalizado"

#### Fluxo

```text
Cliente → IA Ativa → Menciona "Atendimento finalizado"
            ↓
       Status = pause
            ↓
    Aguarda 1 minuto
            ↓
    Cliente responde "Atendimento finalizado"
            ↓
       Status = reativada
            ↓
       IA volta ativa
```

#### Benefícios

- Permite intervenção humana quando necessário
- Cliente pode solicitar atendimento humano
- Retorno automático após 1 minuto

---

### 8.4 Split de Mensagens

**Objetivo**: Dividir respostas longas em mensagens menores para WhatsApp

#### Configuração

- **Tamanho**: 300-500 caracteres por mensagem
- **Modelo**: OpenAI GPT-4o-mini
- **Output**: JSON array de mensagens

#### Formatação

- `**negrito**` → `*negrito*`
- `~~tachado~~` → `~tachado~`
- `*itálico*` → `_itálico_`
- Links → `` `www.link.com` ``

#### Regras para Split de Mensagens

- Nunca cortar informações importantes
- Divisão natural (como conversa humana)
- Nunca separar mensagem vazia
- Preservar contexto entre splits

---

## 9. Regras de Negócio

### 9.1 Validação de Pedidos

#### 9.1.1 Valor Mínimo

- **Entrega**: R$ 120,00
- Informar ao cliente se não atingir
- Sugerir produtos adicionais

#### 9.1.2 Sequência Obrigatória

1. Produtos (validação via catálogo)
2. Marca (após validação)
3. Quantidade (após marca)
4. Nome do cliente
5. Endereço
6. Telefone
7. Preferência de substituição
8. Forma de pagamento
9. Troco (se dinheiro)

**NUNCA** pular etapas ou pedir múltiplas informações juntas.

---

### 9.2 Validação de Produtos

#### Regras Críticas

1. **SEMPRE** chamar `consultar_catalogo_precos` quando produto mencionado
2. **NUNCA** aceitar produto sem validação
3. Se `encontrou=false`: informar indisponibilidade + sugerir alternativas
4. Listar categorias e faixa de preço das alternativas
5. Validar marca usando retorno do catálogo

#### Resposta Padrão para Produto Não Encontrado

```text
"Infelizmente não temos [produto/marca].
Temos disponível: [produtos.nome do retorno].
Faixa de preço: R$ [preco_minimo] a R$ [preco_maximo]"
```

---

### 9.3 Forma de Pagamento

#### Opções

- Dinheiro (perguntar sobre troco)
- PIX
- Cartão de débito
- Cartão de crédito

#### Fluxo - Etapa Forma de Pagamento

1. Ana pergunta diretamente (não usa agente)
2. Se dinheiro: "Vai precisar de troco?"
3. Registrar resposta
4. Incluir no resumo final

---

### 9.4 Endereço

#### Campos Obrigatórios

- Rua
- Bairro
- Número da casa
- Ponto de referência

#### Campos Opcionais

- Quadra
- Lote
- Número

#### Validação

- Confirmar dados antes de finalizar
- Perguntar se pode usar mesmo número WhatsApp

---

## 10. Segurança e Privacidade

### 10.1 Controle de Acesso

- Endereços completos armazenados
- Números de telefone armazenados
- Histórico de conversas mantido
- **Recomendação**: Implementar LGPD compliance

### 10.2 Dados Sensíveis

- OpenAI API Key
- Supabase API Key
- Evolution API credentials
- Google Drive OAuth2
- PostgreSQL credentials
- Redis credentials

**Armazenamento**: n8n encrypted credentials

---

## 11. Monitoramento e Logs

### 11.1 Logs de Conversa

- **Tabela**: `chat_messages`
- **Campos**:
  - user_message
  - bot_message
  - message_type (incoming/outcoming)
  - created_at
  - nomewpp

### 11.2 Tracking de Sessões

- **Tabela**: `chats`
- **Campos**:
  - phone
  - created_at
  - updated_at

### 11.3 Métricas Importantes

- Total de atendimentos por dia
- Taxa de conclusão de pedidos
- Produtos mais consultados
- Tempo médio de atendimento
- Mensagens por conversa

---

## 12. Manutenção e Operação

### 12.1 Treinamento Diário

- **Horário**: 4h da manhã
- **Fonte**: Google Drive
- **Processo**: Automático
- **Resultado**: Vector store atualizado

### 12.2 Limpeza de Dados

#### Scripts Disponíveis

- `Deletar Histórico` - Limpa n8n_chat_histories
- `Deletar conteúdo Chat_Messages` - Limpa chat_messages
- `Deletar conteúdo Clientes Supermercado` - Limpa clientes
- `Deletar conteúdo Chats` - Limpa sessões
- `Deletar conteúdo Documentos` - Limpa vector store

**CUIDADO**: Operações irreversíveis

### 12.3 Manutenção do Catálogo

- Atualizar tabela `produtos_catalogo` manualmente
- Garantir campo `ativo=true` para produtos disponíveis
- Manter preços atualizados
- Atualizar estoque regularmente

---

## 13. Limitações Conhecidas

### 13.1 Técnicas

1. **Split de mensagens** - Pode falhar em casos extremos de formatação
2. **Transcrição de áudio** - Depende de qualidade do áudio
3. **Análise de imagem** - Pode não identificar produtos específicos
4. **Memória** - Limitada a 20 mensagens (pode perder contexto em conversas muito longas)

### 13.2 Funcionais

1. **Forma de pagamento** - Sem agente especializado (tratamento manual)
2. **Cálculo de entrega** - Não implementado (sem agente de logística)
3. **Validação de estoque em tempo real** - Depende de atualização manual
4. **Agendamento de entrega** - Não implementado

### 13.3 Operacionais

1. **Dependência de APIs externas** - OpenAI, Supabase, Evolution
2. **Custo por mensagem** - OpenAI cobra por token
3. **Rate limits** - APIs têm limites de requisições

---

## 14. Roadmap Futuro

### 14.1 Fase 1 - Melhorias Imediatas

- [ ] Implementar agente de estoque dedicado
- [ ] Criar agente de logística para cálculo de prazo de entrega
- [ ] Implementar sistema de notificações (status do pedido)
- [ ] Dashboard de métricas em tempo real

### 14.2 Fase 2 - Expansão

- [ ] Suporte a múltiplos mercados
- [ ] Sistema de promoções automáticas
- [ ] Programa de fidelidade integrado
- [ ] Chatbot para outras plataformas (Telegram, Instagram)

### 14.3 Fase 3 - Inteligência Avançada

- [ ] Recomendação de produtos baseada em histórico
- [ ] Previsão de demanda
- [ ] Detecção de fraudes
- [ ] Análise de sentimento do cliente

---

## 15. Guia de Deploy

### 15.1 Pré-requisitos

1. Conta n8n (self-hosted ou cloud)
2. Conta OpenAI com créditos
3. Conta Supabase
4. Instância Evolution API configurada
5. Servidor Redis
6. PostgreSQL com extensão pgvector
7. Conta Google Drive (OAuth2)

### 15.2 Configuração Inicial

#### Passo 1: Banco de Dados

```sql
-- Habilitar pgvector
CREATE EXTENSION vector;

-- Criar tabelas (executar scripts da seção 5.1)
-- Criar função match_documents
```

#### Passo 2: n8n

1. Importar workflows:
   - `Agente principal - Atendente mercado.json`
   - `Agente de Catálogo e Preços.json`
2. Configurar credenciais:
   - OpenAI API
   - Supabase
   - Evolution API
   - Google Drive
   - PostgreSQL
   - Redis

#### Passo 3: Evolution API

1. Criar instância
2. Conectar WhatsApp
3. Configurar webhook apontando para n8n
4. Testar conexão

#### Passo 4: Treinamento Inicial

1. Criar pasta no Google Drive
2. Adicionar documentos de treinamento
3. Executar workflow de treinamento manualmente
4. Validar inserção no vector store

#### Passo 5: Popular Catálogo

1. Importar produtos para tabela `produtos_catalogo`
2. Garantir campos obrigatórios preenchidos
3. Validar com agente de catálogo

### 15.3 Validação

1. Enviar mensagem de teste via WhatsApp
2. Verificar logs no n8n
3. Confirmar resposta da IA
4. Testar fluxo completo de pedido
5. Validar histórico no Supabase

---

## 16. Troubleshooting

### 16.1 IA não responde

**Possíveis causas**:

- Status `atendimento_ia = pause`
- Webhook não configurado
- Credenciais OpenAI inválidas

**Solução**:

1. Verificar tabela `clientes_supermercado`
2. Atualizar status para `reativada`
3. Testar webhook Evolution API
4. Validar API key OpenAI

---

### 16.2 Produtos não encontrados

**Possíveis causas**:

- Catálogo vazio
- Campo `ativo = false`
- Query de busca incorreta

**Solução**:

1. Verificar tabela `produtos_catalogo`
2. Confirmar campo `ativo = true`
3. Testar agente de catálogo isoladamente
4. Validar query SQL do agente

---

### 16.3 Memória/Contexto perdido

**Possíveis causas**:

- Janela de contexto excedida (>20 msgs)
- Conexão PostgreSQL falhou
- Chave de sessão incorreta

**Solução**:

1. Verificar conexão com PostgreSQL
2. Confirmar tabela `n8n_chat_histories`
3. Validar session_id = telefone
4. Aumentar contextWindowLength se necessário

---

### 16.4 Mensagens não divididas corretamente

**Possíveis causas**:

- Output parser falhou
- Formato JSON inválido
- Mensagem muito grande

**Solução**:

1. Verificar logs do node "Split de mensagens"
2. Validar prompt do LLM
3. Testar manualmente com exemplo
4. Ajustar tamanho de chunk

---

## 17. Glossário

| Termo             | Definição                                                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------------------------- |
| **RAG**           | Retrieval-Augmented Generation - Técnica de IA que busca informações em documentos antes de gerar resposta |
| **Vector Store**  | Banco de dados otimizado para busca vetorial/semântica                                                     |
| **Embedding**     | Representação numérica de texto usada para busca semântica                                                 |
| **Supervisor**    | Agente principal que orquestra outros agentes                                                              |
| **Tool**          | Ferramenta disponível para o agente usar                                                                   |
| **Chunk**         | Pedaço de texto dividido para processamento                                                                |
| **Session**       | Sessão de conversa identificada por telefone                                                               |
| **Evolution API** | Plataforma de integração com WhatsApp                                                                      |
| **n8n**           | Plataforma de automação de workflows                                                                       |
| **Supabase**      | Backend-as-a-Service baseado em PostgreSQL                                                                 |

---

## 18. Referências

### 18.1 Documentação Externa

- [n8n Documentation](https://docs.n8n.io/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Supabase Docs](https://supabase.com/docs)
- [Evolution API](https://doc.evolution-api.com/)
- [pgvector](https://github.com/pgvector/pgvector)

### 18.2 Arquivos do Projeto

- [workflows/Agente principal - Atendente mercado.json](../workflows/Agente%20principal%20-%20Atendente%20mercado.json)
- [workflows/Agente de Catálogo e Preços.json](../workflows/Agente%20de%20Catálogo%20e%20Preços.json)
- [LICENSE](../LICENSE)

---

## 19. Controle de Versão

| Versão | Data     | Autor        | Mudanças              |
| ------ | -------- | ------------ | --------------------- |
| 1.0    | Dez/2025 | Rafael Lopes | Versão inicial do PRD |

---

## 20. Aprovações

| Stakeholder  | Cargo        | Aprovação | Data     |
| ------------ | ------------ | --------- | -------- |
| Rafael Lopes | Proprietário | ✓         | Dez/2025 |

---

**Documento Confidencial**  
Copyright © 2025 Rafael Lopes. Todos os direitos reservados.

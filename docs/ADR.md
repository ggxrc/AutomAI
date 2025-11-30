# Architecture Decision Record (ADR)

## AutomAI - Architectural Decisions

---

## ADR-001: Extension Architecture Pattern

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

A extensão AutomAI precisa de uma arquitetura que suporte:
- Execução de um loop contínuo e autônomo
- Múltiplos serviços trabalhando em conjunto
- Estado persistente e recuperável
- Integração com VS Code e Copilot

### Decision

Adotar uma **arquitetura em camadas** com os seguintes componentes:

1. **Command Layer**: Handlers de comandos do VS Code
2. **Controller Layer**: Gerenciadores de fluxo e estado
3. **Service Layer**: Serviços de negócio (Implementer, Debugger, Reviewer, Critic, Documenter)
4. **Integration Layer**: Integrações com APIs externas

### Consequences

**Positivas:**
- Clara separação de responsabilidades
- Facilita testes unitários
- Permite evolução independente de componentes
- Facilita manutenção

**Negativas:**
- Mais código boilerplate inicial
- Curva de aprendizado para novos contribuidores

---

## ADR-002: Loop Execution Model

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

O loop autônomo precisa executar continuamente sem bloquear a UI do VS Code, permitindo pausas e interrupções.

### Options Considered

1. **Synchronous Loop**: Execução sequencial simples
2. **Worker Thread**: Execução em thread separada
3. **Async/Await Loop**: Execução assíncrona no main thread

### Decision

Utilizar **Async/Await Loop** com controle de estado.

```typescript
async function executeLoop(): Promise<void> {
  while (isRunning) {
    await executePhase();
    await delay(cooldownMs);
  }
}
```

### Rationale

- VS Code extensions rodam no main thread
- Worker threads têm limitações de API
- Async/await permite yield para a event loop
- Controle de estado simples e eficaz

### Consequences

**Positivas:**
- Não bloqueia a UI
- Acesso completo às APIs do VS Code
- Implementação simples
- Fácil debug

**Negativas:**
- Operações longas podem causar delays
- Requer cuidado com operações síncronas

---

## ADR-003: Task Management Strategy

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

O sistema precisa de uma fonte de tarefas para o Copilot executar. Diferentes projetos podem ter diferentes formas de organizar tarefas.

### Options Considered

1. **Markdown File**: Arquivo `.md` com lista de tarefas
2. **JSON File**: Arquivo estruturado com tarefas
3. **GitHub Issues**: Integração direta com Issues
4. **Custom API**: Endpoint externo

### Decision

Iniciar com **Markdown File** como fonte primária, com extensibilidade para outras fontes.

### Task File Format

```markdown
# Tasks

## [ ] Implementar feature X
Descrição detalhada da feature X...

## [ ] Corrigir bug Y
Descrição do bug Y...

## [x] Feature Z (completed)
Descrição da feature Z...
```

### Rationale

- Markdown é familiar para desenvolvedores
- Fácil de editar manualmente
- Integra com documentação existente
- Pode evoluir para outras fontes

### Consequences

**Positivas:**
- Baixa barreira de entrada
- Não requer infraestrutura adicional
- Fácil de versionar com o código

**Negativas:**
- Menos estruturado que JSON
- Parsing pode ser complexo
- Não tem metadados ricos

---

## ADR-004: Critique System Design

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

O sistema de crítica precisa avaliar se uma implementação é aceitável ou se precisa de mais debug/refatoração.

### Decision

Implementar um **sistema de scoring multi-critério** com threshold configurável.

### Criteria

| Critério | Peso | Descrição |
|----------|------|-----------|
| Tests Passing | 30% | Todos os testes passam |
| Lint Clean | 20% | Sem erros de linting |
| Build Success | 20% | Build compila sem erros |
| Code Quality | 15% | Métricas de qualidade |
| Documentation | 15% | Código documentado |

### Scoring Algorithm

```typescript
function calculateScore(results: CritiqueResults): number {
  let score = 0;
  
  if (results.testsPass) score += 30;
  if (results.lintClean) score += 20;
  if (results.buildSuccess) score += 20;
  score += results.codeQuality * 0.15;
  score += results.documentation * 0.15;
  
  return score;
}

function shouldContinue(score: number, threshold: number = 70): boolean {
  return score >= threshold;
}
```

### Consequences

**Positivas:**
- Avaliação objetiva
- Threshold configurável
- Critérios claros
- Extensível

**Negativas:**
- Pode não capturar nuances
- Requer calibração inicial

---

## ADR-005: State Persistence Strategy

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

O estado do loop precisa ser persistido para recuperação em caso de falhas e para permitir pausas/retomadas.

### Options Considered

1. **In-Memory Only**: Sem persistência
2. **VS Code Workspace State**: API nativa do VS Code
3. **File-based**: Arquivo JSON no workspace
4. **SQLite**: Banco de dados local

### Decision

Utilizar **VS Code Workspace State** como armazenamento primário.

```typescript
// Save state
context.workspaceState.update('automAI.state', state);

// Load state
const state = context.workspaceState.get<LoopState>('automAI.state');
```

### Rationale

- API nativa e suportada
- Automático e transparente
- Escopo por workspace
- Sincronizado com Settings Sync

### Consequences

**Positivas:**
- Integração nativa com VS Code
- Sem dependências adicionais
- Backup automático

**Negativas:**
- Limitado em tamanho
- Menos flexível que arquivo

---

## ADR-006: Error Handling Strategy

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

Erros podem ocorrer em qualquer fase do loop. O sistema precisa ser resiliente e informativo.

### Decision

Implementar **estratégia de retry com fallback**.

### Error Categories

| Categoria | Ação | Exemplo |
|-----------|------|---------|
| Transient | Retry 3x | Network timeout |
| Recoverable | Log + Continue | Lint warning |
| Critical | Stop + Notify | API unavailable |
| Fatal | Stop + Alert | Memory exhausted |

### Implementation

```typescript
async function executeWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (isFatalError(error)) throw error;
      if (i === maxRetries - 1) throw error;
      await delay(Math.pow(2, i) * 1000); // Exponential backoff
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Consequences

**Positivas:**
- Sistema mais robusto
- Menos interrupções
- Logs informativos

**Negativas:**
- Pode mascarar problemas
- Delays em erros transientes

---

## ADR-007: Documentation Generation Format

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

Cada implementação deve ser documentada automaticamente de forma detalhada.

### Decision

Gerar documentação em **Markdown** com estrutura padronizada.

### Documentation Template

```markdown
# Implementation: [Task Title]

**Date:** YYYY-MM-DD  
**Task ID:** TASK-XXX  
**Duration:** XX minutes  

## Summary

Brief summary of what was implemented.

## Changes

### Files Modified
- `path/to/file1.ts` - Description of changes
- `path/to/file2.ts` - Description of changes

### Statistics
- Lines Added: XX
- Lines Removed: XX
- Files Changed: XX

## Implementation Details

Detailed description of the implementation approach.

## Testing

- [ ] Unit tests added
- [ ] Integration tests added
- [ ] Manual testing performed

## Notes

Any additional notes or considerations.
```

### Consequences

**Positivas:**
- Formato legível e padrão
- Fácil de versionar
- Integra com outras ferramentas

**Negativas:**
- Pode ficar verboso
- Requer manutenção do template

---

## ADR-008: Logging and Observability

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

Para debug e monitoramento, o sistema precisa de logging adequado.

### Decision

Utilizar o **Output Channel do VS Code** como destino principal de logs.

```typescript
const outputChannel = vscode.window.createOutputChannel('AutomAI');

function log(level: LogLevel, message: string, data?: any): void {
  const timestamp = new Date().toISOString();
  const formatted = `[${timestamp}] [${level}] ${message}`;
  
  outputChannel.appendLine(formatted);
  if (data) {
    outputChannel.appendLine(JSON.stringify(data, null, 2));
  }
}
```

### Log Levels

| Level | Usage |
|-------|-------|
| DEBUG | Informações detalhadas para desenvolvimento |
| INFO | Eventos normais do sistema |
| WARN | Situações inesperadas mas não críticas |
| ERROR | Erros que afetam funcionalidade |

### Consequences

**Positivas:**
- Integração nativa com VS Code
- Visível no painel de Output
- Sem dependências externas

**Negativas:**
- Sem persistência por padrão
- Limitado em análise avançada

---

## ADR-009: Copilot Integration Approach

**Date:** 2025-11-30  
**Status:** Proposed  
**Deciders:** AutomAI Team  

### Context

A extensão precisa se comunicar com o Copilot para gerar implementações. A API do Copilot não é totalmente pública.

### Options Considered

1. **Chat API**: Usar a API de chat do Copilot
2. **Inline Suggestions**: Usar sugestões inline
3. **Command Execution**: Executar comandos do Copilot
4. **Language Model API**: Usar a API de Language Model do VS Code

### Decision

Utilizar a **Language Model API do VS Code** (quando disponível) com fallback para **Chat API**.

```typescript
// Using Language Model API
const models = await vscode.lm.selectChatModels({
  vendor: 'copilot',
  family: 'gpt-4'
});

if (models.length > 0) {
  const response = await models[0].sendRequest(messages, options, token);
  // Process response
}
```

### Rationale

- Language Model API é a forma oficial
- Chat API está amplamente disponível
- Permite flexibilidade na implementação

### Consequences

**Positivas:**
- Uso de APIs oficiais
- Compatibilidade futura
- Flexibilidade

**Negativas:**
- APIs ainda em evolução
- Pode haver limitações não documentadas

---

## ADR-010: Configuration Management

**Date:** 2025-11-30  
**Status:** Accepted  
**Deciders:** AutomAI Team  

### Context

A extensão precisa de configurações personalizáveis pelo usuário.

### Decision

Utilizar o **sistema de configurações do VS Code** através do `contributes.configuration`.

```json
{
  "contributes": {
    "configuration": {
      "title": "AutomAI",
      "properties": {
        "automai.taskSource": {
          "type": "string",
          "default": "file",
          "enum": ["file", "issues", "custom"],
          "description": "Source of tasks for the autonomous loop"
        },
        "automai.critiqueMinScore": {
          "type": "number",
          "default": 70,
          "minimum": 0,
          "maximum": 100,
          "description": "Minimum score for a critique to pass"
        }
      }
    }
  }
}
```

### Consequences

**Positivas:**
- Interface familiar para usuários
- Sincroniza com Settings Sync
- Validação automática

**Negativas:**
- Limitado a tipos básicos
- Mudanças requerem reload

---

## Summary of Decisions

| ADR | Decision | Status |
|-----|----------|--------|
| ADR-001 | Layered Architecture | Accepted |
| ADR-002 | Async/Await Loop | Accepted |
| ADR-003 | Markdown Task Files | Accepted |
| ADR-004 | Multi-criteria Scoring | Accepted |
| ADR-005 | VS Code Workspace State | Accepted |
| ADR-006 | Retry with Fallback | Accepted |
| ADR-007 | Markdown Documentation | Accepted |
| ADR-008 | VS Code Output Channel | Accepted |
| ADR-009 | Language Model API | Proposed |
| ADR-010 | VS Code Configuration | Accepted |

---

## Revision History

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2025-11-30 | 1.0 | AutomAI Team | Initial version with 10 ADRs |

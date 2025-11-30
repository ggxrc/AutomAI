# Request for Comments (RFC)

## AutomAI - Technical Proposal for Copilot Autonomous Agent Extension

**RFC Number:** RFC-001  
**Title:** AutomAI Autonomous Development Loop  
**Author:** AutomAI Team  
**Status:** Draft  
**Created:** 2025-11-30  
**Last Updated:** 2025-11-30  

---

## 1. Abstract

Este RFC propõe a arquitetura técnica e os detalhes de implementação para a extensão AutomAI do VS Code. A extensão automatiza o GitHub Copilot para trabalhar de forma autônoma em um ciclo contínuo de desenvolvimento que inclui implementação, debug, revisão, crítica, documentação e progressão de tarefas.

---

## 2. Motivation

### 2.1 Background

O GitHub Copilot revolucionou o desenvolvimento de software ao fornecer sugestões de código inteligentes. No entanto, sua utilização ainda requer interação constante do desenvolvedor para cada tarefa. 

### 2.2 Problem

Atualmente, desenvolvedores precisam:
1. Iniciar manualmente cada interação com o Copilot
2. Validar e testar manualmente cada implementação
3. Documentar manualmente o trabalho realizado
4. Gerenciar a progressão de tarefas manualmente

### 2.3 Proposal

Criar uma extensão que orquestre automaticamente o Copilot através de um loop estruturado, maximizando a utilização dos requests disponíveis e garantindo qualidade através de validação automática.

---

## 3. Detailed Design

### 3.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                           AutomAI Extension                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Command   │  │    Loop     │  │    Task     │  │   Request   │ │
│  │   Handler   │  │  Controller │  │   Manager   │  │   Monitor   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │                │         │
│         └────────────────┼────────────────┼────────────────┘         │
│                          │                │                          │
│                          ▼                ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                      Core Services Layer                         ││
│  ├─────────────┬─────────────┬─────────────┬───────────────────────┤│
│  │ Implementer │   Debugger  │   Reviewer  │      Critic           ││
│  ├─────────────┼─────────────┼─────────────┼───────────────────────┤│
│  │ Documenter  │   Logger    │   State     │    Config             ││
│  └─────────────┴─────────────┴─────────────┴───────────────────────┘│
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │                    Integration Layer                             ││
│  ├─────────────────────────────┬───────────────────────────────────┤│
│  │     VS Code API             │       Copilot Integration          ││
│  └─────────────────────────────┴───────────────────────────────────┘│
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Core Components

#### 3.2.1 Command Handler
Responsável por registrar e processar comandos do VS Code.

```typescript
interface CommandHandler {
  registerCommands(): void;
  startLoop(): Promise<void>;
  stopLoop(): Promise<void>;
  getStatus(): LoopStatus;
}
```

#### 3.2.2 Loop Controller
Gerencia o ciclo principal de execução.

```typescript
interface LoopController {
  start(): Promise<void>;
  stop(): Promise<void>;
  pause(): Promise<void>;
  resume(): Promise<void>;
  getCurrentPhase(): LoopPhase;
  onPhaseChange(callback: (phase: LoopPhase) => void): void;
}

enum LoopPhase {
  IDLE = 'idle',
  IMPLEMENTING = 'implementing',
  DEBUGGING = 'debugging',
  REVIEWING = 'reviewing',
  CRITIQUING = 'critiquing',
  DOCUMENTING = 'documenting',
  TRANSITIONING = 'transitioning'
}
```

#### 3.2.3 Task Manager
Gerencia as tarefas a serem executadas.

```typescript
interface TaskManager {
  loadTasks(): Promise<Task[]>;
  getCurrentTask(): Task | null;
  completeTask(taskId: string): Promise<void>;
  getNextTask(): Task | null;
  hasMoreTasks(): boolean;
}

interface Task {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  priority: number;
  createdAt: Date;
  completedAt?: Date;
}
```

#### 3.2.4 Request Monitor
Monitora a disponibilidade de requests do Copilot.

```typescript
interface RequestMonitor {
  getAvailableRequests(): number;
  hasAvailableRequests(): boolean;
  onRequestsExhausted(callback: () => void): void;
  trackRequest(): void;
}
```

### 3.3 Core Services

#### 3.3.1 Implementer Service
Responsável por executar a implementação da tarefa.

```typescript
interface ImplementerService {
  implement(task: Task): Promise<ImplementationResult>;
  getImplementedFiles(): string[];
  getChanges(): CodeChange[];
}

interface ImplementationResult {
  success: boolean;
  filesModified: string[];
  linesAdded: number;
  linesRemoved: number;
  errors?: string[];
}
```

#### 3.3.2 Debugger Service
Responsável por executar e debugar o código.

```typescript
interface DebuggerService {
  runTests(): Promise<TestResult>;
  runLinter(): Promise<LintResult>;
  runBuild(): Promise<BuildResult>;
  fixErrors(errors: Error[]): Promise<FixResult>;
}

interface TestResult {
  passed: number;
  failed: number;
  errors: TestError[];
}
```

#### 3.3.3 Reviewer Service
Analisa o código implementado.

```typescript
interface ReviewerService {
  reviewChanges(): Promise<ReviewResult>;
  getChangeSummary(): ChangeSummary;
}

interface ReviewResult {
  filesReviewed: number;
  issues: ReviewIssue[];
  suggestions: Suggestion[];
}
```

#### 3.3.4 Critic Service
Avalia a qualidade da implementação.

```typescript
interface CriticService {
  evaluate(implementation: ImplementationResult, review: ReviewResult): Promise<CriticResult>;
  shouldContinue(): boolean;
  getRecommendation(): Recommendation;
}

interface CriticResult {
  approved: boolean;
  score: number; // 0-100
  reasons: string[];
  recommendation: Recommendation;
}

enum Recommendation {
  CONTINUE = 'continue',
  REFACTOR = 'refactor',
  REVERT = 'revert',
  MANUAL_REVIEW = 'manual_review'
}
```

#### 3.3.5 Documenter Service
Gera documentação automática.

```typescript
interface DocumenterService {
  documentImplementation(task: Task, changes: CodeChange[]): Promise<Documentation>;
  updateChangelog(entry: ChangelogEntry): Promise<void>;
  generateReport(): Promise<Report>;
}

interface Documentation {
  summary: string;
  details: string;
  filesAffected: string[];
  timestamp: Date;
}
```

### 3.4 Loop Execution Flow

```typescript
class AutonomousLoop {
  private isRunning: boolean = false;
  
  async execute(): Promise<void> {
    this.isRunning = true;
    
    while (this.isRunning) {
      // Check preconditions
      if (!this.requestMonitor.hasAvailableRequests()) {
        this.logger.info('No requests available, stopping loop');
        break;
      }
      
      const task = this.taskManager.getNextTask();
      if (!task) {
        this.logger.info('No more tasks, stopping loop');
        break;
      }
      
      try {
        // Phase 1: Implement
        this.setPhase(LoopPhase.IMPLEMENTING);
        const implementation = await this.implementer.implement(task);
        
        // Phase 2: Debug (with retry loop)
        let debugResult: DebugResult;
        let critiqueApproved = false;
        
        do {
          this.setPhase(LoopPhase.DEBUGGING);
          debugResult = await this.debugger.debug(implementation);
          
          // Phase 3: Review
          this.setPhase(LoopPhase.REVIEWING);
          const review = await this.reviewer.reviewChanges();
          
          // Phase 4: Critique
          this.setPhase(LoopPhase.CRITIQUING);
          const critique = await this.critic.evaluate(implementation, review);
          
          if (critique.approved) {
            critiqueApproved = true;
          } else {
            // Go back to debugging
            this.logger.info('Critique not approved, returning to debug phase');
          }
        } while (!critiqueApproved && this.isRunning);
        
        // Phase 5: Document
        this.setPhase(LoopPhase.DOCUMENTING);
        await this.documenter.documentImplementation(task, implementation.changes);
        
        // Phase 6: Transition to next task
        this.setPhase(LoopPhase.TRANSITIONING);
        await this.taskManager.completeTask(task.id);
        
      } catch (error) {
        this.logger.error('Error in loop execution', error);
        this.handleError(error);
      }
    }
    
    this.setPhase(LoopPhase.IDLE);
  }
}
```

### 3.5 State Management

```typescript
interface LoopState {
  status: LoopStatus;
  currentPhase: LoopPhase;
  currentTask: Task | null;
  tasksCompleted: number;
  tasksFailed: number;
  requestsUsed: number;
  startedAt: Date | null;
  lastActivityAt: Date | null;
  errors: Error[];
}

enum LoopStatus {
  IDLE = 'idle',
  RUNNING = 'running',
  PAUSED = 'paused',
  STOPPED = 'stopped',
  ERROR = 'error'
}
```

### 3.6 Configuration

```typescript
interface AutomAIConfig {
  // Task source configuration
  taskSource: 'file' | 'issues' | 'custom';
  taskFilePath?: string;
  
  // Critique thresholds
  critiqueMinScore: number; // 0-100, default 70
  maxDebugIterations: number; // default 3
  
  // Request management
  maxRequestsPerSession: number;
  requestCooldownMs: number;
  
  // Logging
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  logToFile: boolean;
  
  // Documentation
  documentationPath: string;
  autoUpdateChangelog: boolean;
}
```

---

## 4. Implementation Plan

### 4.1 Phase 1: Foundation (Week 1-2)
- Set up project structure
- Implement Command Handler
- Implement basic Loop Controller
- Create State Management system

### 4.2 Phase 2: Core Services (Week 3-5)
- Implement Implementer Service
- Implement Debugger Service
- Implement Reviewer Service
- Integrate with Copilot

### 4.3 Phase 3: Quality Assurance (Week 6-7)
- Implement Critic Service
- Define critique criteria
- Implement retry logic

### 4.4 Phase 4: Documentation (Week 8-9)
- Implement Documenter Service
- Create documentation templates
- Implement changelog updates

### 4.5 Phase 5: Polish (Week 10-11)
- Testing and bug fixes
- Performance optimization
- Documentation

---

## 5. File Structure

```
automai/
├── src/
│   ├── extension.ts              # Entry point
│   ├── commands/
│   │   ├── index.ts
│   │   ├── startLoop.ts
│   │   └── stopLoop.ts
│   ├── controllers/
│   │   ├── loopController.ts
│   │   └── stateController.ts
│   ├── services/
│   │   ├── implementer.ts
│   │   ├── debugger.ts
│   │   ├── reviewer.ts
│   │   ├── critic.ts
│   │   ├── documenter.ts
│   │   └── logger.ts
│   ├── managers/
│   │   ├── taskManager.ts
│   │   └── requestMonitor.ts
│   ├── integrations/
│   │   └── copilot.ts
│   ├── models/
│   │   ├── task.ts
│   │   ├── state.ts
│   │   └── config.ts
│   └── utils/
│       ├── constants.ts
│       └── helpers.ts
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   └── ADR.md
├── test/
│   └── ...
└── package.json
```

---

## 6. API Design

### 6.1 VS Code Commands

| Command | Description |
|---------|-------------|
| `automai.startLoop` | Inicia o loop autônomo |
| `automai.stopLoop` | Para o loop autônomo |
| `automai.pauseLoop` | Pausa o loop autônomo |
| `automai.resumeLoop` | Retoma o loop pausado |
| `automai.getStatus` | Retorna o status atual do loop |

### 6.2 VS Code Settings

```json
{
  "automai.taskSource": "file",
  "automai.taskFilePath": "./tasks.md",
  "automai.critiqueMinScore": 70,
  "automai.maxDebugIterations": 3,
  "automai.logLevel": "info",
  "automai.documentationPath": "./docs/implementations"
}
```

---

## 7. Testing Strategy

### 7.1 Unit Tests
- Test each service independently
- Mock Copilot integration
- Test state transitions

### 7.2 Integration Tests
- Test loop execution flow
- Test error handling
- Test configuration changes

### 7.3 E2E Tests
- Test complete task execution
- Test multiple task execution
- Test stop/pause/resume

---

## 8. Security Considerations

### 8.1 Code Execution
- Sandbox code execution where possible
- Limit file system access
- Validate all inputs

### 8.2 Data Privacy
- No telemetry without user consent
- Local-only storage of state
- Secure handling of task data

---

## 9. Alternatives Considered

### 9.1 External Orchestration
**Rejected**: Would require external dependencies and complicate installation.

### 9.2 Simple Script Execution
**Rejected**: Would not provide the level of integration and feedback needed.

### 9.3 Web-based Dashboard
**Rejected**: Adds unnecessary complexity for initial version.

---

## 10. Open Questions

1. How to best integrate with Copilot's internal API?
2. What are the best criteria for the Critic service?
3. Should we support custom task sources beyond file-based?
4. How to handle long-running tasks?

---

## 11. References

- [VS Code Extension API](https://code.visualstudio.com/api)
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)

---

## 12. Changelog

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2025-11-30 | AutomAI Team | Initial draft |

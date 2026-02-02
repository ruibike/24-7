Este repositório implementa uma arquitectura de orquestração de agentes determinística. O objectivo não é ter LLMs a conversar entre si, mas sim workers especializados a passar trabalho estruturado sob a supervisão estrita de um Coordenador em código.🧠 Princípios FundamentaisO Coordinator NÃO é um LLM: É código (Python) determinístico, barato e auditável.Workers são Stateless: Não há memória entre chamadas. Entrada JSON → Processamento → Saída JSON.Comunicação Estrita: Nada de texto livre. Tudo é validado via Schema.Hierarquia de Modelos: O modelo certo para a tarefa certa (7B vs 70B vs 8x22B).🏗 ArquitecturaO Coordinator (The Shift Manager)O coração do sistema. Não alucina, não se cansa e não custa tokens.Role: Gestor de fila, Validador de Schema, Roteador.Tech: Python (não AI).Responsabilidades:Recebe trabalho da Queue.Decide (via regras if/else) qual Worker chamar.Valida a estrutura do JSON de saída.Gere retry, backoff e escalonamento de modelos.Persiste dados no Storage (Postgres/Vector DB).Os Workers (The Specialists)TierModelo ExemploCusto/SpeedFunção PrincipalWorker 7BLlama-3-8B / Mistral⚡ Fast / $ LowTriage, Extração Simples, Deduplicação.Worker 70BLlama-3-70B⚖️ MediumClassificação, Análise Semântica, Design de PoC.Worker 8x22BMixtral 8x22B / GPT-4🧠 Heavy / $$ HighRelatórios Finais, Casos Complexos, Escalonamento.🔄 Workflow de ComunicaçãoO fluxo é unidirecional e controlado pelo Coordinator.Fragmento do códigograph TD
    Q[Queue] -->|Item| C{COORDINATOR}
    
    subgraph "Decision Logic (Python)"
    C -->|Router Rules| W_Select
    C -->|Schema Validation| Valid
    end
    
    W_Select -->|Simple Task| W1[Worker 7B]
    W_Select -->|Medium Task| W2[Worker 70B]
    W_Select -->|Complex Task| W3[Worker 8x22B]
    
    W1 -->|JSON| C
    W2 -->|JSON| C
    W3 -->|JSON| C
    
    Valid -->|Invalid/Low Conf| Retry[Retry / Escalate]
    Valid -->|Success| DB[(Storage)]
    
    DB -->|Next Step| Q
📋 Task Pipeline & RoutingExemplo real de processamento de vulnerabilidades (CVE):1. TRIAGE (Worker 7B)Input: Raw CVE, advisory.Objetivo: Filtrar ruído. É relevante?Output: { relevance: 0-100, category_guess, needs_deeper: bool }2. CLASSIFY (Worker 70B)Input: Item triado.Objetivo: Categorização semântica precisa.Output: { category, platform, impact, confidence }Regra: Se confidence < 70% → ESCALATE.3. ANALYZE (Worker 70B)Input: Item classificado + contexto.Objetivo: Avaliar risco real e caminhos de exploit.Output: { risk_score, exploit_path, poc_viable: bool }4. POC DESIGN (Worker 70B ou 8x22B)Input: Análise profunda.Objetivo: Criar passos de reprodução.Output: { poc_code, test_steps, expected_result }5. LAB TEST (System - No LLM)Input: Script do PoC.Ação: Spin up microVM → Executa → Captura Output → Destroy.Output: { success: bool, artifacts: [] }6. REPORT (Worker 8x22B)Input: Todo o contexto acumulado + Evidências do Lab.Objetivo: Relatório final para humanos (formato HackerOne/Jira).Output: Markdown formatado.📈 Matriz de Escalonamento (Escalation Rules)O Coordinator decide quando "chamar o supervisor" (modelo maior).CondiçãoAção do CoordinatorWorker 7B conf < 50%Retry com Worker 70BSchema Inválido (2x)Escala para modelo superiorWorker 70B conf < 70%Escala para Worker 8x22BFalha na ClassificaçãoEscala para Worker 8x22BTarefa é "REPORT"Direto para Worker 8x22B (Qualidade crítica)Falha Geral (3x)Envia para Dead-letter Queue + Alerta Humano🚫 Anti-Patterns (O que NÃO fazemos)❌ Agentes conversam livremente: Nunca. Workers respondem a prompts estruturados com JSON.❌ Agente A pergunta ao Agente B: O Coordinator é o único que decide quem fala a seguir.❌ Agentes "negociam": Se o output é mau, o código escala. Não há debate.❌ Memória Implícita: Workers não lembram do passado. Todo o contexto necessário é injetado no prompt a cada passo.❌ LLM como Router: O routing é baseado em regras lógicas, não em "vibe" de um LLM.🛠 Tech Stack RecomendadaCoordinator: Python 3.10+ (Pydantic para validação)Orchestration: Temporal.io / Celery / ArqInference: vLLM / Ollama (para local) ou API Providers (Groq/OpenAI/Anthropic)Storage: PostgreSQL (JSONB) + Qdrant/Chroma (Vector)"Determinístico = previsível = debuggável."

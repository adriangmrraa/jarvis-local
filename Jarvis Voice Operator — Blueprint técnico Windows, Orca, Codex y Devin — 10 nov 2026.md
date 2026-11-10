# JARVIS VOICE OPERATOR — Especificación técnica e implementación en Windows

Guía de investigación, selección de repositorios, instalación, arquitectura y plan de construcción para agentes de programación. Versión 1.0 · 10 de noviembre de 2026 · Objetivo: asistente de voz operativo, local-first, conectado a Orca, Codex y Devin.

# 0\. Decisión ejecutiva

RECOMENDACIÓN: comenzar con el repositorio MIT Theycallmeinsane/jarvis como interfaz de voz nativa de Windows; conservar Orca instalado como gestor de worktrees y sesiones; construir una capa de orquestación propia, persistente y orientada a eventos (Voice Operator Core), con un adaptador Orca CLI y, cuando sea necesario, un adaptador Codex app-server. Evaluar OpenClaw Windows Hub como arquitectura alternativa si el prototipo exige gateway, sesiones y automatización 24/7; NO iniciar dos orquestadores principales simultáneamente.  
No existe evidencia de que uno solo de estos repositorios entregue de fábrica todas las capacidades requeridas. “100% funcionando” significa cumplir una batería explícita de pruebas en la máquina de destino, no prometer acceso infalible a cualquier aplicación. La PC indicada tiene 8 GB RAM; ejecutar modelos grandes de inferencia local junto con Orca, navegadores y worktrees no es una configuración apropiada. Mantener STT/TTS locales, tareas y herramientas en la PC; usar los modelos ya autenticados de Codex/Devin para razonamiento cuando resulte necesario. Local-first no equivale a operación completamente offline.  
PRINCIPIO RECTOR: la voz NO debe bloquearse mientras corren tareas. Conversación, ejecución, supervisión y notificaciones son subsistemas separados, conectados por una cola de eventos durable. Toda mutación de archivos/código sucede en un alcance identificado, preferentemente un worktree, nunca por defecto en main.

# 1\. Definición del producto y experiencia deseada

El usuario quiere hablar de manera continua con Jarvis mientras este inspecciona proyectos, investiga, utiliza skills y MCP, delega trabajos a Codex o Devin, crea worktrees en Orca, ejecuta tests, comitea en ramas aisladas, supervisa agentes y anuncia novedades sin interrumpir la conversación. La conversación debe tolerar cambios de foco y órdenes como “seguí con otro proyecto”, “¿cómo viene el agente?”, “mandale esta instrucción”, “cancelá solo esa tarea” y “no fusiones todavía”.  
Un ejemplo de éxito: “Jarvis, abrí Muzapp, creá dos worktrees, pedile a Codex revisar API y a Devin revisar Redis; mientras corren investigá cambios de WhatsApp Cloud API; cuando termine cada uno reportame diffs, pruebas y commits”. El usuario puede seguir conversando; hay tareas concurrentes y reportes posteriores. “Devin” puede ser remoto aunque se controle desde la máquina: verificar credenciales, licencias y canal disponible antes de ofrecer ejecución.  
Fuera de alcance del MVP: autonomía irrestricta sobre toda aplicación, evasión de UAC, lectura de contraseñas, envío de comunicaciones sin consentimiento, merges automáticos a main, despliegues productivos sin control y garantizar acciones GUI en apps que no exponen elementos accesibles.

# 2\. Proyectos existentes que vale la pena reutilizar

## 2.1. Selección priorizada

A. Theycallmeinsane/jarvis — https\://github.com/Theycallmeinsane/jarvis — MIT. Windows 10/11, wake word “Hey Jarvis”, atajo Ctrl+Alt+J, faster-whisper, VAD, Piper/Kokoro, bandeja del sistema, MCP, memoria, rutinas, controles del escritorio y opción de modelos compatibles. Incluye script setup.ps1, jarvis-tray.bat y tests. MEJOR CANDIDATO para clonar/estudiar como capa de voz liviana. ADVERTENCIA: comunidad muy pequeña, pocos commits y no debe asumirse madurez de producto; auditar código e instalador.  
B. Orca — https\://github.com/stablyai/orca — MIT; instalador Windows https\://github.com/stablyai/orca/releases/latest/download/orca-windows-setup.exe ; documentación CLI https\://www\.onorca.dev/docs/cli/overview y https\://www\.onorca.dev/docs/cli/reference ; orquestación https\://github.com/stablyai/orca/blob/main/skill-guides/orchestration.md . Fuente de verdad para worktrees, agents y terminales. NO clonar Orca para reconstruir sus funcionalidades: conectar con su CLI, cuyo comando se registra desde Settings → General → Orca CLI.  
C. Codex app-server — https\://learn.chatgpt.com/docs/app-server y https\://github.com/openai/codex . Protocolo JSON-RPC 2.0 por stdio, soporta sesiones, aprobaciones y eventos. Úsese para conversaciones persistentes e integración profunda; el transporte WebSocket está marcado como experimental. No sustituir Orca para gestionar sus worktrees.  
D. OpenClaw — https\://github.com/openclaw/openclaw y https\://docs.openclaw.ai/platforms/windows . Gateway con sesiones, tools, skills, nodos y Windows Hub nativo (WinUI), bandeja, voz/talk y system.run controlado. ALTERNATIVA robusta para el núcleo agéntico si el Jarvis de Windows no resiste pruebas; su Hub propone gateway WSL propio. Necesita evaluar memoria, servicios, permisos y complejidad de Windows/WSL2. No instalar junto con otro coordinador principal hasta decidir.  
E. Open Interpreter Desktop — https\://www\.openinterpreter.com/docs/desktop/install ; CLI en https\://github.com/openinterpreter/openinterpreter . La app ofrece control de software Windows y STT local; el repositorio CLI actual es un agente Rust basado en Codex y NO debe confundirse con los viejos ejemplos Python. Candidato para la capa Computer Use, sujeto a pruebas y permisos.  
F. OpenJarvis — https\://github.com/open-jarvis/OpenJarvis y https\://open-jarvis.github.io/OpenJarvis/getting-started/install/ . Agentes, memoria, herramientas y modelos locales/cloud, instalador PowerShell. Útil como biblioteca de capacidades o segunda alternativa de núcleo, pero NO como reemplazo automático de la capa de voz full-duplex ni de Orca. Evitar meter todo su stack Ollama/Rust si no hace falta en 8 GB.

## 2.2. Referencias complementarias, no obligatorias

FastVibe — https\://github.com/zzdhybthu/FastVibe — MIT, Kanban, SQLite, WebSocket, tareas de Codex/Claude y worktrees, voz desde UI web. Es una excelente referencia de arquitectura de cola, retomado tras reinicios y preguntas a agentes. Evitar correr FastVibe y Orca gestionando el MISMO repositorio al mismo tiempo: duplicarían estado y podrían competir por ramas.  
command-bridge — https\://github.com/JuanjoFuchs/command-bridge — interfaz STT/TTS local para varios agentes CLI. Referencia para conversación asíncrona y puente de voz. Revisar soporte y licencia antes de reutilizar código.  
Hark — https\://github.com/harkharness/hark — concepto casi idéntico (cockpit de voz para CLI Codex/Claude/ACP), pero está planteado para macOS. Usar de referencia UX y flujo, no como instalador Windows.  
Pipecat — https\://github.com/pipecat-ai/pipecat — framework en Python de voz en tiempo real y turn-taking, interrupciones, múltiples agentes. Elegirlo solo si el loop de voz del repositorio inicial falla las pruebas de barge-in/latencia; evita reconstruir audio desde cero, pero puede añadir complejidad.  
o8 — https\://github.com/hurttlocker/o8 — control multiagente, revisión, memoria y approvals; Windows dispone de previews según su autor, voz Symon específica de Mac. Referencia de calidad de control plane y revisión del commit exacto.  
01 — https\://github.com/OpenInterpreter/01 — plataforma experimental de voz/control de software, licencia AGPL-3.0. NO incorporarla al producto derivado sin revisar implicaciones de licencia, estado y advertencias explícitas de falta de salvaguardas.  
Vesper — https\://github.com/Awakelol/vesper-assistant — asistente Windows con Vosk, Whisper, Piper/Kokoro, GUI, \~60 herramientas y opción de Claude Code CLI. Referencia alternativa de acciones de escritorio.  
Onlyflames360/jarvis — https\://github.com/onlyflames360/jarvis — barge-in, STT local, Piper, wake word y registro de herramientas; ver advertencia de que el ZIP de descarga requiere que exista un release. Referencia de diseño de audio; no presumir que el instalador esté publicado.

## 2.3. Evidencia comunitaria (Reddit), no equivalente a pruebas independientes

r/ClaudeCode, 15/09/2026: autor de command-bridge describe alternar entre múltiples agentes mientras otros siguen procesando y un comentarista valora un “liaison” de voz que resume y traduce órdenes para el agente; https\://www\.reddit.com/r/ClaudeCode/comments/1wh7iqo/i\_wanted\_to\_use\_voice\_mode\_with\_multiple\_claude/ . Implicación: NO entregar toda la transcripción oral a cada coding agent; poner una capa conversacional de contexto delante.  
r/vibecoding, 15/03/2026: FastVibe, instrucciones por voz, worktrees y Kanban; https\://www\.reddit.com/r/vibecoding/comments/1rufwrs/why\_type\_when\_you\_can\_massdeploy\_claude\_code/ . Implicación: cada agente recibe una tarea atómica, no un mega-prompt.  
r/coding\_agents, 14/08/2026: creador de o8 describe la carga de coordinar varios agentes, reviews y decisiones; https\://www\.reddit.com/r/coding\_agents/comments/1vojarf/if\_you\_already\_use\_codex\_claude\_openclaw\_etc/ . Implicación: usar trazabilidad, revisión independiente y autoridad humana para merges.  
r/ClaudeWorkflows, octubre 2026: se describen guías de coordinación con worktrees, pruebas y voz, pero son relatos de usuarios, no evidencia de fiabilidad general; https\://www\.reddit.com/r/ClaudeWorkflows/comments/1xd2msw/workflow\_advanced\_claude\_code\_workflow\_multiagent/ .

# 3\. Arquitectura objetivo (fuente de verdad única)

Micrófono → Wake word / hotkey → VAD → STT local → Dialogue Manager  
                                              │  
                                              ├── respuesta TTS local (no bloqueante)  
                                              └── Intent / Plan → Task Manager (SQLite \+ eventos)  
                                                                   │  
                   ┌───────────────────────────────────────────────┼──────────────────────────┐  
                   ▼                                               ▼                          ▼  
            OrcaAdapter                                      LocalToolsAdapter          ResearchAdapter  
      worktrees, workers, logs                            PowerShell, Git, skills       web, docs, browser  
                   │                                               │                          │  
             Codex / Devin                                Computer Use seguro         evidencias  
                   └───────────────────────────────────────────────┴──────────────────────────┘  
                                                                   ▼  
                                                      estados, diffs, tests, eventos  
                                                                   ▼  
                                              notificaciones breves por voz \+ panel  
Componentes: (1) Voice Frontend mantiene escucha, endpointing, TTS y barge-in; (2) Dialogue Manager interpreta referencias (“ese worker”, “el segundo”), prioridades y confirma operaciones; (3) Task Manager transaccional persiste tareas/estados y reanuda después de reinicio; (4) Adapter Layer invoca herramientas sin mezclar salida de procesos con texto al usuario; (5) Policy Engine autoriza efectos; (6) Project Context Registry enlaza path, repo, rama, reglas y skills; (7) observabilidad muestra qué se hizo, no lo que el agente dijo que hizo.  
Para 8 GB: un único proceso STT local pequeño (modelo multilingüe tiny/base CPU), TTS Piper CPU y modelo de razonamiento vía Codex/servicio ya autorizado. Limitar inicialmente a 1–2 workers pesados, medir memoria real y solo incrementar tras pruebas. Si se usa WSL2/OpenClaw, asignar presupuesto de memoria y medir impacto en Orca; no asumir que todo entra.

# 4\. Instalación inicial en Windows — pasos ejecutables y comprobables

## 4.1. Preparación y auditoría NO destructiva

Abrir PowerShell normal (no Administrador salvo petición explícita de un instalador). No deshabilitar antivirus, UAC o políticas globales. Confirmar Windows 10/11 x64, micrófono y audífonos. Primero registrar entorno, versiones, ubicaciones y repos existentes; nunca sobrescribir configuraciones.  
\$PSVersionTable.PSVersion  
Get-CimInstance Win32\_OperatingSystem \| Select-Object Caption,Version,OSArchitecture  
Get-CimInstance Win32\_ComputerSystem \| Select-Object TotalPhysicalMemory  
Get-Command git,node,pnpm,python,py,uv,orca,codex,devin,opencode,ollama \-ErrorAction SilentlyContinue \| Select-Object Name,Source  
wsl \--list \--verbose  
\# Ejecutar SOLO los comandos disponibles:  
git \--version  
node \--version  
python \--version  
Guardar resultados en un inventario local en C:\\AI\\jarvis-operator\\docs\\ENVIRONMENT.md, sin tokens, claves ni datos confidenciales. Si Orca ya funciona, NO reinstalarlo: conservar estado y worktrees. En Orca habilitar Settings → General → Orca CLI; verificar su comando desde PowerShell, no solo Git Bash.

## 4.2. Herramientas base

Instalar únicamente dependencias faltantes; sus versiones pueden cambiar, por lo que revisar release notes y requisitos reales de cada repo. Git for Windows y Python 3.11/3.12 recomendados para el prototipo de voz, Node.js compatible con herramientas existentes. La instalación de Orca es por release oficial de Windows.  
\# Ejemplos con winget; ejecutar por separado solo si faltan:  
winget install \--id Git.Git \-e  
winget install \--id Python.Python.3.11 \-e  
winget install \--id OpenJS.NodeJS.LTS \-e  
\# Opcional si el proyecto lo exige:  
winget install \--id astral-sh.uv \-e  
\# Verificar tras abrir una terminal nueva:  
git \--version; py \-3.11 \--version; node \--version  
Si una entrada de winget no resuelve, usar el instalador oficial publicado por el proyecto y registrar versión exacta. No ejecutar instaladores remotos canalizados directamente a iex antes de inspeccionarlos. Reutilizar credenciales ya existentes de Codex/Orca: nunca copiarlas al repositorio.

## 4.3. Descargar y evaluar Jarvis de Windows en una carpeta separada

New-Item \-ItemType Directory \-Force C:\\AI\\sources \| Out-Null  
Set-Location C:\\AI\\sources  
git clone https\://github.com/Theycallmeinsane/jarvis.git jarvis-windows-upstream  
Set-Location .\\jarvis-windows-upstream  
git rev-parse HEAD  
Get-Content .\\setup.ps1 \-TotalCount 120  
Get-Content .\\requirements.txt \-TotalCount 120  
\# Tras revisión humana de script, dependencias y licencias:  
powershell \-ExecutionPolicy Bypass \-File .\\setup.ps1  
\# Lanzar según README del commit clonado:  
.\\jarvis-tray.bat  
El instalador oficial selecciona un perfil de modelo según hardware; en una máquina de 8 GB sin GPU potente evitar cargar qwen3.5:0.8b como cerebro encargado de decisiones de alto riesgo. Se puede usar como detector de intenciones simples y delegar el razonamiento al coordinador conectado a Codex; configurar la preferencia siguiendo el README del commit instalado. Antes de conectar la PC real, probar wake word, micrófono, TTS y una herramienta inocua (abrir una app de prueba).

## 4.4. Verificar Orca y skills

\# Orca debe estar abierto y la CLI registrada en Settings.  
orca \--help  
orca status \--json  
orca repo list \--json  
orca worktree list \--json  
orca skills \--help  
orca orchestration \--help  
orca computer capabilities \--json  
IMPORTANTE: comandos Orca dependen de la versión instalada. Si un subcomando no existe, consultar 'orca \--help' y la referencia CLI de esa versión, NO inventar flags. Instalar la skill oficial para los agentes solo si el comando de la versión lo permite: 'orca skills install \--skill orca-cli' o seguir https\://www\.onorca.dev/docs/cli/overview . El adaptador futuro deberá basarse en 'orca \--help' y JSON real del binario instalado.

## 4.5. Verificar Codex y Devin

codex \--version  
codex \--help  
codex app-server \--help  
\# Validar login con el mecanismo oficial de Codex del equipo:  
codex login status  
\# Solo comprobar Devin si existe como CLI:  
Get-Command devin \-ErrorAction SilentlyContinue  
Para el app-server de Codex, usar stdio por defecto desde un proceso hijo gestionado por el backend; implementar cliente JSONL/JSON-RPC con inicialización, conversaciones, eventos y aprobaciones de acuerdo con la documentación de la versión instalada. NO usar el listener WebSocket experimental como primera opción. Devin requiere descubrir su integración efectiva disponible en Orca, API o CLI; no presumir un ejecutable 'devin' o que la suscripción incluya un API adicional. Si no hay transporte comprobado, marcar Devin 'no disponible' y continuar con Codex.

# 5\. Repositorio de integración y estructura a generar

Crear un repositorio NUEVO, privado inicialmente, llamado 'jarvis-voice-operator'. Se puede hacer fork de Jarvis Windows o usar dependencia/adaptador externo. RECOMENDACIÓN: fork o copia auditada para cambios de voz, pero guardar el core de tareas en un repo propio; registrar upstream y commit de cada dependencia. No modificar carpetas de instalación o settings de Orca ni .codex directamente.  
C:\\AI\\jarvis-voice-operator\\  
  README.md  
  AGENTS.md  
  docs\\ARCHITECTURE.md  
  docs\\ENVIRONMENT.md  
  docs\\RUNBOOK-WINDOWS.md  
  docs\\SECURITY.md  
  docs\\ACCEPTANCE-TESTS.md  
  config\\operator.example.yaml  
  core\\api.py  
  core\\dialogue.py  
  core\\policy.py  
  core\\task\_manager.py  
  core\\event\_bus.py  
  core\\project\_registry.py  
  adapters\\orca\_cli.py  
  adapters\\codex\_app\_server.py  
  adapters\\devin.py  
  adapters\\windows\_tools.py  
  adapters\\skills.py  
  voice\\frontend\_bridge.py  
  storage\\migrations\\  
  scripts\\doctor.ps1  
  scripts\\start.ps1  
  scripts\\stop.ps1  
  tests\\unit\\  
  tests\\integration\\  
  tests\\e2e\\  
  .env.example  
  .gitignore  
El contenido anterior es una ESPECIFICACIÓN para que los agentes lo creen, no archivos ya existentes ni comandos que puedan ejecutarse inmediatamente. Preferir Python 3.11 \+ FastAPI \+ SQLite WAL \+ asyncio para el núcleo por la compatibilidad con Jarvis Windows y STT; GUI reutilizada inicialmente del repositorio de voz. No añadir React/Tauri hasta validar el flujo de voz y tres tareas simultáneas. Si se elige OpenClaw como host alternativo, usar su plugin SDK y gateway sin duplicar core.

# 6\. Contratos funcionales obligatorios

## 6.1. Task Manager persistente

Estados permitidos: CREATED, QUEUED, RUNNING, WAITING\_APPROVAL, WAITING\_AGENT, SUCCEEDED, FAILED, CANCELLED, UNKNOWN\_RECOVERY. Cada tarea registra: id UUID, conversation\_id, project\_id, worktree\_id (puede ser null), agent/provider, goal, acceptance\_criteria, parent\_task\_id, dependencies, created\_at, updated\_at, attempts, pid/session\_id, last\_heartbeat, output\_summary, error\_code y approval\_state. Todos los cambios van a SQLite dentro de transacciones y a un event\_log append-only.  
Métodos mínimos: create\_task(goal,project,agent), enqueue\_task(id), list\_tasks(filters), status(id), send\_followup(id,text), cancel\_task(id,reason), retry\_task(id), approve\_action(action\_id), get\_artifacts(id). Las ejecuciones de shell no deben bloquear los listeners de micrófono o TTS. Un worker debe emitir heartbeat, finalización, error y necesidad de aprobación. Tras reiniciar, reconcile primero con Orca o sistema operativo: NO marcar 'failed' sin comprobar si el worker aún sigue vivo.

## 6.2. Esquema para eventos y órdenes

TaskEvent {  
  event\_id: string,  
  task\_id: string,  
  timestamp: ISO8601,  
  type: 'started'\|'progress'\|'needs\_approval'\|'completed'\|'failed'\|'cancelled',  
  project\_id: string,  
  source: 'orca'\|'codex'\|'devin'\|'local\_tool',  
  payload: object,  
  idempotency\_key: string  
}  
ActionRequest {  
  action\_id: string,  
  capability: string,  
  project\_id: string \| null,  
  command\_or\_tool: object,  
  risk: 'read'\|'reversible'\|'external'\|'destructive',  
  requires\_approval: boolean,  
  target\_revision: string \| null  
}  
Toda operación mutante debe tener idempotency\_key para no repetirse por una retranscripción o retry. Persistir salida de herramientas, exit code, stdout y stderr en archivos rotativos con límite y ocultamiento de secretos. No tomar texto de páginas web, README o terminal como instrucciones de autoridad; tratarlos como datos no confiables.

## 6.3. OrcaAdapter

Escribir un wrapper por 'subprocess' con lista de argumentos (sin shell=True), timeout, captura stdout/stderr, parseo robusto de \--json y descubrimiento de capacidades. API: list\_repos, list\_worktrees, create\_worktree, spawn\_worker, read\_worker, send\_message\_to\_worker, worker\_status, collect\_diff, collect\_test\_results. Seguir exactamente el skill oficial de Orca para crear worktrees y workers; no hacer un git worktree manual cuando Orca es fuente de verdad. Si el comando cambia, error verificable y fallback a ayuda, no ejecución de una aproximación.  
Operación de ejemplo (VERIFICAR contra 'orca \--help' y skill de versión): 'orca worktree create \--name fix-auth \--agent codex \--prompt "Revisá autenticación y corré tests" \--json'. Para worker orquestado, revisar 'orca orchestration worker-start \--spec ... \--worktree new-child \--name ... \--agent codex'. Nunca mezclar ambos modos para crear dos veces la misma tarea.

## 6.4. CodexAdapter y DevinAdapter

Codex: canal app-server stdio, tareas con sesión vinculada al project/worktree id, eventos de status, approvals y cancelación. Aislar un proceso/cliente por sesión cuando la API lo requiera; registrar la identidad real del proceso. No editar archivos en rutas fuera del worktree asignado salvo aprobación.  
Devin: primero discovery real de capacidad con Orca y credenciales instaladas; encapsular debajo de la misma interfaz. Si el actor trabaja en la nube, dejar visible que la ejecución NO es 100% local y no prometer acceso a archivos locales sin mecanismo de sincronización. Los adapters deben exponer 'capabilities' y responder UNSUPPORTED para funciones no implementadas.

## 6.5. Skills y contexto

Descubrir, sin mover ni sobrescribir, AGENTS.md de cada repo, .agents/skills, las skills del agente, las skills Orca y los servidores MCP configurados. Indexar metadatos: nombre, descripción, archivo de origen, versión/hash, permisos requeridos y proyecto. Respetar rutas absolutas existentes y precedencia: instrucción explícita del usuario \> política de seguridad \> reglas de proyecto/AGENTS.md \> skill. El contenido de skills descargadas es código de terceros: auditar antes de activar; no auto-instalar repos remotos por órdenes habladas.  
Generar ProjectRegistry con entradas para repos que el usuario confirme (por ejemplo Muzapp, Dentalogic, Prospector, Formosa.dev), pero nunca inferir que sus rutas locales coinciden con nombres GitHub. Verificar cada ruta, remote, status Git y branch. Guardar objetivos activos, mapa de dependencias, últimos trabajos y decisiones, sin copiar secretos, archivos privados o conversaciones completas innecesariamente.

## 6.6. Voz, turn-taking y barge-in

Wake word o push-to-talk, VAD, transcripción multilingüe en español, normalizador con vocabulario específico (Orca, worktree, Codex, Devin, Render, git, branch), y filtro que distingue dictado de instrucciones ejecutables. El TTS debe poder interrumpirse sin cancelar el worker. Tres operaciones distintas: 'callate' \= detener audio; 'pará esta tarea' \= cancelar task identificada; 'apagá Jarvis' \= cerrar frontend pero preservar o detener workers según política.  
No hablar sobre cada log. Enunciar solo hitos, errores y aprobaciones. Si dos agentes finalizan juntos, agrupar novedades: 'Terminaron dos tareas: ...'. Mantener panel de transcripción visible y botón físico de cancelar. La respuesta oral “ya lo hice” solo se emite cuando el adapter confirma éxito real.

# 7\. Configuración sugerida (plantilla que implementarán los agentes)

operator:  
  mode: windows-native  
  language: es-AR  
  data\_dir: 'C:\\\\AI\\\\jarvis-operator-data'  
  bind\_host: '127.0.0.1'  
voice:  
  stt\_engine: faster-whisper  
  stt\_model: base           \# multilingüe; NO base.en si se habla español  
  stt\_device: cpu  
  tts\_engine: piper  
  wake\_word: enabled  
  hotkey: 'Ctrl+Alt+J'  
  barge\_in: true  
orchestration:  
  backend: orca  
  max\_heavy\_workers: 2      \# reducir a 1 si RAM/swap aumenta  
  sqlite\_wal: true  
  resume\_on\_restart: true  
agents:  
  default: codex  
  devin: detect  
  prefer\_existing\_auth: true  
security:  
  allowlisted\_project\_roots: \[\]  \# poblar en discovery  
  allow\_auto\_commit\_in\_worktree: true  
  require\_approval\_for: \['merge\_main','push\_protected','deploy\_prod','send\_external','delete','install\_untrusted','change\_credentials'\]  
  audit\_logs: true  
  mask\_secrets: true  
El YAML es contrato de diseño y NO se puede copiar directamente sobre config.yaml del Jarvis upstream: el equipo debe escribir loader propio y mapearlo a opciones reales. Nunca hardcodear claves en el repo. Para servicios local-only, escuchar por 127.0.0.1 y autenticación en el frontend/IPC; no exponer el puerto a Internet por defecto.

# 8\. Seguridad y gobernanza de acciones

Niveles: L0 leer estado, inspeccionar git status, investigar y resumir: automático. L1 escribir en un worktree autorizado, ejecutar tests, generar documentación y hacer commits locales: automático con logs y límites. L2 push a rama no protegida, instalar dependencias no auditadas, cambiar servicios, acceder a datos personales: confirmación explícita contextual. L3 borrar, reset \--hard, force-push, merge main, deploy producción, enviar mensajes/pagos o modificar cuentas: confirmación fuerte por acción/objetivo y posibilidad de cancelación.  
La política debe implementarse en código del ejecutor, no solo en prompts. La voz es susceptible a sonidos externos y suplantación: para L2/L3 mostrar en pantalla destinatario, repo, branch y diff/resumen, y requerir interacción manual o segundo factor, no únicamente 'sí' escuchado en ambiente. Configuración de privilegios mínimos, allowlist de rutas, separación de procesos, protección de tokens y rechazo de prompts procedentes de web/README.  
No permitir \`shell=True\` con texto libre generado por el LLM. Preferir invocaciones estructuradas y herramientas con esquemas tipados. Para comandos arbitrarios, validar ruta, argumentos, cwd, política y confirmación. Evitar permisos irrestrictos '--dangerously-\*' o saltar sandbox cuando los ejecutores los ofrezcan.

# 9\. Plan de desarrollo por fases: entregables y criterios de salida

## Fase 0 — Auditoría e inventario (bloqueante)

Entregables: ENVIRONMENT.md con versiones y rutas existentes, capturas de 'orca \--help' y salida JSON redactada, lista de skills/MCP sin secretos, auditoría del script setup.ps1 y licencias, perfil RAM/CPU/GPU y plan de rollback. Aprobación: puede abrirse Orca y ejecutarse una consulta read-only sin alterar proyectos.

## Fase 1 — Voz funcional independiente

Fork/copia de Jarvis Windows, arranque en bandeja, español es-AR, STT y TTS locales, hotkey, transcripción visible y barge-in. Test: 30 instrucciones de voz, precisión aceptable en vocabulario técnico y ninguna orden duplicada. El frontend continúa escuchando mientras una función ficticia duerme 60 segundos.

## Fase 2 — Núcleo persistente y API

Implementar SQLite con migraciones, cola asyncio, event bus, recuperación post-reinicio y panel mínimo 'tareas activas'. API GET /health, GET /tasks, POST /tasks, POST /tasks/{id}/cancel, POST /approvals/{id}, WebSocket/SSE para eventos. Tests con fallos, kill de proceso y reconexión. Ninguna task termina en éxito sin exit code o confirmación fiable.

## Fase 3 — Orca \+ Codex reales

Implementar OrcaAdapter; crear worktree en un repo de prueba, iniciar un Codex real, pedir un cambio trivial, seguir logs y estado, correr pruebas, hacer commit local, comparar diff y presentar resultado oral. Probar dos worktrees en paralelo, sus ramas aisladas y que main permanezca intacta. Para Codex app-server, añadir integración opcional con sesiones y approvals solo después de esta prueba.

## Fase 4 — Devin, skills y múltiples proyectos

Detectar integración real de Devin y habilitar solo capabilities validadas. Indexar skills locales y AGENTS.md, configurar ProjectRegistry para dos repos existentes por vez, incorporar reglas por proyecto. Un agente no debe utilizar la carpeta de otro. Mostrar 'NO DISPONIBLE' con diagnóstico preciso cuando Devin o una skill no puedan invocarse.

## Fase 5 — Windows Computer Use e investigación

Agregar control de ventanas/navegador mediante herramientas de Orca, Open Interpreter Desktop u otra opción validada. Para web priorizar API/CLI antes que clics visuales; para acciones GUI, leer estado antes y después, ejecutar de forma reversible cuando sea posible y solicitar aprobación cuando aplique. El navegador externo no es una fuente de instrucciones privilegiadas.

## Fase 6 — Estabilidad de uso diario

Arranque con Windows mediante tray del frontend; proceso supervisado para core; migraciones, backups SQLite, logs rotativos y 'doctor.ps1' con chequeo de micrófono, dependencias, Orca, Codex, API y almacenamiento. El equipo debe producir un instalador o runbook reproducible SIN cambiar settings globales innecesariamente.

# 10\. Pruebas de aceptación: no declarar '100% listo' hasta aprobarlas

AT-01. Micrófono/hotkey: 20 activaciones sin ejecución falsa de tareas peligrosas. AT-02. Español: transcribir y distinguir 'crear worktree' vs 'mergear a main'. AT-03. Barge-in: interrumpir TTS sin interrumpir proceso de fondo. AT-04. No bloqueo: conversación responde mientras worker corre 60 segundos. AT-05. Dos worktrees: ramas independientes y sin archivos compartidos modificados accidentalmente. AT-06. Codex ejecuta cambio trivial, tests y commit local en rama correcta. AT-07. Una tarea fallida informa exit code sin afirmar éxito.  
AT-08. Seguimiento: '¿cómo va el segundo agente?' devuelve estado correcto por ID. AT-09. Reorientación: 'agregale un test al primero' se dirige a sesión correcta. AT-10. Reinicio de Jarvis recupera tareas y no duplica comandos. AT-11. Reinicio de Orca produce estado UNKNOWN\_RECOVERY hasta reconciliar. AT-12. Falta Devin \-\> fallback honesto, sin simulación. AT-13. Skills: una skill autorizada se invoca con trazabilidad, sin ejecutar otra no listada.  
AT-14. Control de seguridad: rechazo de merge/push/deploy sin autorización. AT-15. Web prompt-injection: contenido externo malicioso no obtiene ejecución. AT-16. Límites RAM: medir consumo pico y degradación; si se excede presupuesto, reducir concurrencia automáticamente. AT-17. GUI: clic en app de prueba, verificación de pantalla posterior. AT-18. Sin credenciales en Git ni logs. AT-19. Las voces no anuncian 'completado' antes de completar. AT-20. La sesión conserva contexto del proyecto tras volver a abrir.  
Definición de terminado: scripts de instalación reproducibles, documentación de cada comando, pruebas unitarias/integración/e2e verdes, grabación o transcripción de escenario extremo a extremo, matriz de permisos, instrucciones de rollback y lista explícita de limitaciones verificadas en esa PC.

# 11\. Troubleshooting y decisiones condicionales

Orca no encontrado: abrir Orca, registrar CLI en Settings y reiniciar terminal; no hacer npm install de un 'orca' ajeno. JSON cambiado: fijar versión y adaptar parser; conservar salida completa para pruebas contractuales. Codex app-server no inicializa: probar 'codex app-server \--help', stdio, autenticación y protocolo exacto; usar CLI/Orca mientras se corrige.  
Devin no tiene CLI: no improvisar subcomandos; revisar proveedor y disponibilidad dentro de Orca, API o interfaz oficial. Audio no reconoce español: usar modelo multilingüe, configurar 'es', enumerar dispositivos y descartar 'base.en'. Eco/autoactivación: auriculares o cancelación de eco; separar mic y TTS. La voz se congela: mover herramientas fuera del event loop, usar queue y procesos con timeouts.  
8 GB RAM/swap alto: bajar a 1 worker, cerrar modelos Ollama grandes, limitar procesos Node, usar Whisper tiny/base CPU y Piper; abrir Administrador de tareas y medir. WSL2 consume memoria: usar Windows nativo para Jarvis+Orca inicialmente. Conflicto de ramas: no permitir que FastVibe/OpenClaw/Orca gestionen el mismo worktree como fuentes de verdad simultáneas.  
Acciones GUI inconsistentes: usar CLI/API primero; si no hay API, comprobar permisos, foco de ventana, display scale y elementos accesibles; fallar de manera explícita. Fallo de permisos PowerShell: invocar 'powershell \-ExecutionPolicy Bypass \-File script.ps1' solo para el proceso concreto tras revisión, no bajar políticas globales. Instalador bloqueado: verificar firma y procedencia oficial, no desactivar SmartScreen indiscriminadamente.

# 12\. Instrucciones para entregar a los agentes constructores

PROMPT MAESTRO: 'Actuá como arquitecto e implementador senior de sistemas agénticos Windows. Este documento es la especificación contractual. NO empieces clonando todos los repositorios: primero realizá Fase 0 de inventario read-only y entregá plan con riesgos y decisiones de compatibilidad; confirmá los flags reales de Orca/Codex/Devin instalados. Implementá fases 1 a 6 en commits pequeños, con tests, documentación y evidencia. Usá una sola fuente de verdad para worktrees (Orca) y una sola para tareas durables (Task Manager). Reutilizá scripts y autenticaciones existentes sin exponer secretos. No alteres main, no hagas pushes/despliegues/envíos externos sin aprobación. Ante capacidad inexistente, implementá adapter que reporte UNSUPPORTED; no inventes operaciones. Priorizá voz continua, tareas concurrentes y recuperación tras reinicios sobre efectos visuales. Al cerrar cada fase entregá: cambios, comandos ejecutados, pruebas, fallos conocidos y próximos pasos.'  
División inicial entre agentes: Arquitecto/integrador define contratos; Agente Voz trabaja solo en frontend y audio; Agente Core implementa Task Manager/SQLite y eventos; Agente Orca implementa adaptador y pruebas contractuales; Agente QA revisa casos de aceptación/seguridad. Arrancar con uno o dos workers en 8 GB. Ningún worker aprueba su propio merge; una tarea de QA independiente revisa diffs.

# 13\. Enlaces oficiales y material de referencia

GitHub seleccionados: https\://github.com/Theycallmeinsane/jarvis ; https\://github.com/stablyai/orca ; https\://github.com/openai/codex ; https\://github.com/openclaw/openclaw ; https\://github.com/open-jarvis/OpenJarvis ; https\://github.com/openinterpreter/openinterpreter ; https\://github.com/zzdhybthu/FastVibe ; https\://github.com/JuanjoFuchs/command-bridge ; https\://github.com/pipecat-ai/pipecat ; https\://github.com/harkharness/hark ; https\://github.com/hurttlocker/o8 ; https\://github.com/OpenInterpreter/01 ; https\://github.com/Awakelol/vesper-assistant ; https\://github.com/onlyflames360/jarvis .  
Documentación de instalación, API y permisos: https\://www\.onorca.dev/docs/cli/reference ; https\://www\.onorca.dev/docs/cli/overview ; https\://github.com/stablyai/orca/blob/main/skill-guides/orchestration.md ; https\://learn.chatgpt.com/docs/app-server ; https\://docs.openclaw.ai/platforms/windows ; https\://www\.openinterpreter.com/docs/desktop/install ; https\://open-jarvis.github.io/OpenJarvis/getting-started/install/ .  
Reddit (experiencias comunitarias, no auditoría): https\://www\.reddit.com/r/ClaudeCode/comments/1wh7iqo/i\_wanted\_to\_use\_voice\_mode\_with\_multiple\_claude/ ; https\://www\.reddit.com/r/vibecoding/comments/1rufwrs/why\_type\_when\_you\_can\_massdeploy\_claude\_code/ ; https\://www\.reddit.com/r/coding\_agents/comments/1vojarf/if\_you\_already\_use\_codex\_claude\_openclaw\_etc/ ; https\://www\.reddit.com/r/ClaudeWorkflows/comments/1xd2msw/workflow\_advanced\_claude\_code\_workflow\_multiagent/ .

# 14\. Criterio final de arquitectura

Primera vía: JARVIS WINDOWS (voz) \+ CORE PERSISTENTE (desarrollar) \+ ORCA (worktrees) \+ CODEX/DEVIN (trabajadores) \+ SKILLS/MCP \+ COMPUTER USE por capacidades. Camino alternativo: OPENCLAW WINDOWS HUB como host de sesiones/herramientas, integrado al mismo Orca, si la primera vía no supera pruebas; evaluar audio continuo y consumo de RAM antes de migrar. OpenJarvis, FastVibe, Hark, command-bridge y Pipecat aportan patrones o módulos, pero NO son todos dependencias obligatorias.  
Resultado buscado: trabajar hablando de manera natural mientras los agentes ejecutan tareas reales, verificadas y auditables. El producto está completo solamente cuando funciona en la PC concreta y supera las pruebas descritas. Investigación revisada a 10/11/2026; fijar versiones/commit hashes durante Fase 0 para protegerse de cambios posteriores en repositorios.  

<script setup lang="ts">
import { ref, onMounted, watch, computed, onUnmounted } from "vue";

type Transition = [read: string, write: string, direction: "L" | "R", nextStage: State];
type AuxiliaryAlphabet = [startSymbol: string, markSymbol: string, blankSymbol: string];
type Coord = [X: number, Y: number];

const word = ref<string>("abbaba");
const head = ref<number>(1);

const states = ref<State[]>([]);
const currentState = ref<State>();
const finalStates = ref<State[]>([]);
const auxiliaryAlphabet = ref<AuxiliaryAlphabet>(["*", "X", "B"]);

const contextMenuRef = ref<HTMLUListElement | null>(null);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const ctx = computed(() => canvasRef.value?.getContext("2d") as CanvasRenderingContext2D);
const tapeRef = ref<HTMLDivElement | null>(null);

const tape = ref<string>(auxiliaryAlphabet.value[0] + word.value + auxiliaryAlphabet.value[2]);

class State {
  private _label: string;
  private _transitions: Transition[];
  public _pos: Coord; // Alterado para public para facilitar o acesso na renderização externa
  private _joints: Coord[] = [];
  private _radius: number = 40;

  constructor(label?: string, pos?: Coord) {
    if (label) this._label = label;
    else this._label = `q${states.value.length}`;

    this._transitions = [];
    this._pos = pos ?? [0, 0];

    for (let angle = 0; angle < 360; angle += 45) {
      const rad = angle * (Math.PI / 180);
      const x = this._pos[0] + this._radius * Math.cos(rad);
      const y = this._pos[1] + this._radius * Math.sin(rad);
      this._joints.push([x, y]);
    }

    states.value.push(this);
  }

  get label(): string { return this._label; }
  set label(label: string) { this._label = label; }
  get transitions(): Transition[] { return this._transitions; }

  addTransition(transition: Transition) {
    this._transitions.push(transition);
  }

  removeTransition(transition: Transition | number) {
    if (typeof transition === "number") this.transitions.splice(transition, 1);
    else {
      const index = this._transitions.indexOf(transition);
      if (index > -1) this._transitions.splice(index, 1);
      else throw new Error("Transition not found");
    }
  }

  executeTransition(): State {
    for (const transition of this._transitions) {
      const [read, write, direction, nextState] = transition;
      if (tape.value[head.value] === read) {
        tape.value = tape.value.substring(0, head.value) + write + tape.value.substring(head.value + 1);
        head.value += direction === "L" ? -1 : 1;
        if (head.value < 0) head.value = 0;
        if (head.value >= tape.value.length) head.value = tape.value.length - 1;
        return nextState;
      }
    }
    throw new Error("No applicable transition found");
  }

  drawState(): void {
    if (!ctx.value) return;
    const [x, y] = this._pos;

    ctx.value.beginPath();
    ctx.value.arc(x, y, this._radius, 0, 2 * Math.PI);
    ctx.value.fillStyle = currentState.value === this ? "#ffe082" : "#ffffff";
    ctx.value.fill();
    ctx.value.lineWidth = finalStates.value.includes(this) ? 4 : 2;
    ctx.value.strokeStyle = "#333333";
    ctx.value.stroke();
    
    ctx.value.fillStyle = "#333333";
    ctx.value.font = "bold 16px sans-serif";
    ctx.value.textAlign = "center";
    ctx.value.textBaseline = "middle";
    ctx.value.fillText(this._label, x, y);
  }

  drawTransitions(): void {
    if (!ctx.value) return;
    const [x, y] = this._pos;
    const transitionCounts = new Map<State, [number, Coord]>();

    for (const transition of this._transitions) {
      const [read, write, direction, nextState] = transition;
      const [nextX, nextY] = nextState._pos;

      if (transitionCounts.has(nextState)) {
        const coord = transitionCounts.get(nextState)?.[1] as Coord;
        const count = transitionCounts.get(nextState)?.[0] as number;
        ctx.value.font = "12px sans-serif";
        ctx.value.fillStyle = "#555555";
        ctx.value.fillText(`${read} → ${write}, ${direction}`, coord[0], coord[1] - (14 * count));
        transitionCounts.set(nextState, [count + 1, coord]);
        continue;
      }

      let closestJoints: [Coord, Coord] = [this._pos, nextState._pos];
      let closestDistance: number = Number.POSITIVE_INFINITY;

      let startI: number, endI: number, startJ: number, endJ: number;

      if ((x < nextX) && (y >= nextY)) { startI = 6; endI = 9; startJ = 2; endJ = 5; }
      else if ((x > nextX) && (y >= nextY)) { startI = 4; endI = 7; startJ = 0; endJ = 3; }
      else if ((x > nextX) && (y < nextY)) { startI = 2; endI = 5; startJ = 6; endJ = 9; }
      else { startI = 0; endI = 3; startJ = 4; endJ = 7; }

      for (let i = startI; i < endI; i++) {
        const joint = i < this._joints.length ? this._joints[i] as Coord : this._joints[i - this._joints.length] as Coord;
        for (let j = startJ; j < endJ; j++) {
          const nextJoint = j < nextState._joints.length ? nextState._joints[j] as Coord : nextState._joints[j - nextState._joints.length] as Coord;
          const distance = Math.hypot(joint[0] - nextJoint[0], joint[1] - nextJoint[1]);
          if (distance < closestDistance) {
            closestDistance = distance;
            closestJoints = [joint, nextJoint];
          } 
        }
      }

      const cpX = (closestJoints[0][0] + closestJoints[1][0]) / 2;
      const cpY = (closestJoints[0][1] + closestJoints[1][1]) / 2 - 80;

      ctx.value.beginPath();
      ctx.value.moveTo(closestJoints[0][0], closestJoints[0][1]);
      ctx.value.quadraticCurveTo(cpX, cpY, closestJoints[1][0], closestJoints[1][1]);
      ctx.value.strokeStyle = "#4f46e5";
      ctx.value.lineWidth = 2;
      ctx.value.stroke();

      const dx = closestJoints[1][0] - cpX;
      const dy = closestJoints[1][1] - cpY;
      const angle = Math.atan2(dy, dx);

      const arrowLength = 12;
      ctx.value.beginPath();
      ctx.value.fillStyle = "#4f46e5";
      ctx.value.moveTo(closestJoints[1][0], closestJoints[1][1]);
      ctx.value.lineTo(
        closestJoints[1][0] - arrowLength * Math.cos(angle - Math.PI / 6),
        closestJoints[1][1] - arrowLength * Math.sin(angle - Math.PI / 6)
      );
      ctx.value.lineTo(
        closestJoints[1][0] - arrowLength * Math.cos(angle + Math.PI / 6),
        closestJoints[1][1] - arrowLength * Math.sin(angle + Math.PI / 6)
      );
      ctx.value.closePath();
      ctx.value.fill();

      ctx.value.font = "12px sans-serif";
      ctx.value.fillStyle = "#333333";
      ctx.value.fillText(`${read} → ${write}, ${direction}`, cpX, cpY + 35);

      transitionCounts.set(nextState, [1, [cpX, cpY + 35]]);
    }
  }
}

// Renderizador Global
function renderCanvas() {
  if (!ctx.value || !canvasRef.value) return;
  ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);
  
  // Desenha caminhos primeiro para os nós ficarem por cima
  states.value.forEach(state => state.drawTransitions());
  states.value.forEach(state => state.drawState());
}

function updateCtxMenuPos(pos: Coord): void {
  if (!contextMenuRef.value || !canvasRef.value) return;

  const { x: canvasX, y: canvasY } = canvasRef.value.getBoundingClientRect();
  const [x, y] = pos;

  contextMenuRef.value.style.left = `${x}px`;
  contextMenuRef.value.style.top = `${y}px`;
}

function moveTape() {
  if (!tapeRef.value || !tapeRef.value.children.length) return;
  const tapeSlotsLength = Array.from(tapeRef.value.children).map((slot) => slot.getBoundingClientRect().width);
  tapeRef.value.style.transform = `translateX(${
    (tapeRef.value.getBoundingClientRect().width / 2) - tapeSlotsLength.slice(0, head.value).reduce((sum, value) => sum + value, 0) - (tapeSlotsLength.at(head.value) as number / 2)
  }px)`;
}

function iterate() {
  try {
    currentState.value = currentState.value?.executeTransition();
    moveTape();
    renderCanvas(); // Redesenha para atualizar o estado ativo amarelo
  } catch (error) {
    console.error(error);
  }
}

function createNewState(e: PointerEvent): State {
  if (!canvasRef.value) throw new Error("No defined Canvas");

  const { x: canvasX, y: canvasY } = canvasRef.value.getBoundingClientRect();
  const state = new State(undefined, [e.clientX - canvasX, e.clientY - canvasY]);
  renderCanvas();
  return state;
}

let resizeObserver: ResizeObserver | null = null;

onMounted(() => {
  // Configuração do ResizeObserver para tornar o Canvas responsivo
  resizeObserver = new ResizeObserver((entries) => {
    for (let entry of entries) {
      const { width, height } = entry.contentRect;
      if (canvasRef.value) {
        canvasRef.value.width = width;
        canvasRef.value.height = height;
        renderCanvas();
      }
    }
  });

  if (canvasRef.value?.parentElement) {
    resizeObserver.observe(canvasRef.value.parentElement);
  }

  // Evento de Menu de Contexto
  canvasRef.value!.addEventListener("contextmenu", (e) => {
    e.preventDefault();
    updateCtxMenuPos([e.clientX, e.clientY]);
    contextMenuRef.value!.style.visibility = "visible";
  });

  document.addEventListener("click", () => {
    if (contextMenuRef.value) contextMenuRef.value.style.visibility = "hidden";
  });

  // Inicialização do Autômato Exemplo
  const [startSymbol, markSymbol, blankSymbol] = auxiliaryAlphabet.value;

  const q0 = new State("q0", [150, 250]);
  const q1 = new State("q1", [350, 150]);
  const q2 = new State("q2", [550, 150]);
  const q3 = new State("q3", [750, 250]);
  const q4 = new State("q4", [550, 350]);
  const q5 = new State("q5", [350, 350]);
  const q6 = new State("q6", [150, 450]);

  q0.addTransition(["a", markSymbol, "R", q1]);
  q0.addTransition(["b", markSymbol, "R", q4]);
  q0.addTransition([markSymbol, markSymbol, "R", q0]);
  q0.addTransition([blankSymbol, blankSymbol, "R", q6]);

  q1.addTransition(["a", "a", "R", q1]);
  q1.addTransition(["b", markSymbol, "R", q2]);

  q2.addTransition(["b", "b", "R", q2]);
  q2.addTransition(["a", "a", "R", q2]);
  q2.addTransition([blankSymbol, blankSymbol, "L", q3]);

  q3.addTransition(["a", "a", "L", q3]);
  q3.addTransition(["b", "b", "L", q3]);
  q3.addTransition([markSymbol, markSymbol, "L", q3]);
  q3.addTransition([startSymbol, startSymbol, "R", q0]);

  q4.addTransition(["b", "b", "R", q4]);
  q4.addTransition(["a", markSymbol, "R", q5]);

  q5.addTransition(["a", "a", "R", q5]);
  q5.addTransition(["b", "b", "R", q5]);
  q5.addTransition([blankSymbol, blankSymbol, "L", q3]);

  currentState.value = q0;
  finalStates.value = [q6];

  setTimeout(() => {
    renderCanvas();
    moveTape();
  }, 50);

  watch(word, (newWord) => {
    tape.value = auxiliaryAlphabet.value[0] + newWord + auxiliaryAlphabet.value[2];
    head.value = 1;
    currentState.value = q0;
    setTimeout(() => {
      moveTape();
      renderCanvas();
    }, 0);
  });
});

onUnmounted(() => {
  if (resizeObserver) resizeObserver.disconnect();
});
</script>

<template>
  <div class="app-container">
    <header class="main-header">
      <div class="brand">
        <h1>Máquina de Turing</h1>
        <span class="status-badge">Estado Atual: <strong>{{ currentState?.label }}</strong></span>
      </div>
      <div class="controls-card">
        <div class="input-group">
          <label for="i-word">Palavra de Entrada:</label>
          <input id="i-word" v-model="word" placeholder="Ex: abbaba" />
        </div>
        <button class="btn-primary" @click="iterate" title="Executar Próximo Passo">
          <span class="icon">▶</span> Próximo Passo
        </button>
      </div>
    </header>

    <div class="workspace">
      <aside class="sidebar">
        <section class="info-section">
          <h2>Definição Formal</h2>
          <div class="meta-grid">
            <div class="meta-item"><strong>Alfabeto:</strong> <span>{ {{ [...new Set(word)].join(", ") }} }</span></div>
            <div class="meta-item"><strong>Auxiliar:</strong> <span>{ {{ auxiliaryAlphabet.join(", ") }} }</span></div>
            <div class="meta-item"><strong>Finais:</strong> <span class="badge-final">{{ finalStates.map(s => s.label).join(", ") }}</span></div>
          </div>
        </section>

        <section class="info-section transitions-list-wrapper">
          <h2>Estados & Transições</h2>
          <ul id="states-list">
            <li v-for="(state, index) of states" :key="index" :class="{ active: currentState === state }">
              <div class="state-header">
                <span class="state-node-name">{{ state.label }}</span>
                <span v-if="finalStates.includes(state)" class="sub-badge">Final</span>
              </div>
              <ul class="transition-sublist">
                <li v-for="(transition, tIdx) of state.transitions" :key="tIdx">
                  <span class="t-trigger">'{{ transition[0] }}'</span> → 
                  <span class="t-action">{{ transition[1] }}, {{ transition[2] }}</span> 
                  <span class="t-target">➔ {{ transition[3].label }}</span>
                </li>
                <li v-if="state.transitions.length === 0" class="empty-transitions">Nenhuma transição</li>
              </ul>
            </li> 
          </ul>
        </section>
      </aside>

      <main class="content-area">
        <div id="canva-area">
          <ul ref="contextMenuRef" id="context-menu">
            <li @click="createNewState">➕ Criar Novo Estado</li>
          </ul>
          <canvas id="canva" ref="canvasRef"></canvas>
        </div>

        <div id="machine-area">
          <div class="head-indicator">
            <svg width="20" height="15" viewBox="0 0 100 100" preserveAspectRatio="none">
              <polygon points="0,0 100,0 50,100" />
            </svg>
            <span>Cabeçote</span>
          </div>
          <div class="tape-window">
            <div class="tape" ref="tapeRef">
              <div v-for="(char, index) in tape" :key="index" class="tape-cell" :class="{ 'head-active': index === head }">
                {{ char }}
              </div>
            </div>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>

<style scoped>
/* Reset base e variáveis de escopo local */
.app-container {
  --bg-primary: #f8fafc;
  --bg-surface: #ffffff;
  --border-color: #e2e8f0;
  --text-main: #1e293b;
  --primary-color: #4f46e5;
  --primary-hover: #4338ca;
  --accent-active: #ffe082;
  
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: var(--text-main);
  background-color: var(--bg-primary);
  min-height: 100vh;
  max-height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* Header Estilizado */
.main-header {
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border-color);
  padding: 0.75rem 1.5rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
  z-index: 10;
}
.brand h1 {
  font-size: 1.25rem;
  margin: 0;
  font-weight: 700;
  color: #0f172a;
}
.status-badge {
  font-size: 0.85rem;
  color: #64748b;
}
.controls-card {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}
.input-group {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.input-group label {
  font-size: 0.9rem;
  font-weight: 500;
  color: #475569;
}
.input-group input {
  padding: 0.5rem 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-family: monospace;
  font-size: 1rem;
  width: 160px;
  outline: none;
}
.input-group input:focus {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.15);
}
.btn-primary {
  background: var(--primary-color);
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  font-size: 0.95rem;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: background 0.2s;
}
.btn-primary:hover { background: var(--primary-hover); }

/* Layout Workspace */
.workspace {
  flex: 1;
  display: flex;
  min-height: 0;
}

/* Sidebar */
.sidebar {
  width: 320px;
  background: var(--bg-surface);
  border-right: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  padding: 1.25rem;
  overflow-y: auto;
}
.info-section h2 {
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin: 0 0 0.75rem 0;
  border-bottom: 1px solid var(--border-color);
  padding-bottom: 0.25rem;
}
.meta-grid {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  font-size: 0.9rem;
}
.meta-item {
  display: flex;
  justify-content: space-between;
  background: #f8fafc;
  padding: 0.4rem 0.6rem;
  border-radius: 4px;
}
.badge-final {
  background: #fee2e2;
  color: #ef4444;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  font-weight: bold;
}

/* Listagem de Estados e Transições */
.transitions-list-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
}
#states-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  overflow-y: auto;
}
#states-list > li {
  border: 1px solid var(--border-color);
  border-radius: 6px;
  padding: 0.5rem;
  background: #fdfdfd;
  transition: all 0.2s;
}
#states-list > li.active {
  border-color: #f59e0b;
  background: #fffbeb;
  box-shadow: 0 2px 4px rgba(245, 158, 11, 0.05);
}
.state-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 600;
  margin-bottom: 0.25rem;
}
.state-node-name {
  background: #e2e8f0;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  font-size: 0.85rem;
}
.sub-badge { font-size: 0.75rem; color: #ef4444; font-weight: bold; }
.transition-sublist {
  list-style: none;
  padding-left: 0.5rem;
  margin: 0;
  font-family: monospace;
  font-size: 0.85rem;
  color: #475569;
}
.transition-sublist li {
  padding: 0.15rem 0;
}
.t-trigger { color: #b45309; font-weight: bold; }
.t-action { color: #1e1b4b; }
.t-target { color: #4f46e5; font-weight: bold; }
.empty-transitions { color: #94a3b8; font-style: italic; }

/* Central Content Area (Canvas + Tape) */
.content-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}
#canva-area {
  flex: 1;
  position: relative;
  min-height: 0;
  background: #ffffff;
}
#canva {
  display: block;
  width: 100%;
  height: 100%;
}

/* Context Menu */
#context-menu {
  position: fixed;
  background: white;
  border: 1px solid var(--border-color);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  padding: 0.25rem 0;
  list-style: none;
  margin: 0;
  border-radius: 6px;
  z-index: 100;
  visibility: hidden;
}
#context-menu li {
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background 0.15s;
}
#context-menu li:hover { background: #f1f5f9; }

/* Fita (Tape Section) */
#machine-area {
  background: #f8fafc;
  border-top: 1px solid #334155;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
}
.head-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #ef4444;
  font-size: 0.75rem;
  font-weight: bold;
  margin-bottom: 0.25rem;
  z-index: 2;
}
.head-indicator svg {
  fill: #ef4444;
  width: 16px;
  height: 12px;
}
.tape-window {
  width: 100%;
  max-width: 800px;
  overflow: hidden;
  border: 2px solid #334155;
  background: #1e293b;
  border-radius: 8px;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.3);
}
.tape {
  display: flex;
  transition: transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  width: max-content;
}
.tape-cell {
  width: 60px;
  min-width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: monospace;
  font-size: 2rem;
  font-weight: bold;
  color: #f8fafc;
  border-right: 1px solid #334155;
  transition: background 0.2s, color 0.2s;
}
.tape-cell.head-active {
  background: #eab308;
  color: #0f172a;
}
</style>
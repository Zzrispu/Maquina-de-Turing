<script setup lang="ts">
import { ref, onMounted, watch, reactive } from "vue";

type AuxiliaryAlphabet = [startSymbol: string, markSymbol: string, blankSymbol: string];
type Coord = [X: number, Y: number];
type TransitionGeometry = {
  startJoint: Coord;
  endJoint: Coord;
  controlPoint: Coord;
  labelPos: Coord;
}

const word = ref<string>("abbaba");
const head = ref<number>(1);

const states = ref<State[]>([]);
const stateByElement = new Map<HTMLDivElement, State>();
const transitions = ref<Transition[]>([]);
const transitionByElement = new Map<Element, Transition>();

const currentState = ref<State>();
const inicialState = ref<State>();
const finalStates = ref<State[]>([]);
const auxiliaryAlphabet = ref<AuxiliaryAlphabet>(["*", "X", "B"]);

watch(currentState, (newState, oldState) => {
  newState?._element?.classList.add('current-state');
  oldState?._element?.classList.remove('current-state');
});

watch(inicialState, (newState, oldState) => {
  if (!newState || !newState._element) return;
  newState._element.classList.add('inicial-state');
  oldState?._element?.classList.remove('inicial-state');
});

const focusedState = ref<State>();
const focusedTransition = ref<Transition>();

const canvaAreaRef = ref<HTMLElement | null>(null);
const contextMenuRef = ref<HTMLUListElement | null>(null);

const tapeRef = ref<HTMLDivElement | null>(null);

const tape = ref<string>(auxiliaryAlphabet.value[0] + word.value + auxiliaryAlphabet.value[2]);

class Transition {
  public read: string;
  public write: string;
  public direction: "L" | "R";
  public toState: State;
  public stackCount: number = 0;

  private _fromState: State;

  public elementId: string = "";

  constructor(read: string, write: string, direction: "L" | "R", fromState: State, toState: State) {
    this.read = read;
    this.write = write;
    this.direction = direction;
    this.toState = toState;
    this._fromState = fromState;

    this.updateGeometry();
    transitions.value.push(this);
  }

  private findClosestJoints(): [Coord, Coord] {
    const [x, y] = this._fromState.pos;
    const [nextX, nextY] = this.toState.pos;

    let closestJoints: [Coord, Coord] = [[x, y], [nextX, nextY]];
    let closestDistance: number = Number.POSITIVE_INFINITY;

    let startI: number, endI: number, startJ: number, endJ: number;

    // Proximo estado está no primeiro quadrante em relação ao estado atual como origem
    if ((x < nextX) && (y >= nextY)) {startI = 6; endI = 9; startJ = 2; endJ = 5;    }
    // Segundo quadrante
    else if ((x > nextX) && (y >= nextY)) {startI = 4; endI = 7; startJ = 0; endJ = 3;}
    // Terceiro quadrante
    else if ((x > nextX) && (y < nextY)) {startI = 2; endI = 5; startJ = 6;endJ = 9;}
    // Quarto quadrante
    else {startI = 0; endI = 3; startJ = 4; endJ = 7;}

    for (let i = startI; i < endI; i++) {
      const joint = i < this._fromState.joints.length
        ? this._fromState.joints[i] as Coord
        : this._fromState.joints[i - this._fromState.joints.length] as Coord;

      for (let j = startJ; j < endJ; j++) {
        const nextJoint = j < this.toState.joints.length
          ? this.toState.joints[j] as Coord
          : this.toState.joints[j - this.toState.joints.length] as Coord;

        const distance = Math.hypot(joint[0] - nextJoint[0], joint[1] - nextJoint[1]);
        if (distance < closestDistance) {
          closestDistance = distance;
          closestJoints = [joint, nextJoint];
        } 
      }
    }

    return closestJoints;
  }

  public geometry = reactive<TransitionGeometry>({
    startJoint: [0, 0],
    endJoint: [0, 0],
    controlPoint: [0, 0],
    labelPos: [0, 0],
  });

  updateGeometry(): void {
    if (!canvaAreaRef.value) return;

    const [[startX, startY], [endX, endY]] = this.findClosestJoints();

    const cpX = (startX + endX) / 2;
    const cpY = (startY + endY) / 2 - 80;

    this.geometry.startJoint = [startX, startY];
    this.geometry.endJoint = [endX, endY];
    this.geometry.controlPoint = [cpX, cpY];
    this.geometry.labelPos = [cpX + 12, cpY + 12];
  }

  retarget(newToState: State): void {
    this.toState._removeInTransition(this);
    this.toState = newToState;
    newToState._inTransitions.push(this);
    this.updateGeometry();
  }

  destroy(): void {
    this._fromState._removeOutTransition(this);
    this.toState._removeInTransition(this);

    const globalIndex = transitions.value.indexOf(this);
    if (globalIndex > -1) transitions.value.splice(globalIndex, 1);
  }

  get fromState(): State {
    return this._fromState
  }

  get labelOffSet(): Coord {
    return [this.geometry.labelPos[0], this.geometry.labelPos[1] - this.stackCount * 15];
  }

  get label(): string {
    return `${this.read}, ${this.write}; ${this.direction}`;
  }

  get pathD(): string {
    const { startJoint, controlPoint, endJoint } = this.geometry;
    return `M ${startJoint[0]} ${startJoint[1]} Q ${controlPoint[0]} ${controlPoint[1]} ${endJoint[0]} ${endJoint[1]}`;
  }
}

class State {
  private _label: string;
  public _inTransitions: Transition[];
  private _outTransitions: Transition[];
  public _element: HTMLDivElement | null = null;

  private _pos: Coord =  [0, 0];
  private _dragOffSet: Coord = [0, 0];
  private _joints: Coord[] = [];
  public _radius: number = 40;

  private updateJoints = (): void => {
    this._joints = [];
    for (let angle = 0; angle < 360; angle += 45) {
      const rad = angle * (Math.PI / 180);
      const x = (this.pos[0] + this._radius) + this._radius * Math.cos(rad);
      const y = (this.pos[1] + this._radius) + this._radius * Math.sin(rad);
      this._joints.push([x, y]);
    }
  }

  private applyElementPosition(): void {
    if (!canvaAreaRef.value) return;

    const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();
    if (this._element) {
      this._element.style.left = `${this._pos[0] + canvaX}px`;
      this._element.style.top = `${this._pos[1] + canvaY}px`;
    }
  }

  public _removeInTransition(transition: Transition): void {
    const index = this._inTransitions.indexOf(transition);
    if (index > -1) this._inTransitions.splice(index, 1);
  }

  public _removeOutTransition(transition: Transition): void {
    const index = this._outTransitions.indexOf(transition);
    if (index > -1) this._outTransitions.splice(index, 1);
  }

  constructor(label?: string, pos?: Coord) {
    if (label) this._label = label;
    else this._label = `q${states.value.length}`;

    this._inTransitions = [];
    this._outTransitions = [];
    this.pos = pos ?? [0, 0];

    states.value.push(this);
  }

  get label(): string {
    return this._label;
  }

  set label(label: string) {
    this._label = label;
  }

  get transitions(): Transition[] {
    return this._outTransitions;
  }

  get pos(): Coord {
    return this._pos;
  }

  set pos(pos: Coord) {
    this._pos = pos;
    this.updateJoints();
    this._outTransitions.forEach(t => t.updateGeometry());
    this._inTransitions.forEach(t => t.updateGeometry());

    this.applyElementPosition();
  }

  get joints(): Coord[] {
    return this._joints;
  }

  addTransition(read: string, write: string, direction: "L" | "R", toState: State): Transition {
    const transition = new Transition(read, write, direction, this, toState);
    transition.stackCount = this._outTransitions.filter(ot => ot.toState === toState).length;
    toState._inTransitions.push(transition);
    this._outTransitions.push(transition);

    transition.elementId = this.label + toState.label + transition.stackCount;

    return transition;
  }

  removeTransition(transition: Transition | number) {
    if (typeof transition === "number") this.transitions.splice(transition, 1);
    else {
      const index = this._outTransitions.indexOf(transition);
      if (index > -1) this._outTransitions.splice(index, 1);
      else throw new Error("Transition not found");
    };
  }

  executeTransition(): State {
    for (const transition of this._outTransitions) {
      const {read, write, direction, toState} = transition;
      if (tape.value[head.value] === read) {
        tape.value = tape.value.substring(0, head.value) + write + tape.value.substring(head.value + 1);
        head.value += direction === "L" ? -1: 1;
        if (head.value < 0) head.value = 0;
        if (head.value >= tape.value.length) head.value = tape.value.length - 1;
        return toState;
      }
    }
    throw new Error("No applicable transition found");
  }

  private mouseMove = (e: MouseEvent): void => {
    if (canvaAreaRef.value === null) return;

    const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();

    this.pos = [e.clientX - canvaX - this._dragOffSet[0], e.clientY - canvaY - this._dragOffSet[1]];
  }

  private mouseUp = (e: MouseEvent): void => {
    if (canvaAreaRef.value === null) return;

    canvaAreaRef.value.removeEventListener('mousemove', this.mouseMove);
    canvaAreaRef.value.removeEventListener('mouseup', this.mouseUp);
  }

  private mouseDown = (e: MouseEvent): void => {
    if (this._element === null || canvaAreaRef.value === null) return;

    switch (e.button) {
      case 0:
        const rect = this._element.getBoundingClientRect();
        this._dragOffSet = [e.clientX - rect.left, e.clientY - rect.top];

        canvaAreaRef.value.addEventListener('mousemove', this.mouseMove);
        canvaAreaRef.value.addEventListener('mouseup', this.mouseUp);
        break;

      case 2:
        focusedState.value = this;
        break;
    
      default:
        break;
    }
  }

  renderElement(): void {
    if (!canvaAreaRef.value) throw new Error("No Canvas Area defined");

    const element = document.createElement("div");
    element.classList.add("state-div");
    element.style.position = "fixed";
    element.style.aspectRatio = "1/1";
    element.style.width = `${this._radius * 2}px`;
    element.style.borderRadius = `50%`;
    element.style.display = `grid`;
    element.style.placeItems = `center`;
    element.style.cursor = `move`;

    this._element = element;

    this._element.addEventListener('mousedown', this.mouseDown);

    const label = document.createElement("p");
    label.innerText = this._label;
    this._element.appendChild(label);

    canvaAreaRef.value.appendChild(this._element);
    this.applyElementPosition();
    stateByElement.set(this._element, this);
  }

  destroy(): void {
    [...this._outTransitions].forEach(t => t.destroy());
    [...this._inTransitions].forEach(t => t.destroy());

    if (this._element) {
      this._element.removeEventListener('mousedown', this.mouseDown);
      if (stateByElement.has(this._element)) stateByElement.delete(this._element);
      this._element.remove();
      this._element = null;
    }

    if (canvaAreaRef.value) {
      canvaAreaRef.value.removeEventListener('mousemove', this.mouseMove);
      canvaAreaRef.value.removeEventListener('mouseup', this.mouseUp);
    }

    const index = states.value.indexOf(this);
    if (index > -1) states.value.splice(index, 1);

    if (currentState.value === this) currentState.value = undefined;
    if (focusedState.value === this) focusedState.value = undefined;
    if (inicialState.value === this) inicialState.value = undefined;
    const fiIdx = finalStates.value.indexOf(this);
    if (fiIdx > -1) finalStates.value.splice(fiIdx, 1);
  }
}

function updateCtxMenuPos(pos: Coord): void {
  if (!contextMenuRef.value || !canvaAreaRef.value) return;

  const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();
  const maxTopValue = canvaAreaRef.value.clientHeight - contextMenuRef.value.offsetHeight + canvaY;
  const maxLeftValue = canvaAreaRef.value.clientWidth - contextMenuRef.value.offsetWidth + canvaX;
  const [x, y] = pos;

  contextMenuRef.value.style.left = `${Math.min(maxLeftValue, x)}px`;
  contextMenuRef.value.style.top = `${Math.min(maxTopValue, y)}px`;
}

function moveTape() {
  if (!tapeRef.value || !tapeRef.value.children.length) return;
    
  const tapeSlotsLength = Array.from(tapeRef.value.children).map((slot) => slot.getBoundingClientRect().width);
  tapeRef.value.style.transform = `translateX(${
    (tapeRef.value.getBoundingClientRect().width / 2) - tapeSlotsLength.slice(0, head.value).reduce((sum, value) => sum + value, 0) - (tapeSlotsLength.at(head.value) as number / 2)
  }px`;
}

function iterate() {
  try {
    if (currentState.value && finalStates.value.includes(currentState.value)) {
      alert(`Palavra aceita! Estado final: ${currentState.value.label}`);
      return;
    }
    currentState.value = currentState.value?.executeTransition();
    moveTape();
    if (currentState.value && finalStates.value.includes(currentState.value)) {
      moveTape();
      alert(`Palavra aceita! Estado final: ${currentState.value.label}`);
    }
  } catch (error) {
    if (error instanceof Error && error.message === "No applicable transition found") {
      alert(`Palavra rejeitada! Nenhuma transição aplicável em "${currentState.value?.label}".`);
    } else {
      console.error(error);
      alert(`Erro inesperado: ${error instanceof Error ? error.message : String(error)}`);
    }
  }
}

function createNewState(e: PointerEvent): State {
  if (!canvaAreaRef.value) throw new Error("No defined Canvas");

  const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();
  const state = new State(undefined, [e.clientX - canvaX, e.clientY - canvaY]);
  state.renderElement();
  return state;
}

function createNewTransition(): Transition{
  if (!focusedState.value || !canvaAreaRef.value) throw new Error("No focused State");
  const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();

  const mouseMove = (e: MouseEvent) => {
    tempState.pos = [e.clientX - canvaX, e.clientY - canvaY];
  }

  const mouseDown = (e: MouseEvent) => {
    canvaAreaRef.value!.removeEventListener('mousemove', mouseMove);
    canvaAreaRef.value!.removeEventListener('mousedown', mouseDown);

    const target = e.target as HTMLElement;
    const element = target.closest('div') as HTMLDivElement | null;
    const clickedState = element ? stateByElement.get(element) : undefined;
    console.log(clickedState);

    if (clickedState && clickedState !== tempState) {
      transition.retarget(clickedState);
    } else {
      transition.destroy();
    }

    const [read, write, direction] = prompt("Função de transição", "a a R")!.split(" ");
    transition.read = read ?? "a";
    transition.write = write ?? "a";
    transition.direction = direction as "L" | "R" ?? "R";

    tempState.destroy();
  }

  canvaAreaRef.value.addEventListener('mousemove', mouseMove);
  canvaAreaRef.value.addEventListener('mousedown', mouseDown);

  const tempState = new State();
  tempState._radius = 1;
  const transition = focusedState.value.addTransition("", "", "R", tempState);

  return transition;
}

function editTransition(): Transition {
  if (!focusedTransition.value || !canvaAreaRef.value) throw new Error("No focused transition or defined canvas");

  const r = focusedTransition.value.read;
  const w = focusedTransition.value.write;
  const d = focusedTransition.value.direction;

  const [read, write, direction] = prompt("Nova Transição", `${r} ${w} ${d}`)?.split(" ") ?? [r, w, d];

  focusedTransition.value.read = read ?? "a";
  focusedTransition.value.write = write ?? "a";
  focusedTransition.value.direction = direction as "L" | "R" ?? "R";

  return focusedTransition.value;
}

function deleteTransition(): void {
  if (!focusedTransition.value || !canvaAreaRef.value) throw new Error("No focused transition or defined canvas");

  focusedTransition.value.destroy();
}

function setInicialState(): State {
  if (!focusedState.value) throw new Error("No focused State");

  inicialState.value = focusedState.value;
  currentState.value = inicialState.value;
  return inicialState.value;
}

function toggleFinalState(): void {
  if (!focusedState.value) throw new Error("No focused State");

  if (finalStates.value.includes(focusedState.value)) {
    const index = finalStates.value.indexOf(focusedState.value);
    finalStates.value.splice(index, 1);
    focusedState.value._element?.classList.remove("final-state");
  }
  else {
    finalStates.value.push(focusedState.value);
    focusedState.value._element?.classList.add("final-state");
  }
}

onMounted(() => {
  // Lógica do Custom Context Menu
  {
    // click com o botão esquerdo dentro do canvas
    canvaAreaRef.value!.addEventListener("contextmenu", (e) => {
      e.preventDefault();
      updateCtxMenuPos([e.clientX, e.clientY]);
      if (!focusedState.value?._element?.contains(e.target as Node)) focusedState.value = undefined;
      if (!transitionByElement.has((e.target as Element).closest('g') as Element)) focusedTransition.value = undefined;
      contextMenuRef.value!.style.visibility = "visible";
    });

    // Qualquer click fora do Canvas
    document.addEventListener("click", () => {
      contextMenuRef.value!.style.visibility = "hidden";
      focusedState.value = undefined;
      focusedTransition.value = undefined;
    })
  }

  moveTape();

  const [startSymbol, markSymbol, blankSymbol] = auxiliaryAlphabet.value;

  const q0 = new State("q0", [200, 300]);
  const q1 = new State("q1", [400, 200]);
  const q2 = new State("q2", [600, 200]);
  const q3 = new State("q3", [800, 300]);
  const q4 = new State("q4", [600, 400]);
  const q5 = new State("q5", [400, 400]);
  const q6 = new State("q6", [200, 500]);

  q0.addTransition("a", markSymbol, "R", q1);
  q0.addTransition("b", markSymbol, "R", q4);
  q0.addTransition(markSymbol, markSymbol, "R", q0);
  q0.addTransition(blankSymbol, blankSymbol, "R", q6);

  q1.addTransition("a", "a", "R", q1);
  q1.addTransition("b", markSymbol, "R", q2);

  q2.addTransition("b", "b", "R", q2);
  q2.addTransition("a", "a", "R", q2);
  q2.addTransition(blankSymbol, blankSymbol, "L", q3);

  q3.addTransition("a", "a", "L", q3);
  q3.addTransition("b", "b", "L", q3);
  q3.addTransition(markSymbol, markSymbol, "L", q3);
  q3.addTransition(startSymbol, startSymbol, "R", q0);

  q4.addTransition("b", "b", "R", q4);
  q4.addTransition("a", markSymbol, "R", q5);

  q5.addTransition("a", "a", "R", q5);
  q5.addTransition("b", "b", "R", q5);
  q5.addTransition(blankSymbol, blankSymbol, "L", q3);

  currentState.value = q0;
  inicialState.value = q0;
  finalStates.value = [q6];

  q0.renderElement();
  q1.renderElement();
  q2.renderElement();
  q3.renderElement();
  q4.renderElement();
  q5.renderElement();
  q6.renderElement();
  q6._element?.classList.add("final-state");

  // Quando o input da palavra muda, essa função é chamada
  watch(word, (newWord) => {
    tape.value = auxiliaryAlphabet.value[0] + newWord + auxiliaryAlphabet.value[2];
    head.value = 1;
    currentState.value = inicialState.value;
  });
})
</script>

<template>
  <main>
    <!-- Sidebar -->
    <aside id="machine-config-area">
      <div class="brand">
        <h1>Máquina de Turing</h1>
        <span class="status-badge">Estado atual: <strong>{{ currentState?.label ?? '—' }}</strong></span>
      </div>

      <div class="controls">
        <div class="input-group">
          <label for="i-word">Palavra</label>
          <input id="i-word" v-model="word" placeholder="Ex: abbaba" />
        </div>
        <button class="btn-primary" @click="iterate">Próximo passo</button>
      </div>

      <section class="info-section">
        <h2>Definição formal</h2>
        <div class="meta-grid">
          <div class="meta-item"><span class="meta-label">Alfabeto</span><span>{ {{ [...new Set(word)].join(", ") }} }</span></div>
          <div class="meta-item"><span class="meta-label">Auxiliar</span><span>{ {{ auxiliaryAlphabet.join(", ") }} }</span></div>
          <div class="meta-item"><span class="meta-label">Inicial</span><span>{{ inicialState?.label ?? '—' }}</span></div>
          <div class="meta-item"><span class="meta-label">Finais</span><span class="badge-final">{{ finalStates.map(s => s.label).join(", ") || '—' }}</span></div>
        </div>
      </section>

      <section class="info-section transitions-section">
        <h2>Estados e transições</h2>
        <ul id="states-list">
          <li v-for="(state, index) of states" :key="index" :class="{ active: currentState === state }">
            <div class="state-header">
              <span class="state-node-name">{{ state.label }}</span>
              <div class="state-badges">
                <span v-if="inicialState === state" class="badge-inicial">Inicial</span>
                <span v-if="finalStates.includes(state)" class="badge-final">Final</span>
              </div>
            </div>
            <ul class="transition-sublist">
              <li v-for="(transition, tIdx) of state.transitions" :key="tIdx">
                <span class="t-trigger">{{ transition.read }}</span>
                <span class="t-arrow"> → </span>
                <span class="t-action">{{ transition.write }}, {{ transition.direction }}</span>
                <span class="t-arrow"> → </span>
                <span class="t-target">{{ transition.toState.label }}</span>
              </li>
              <li v-if="state.transitions.length === 0" class="empty-transitions">Nenhuma transição</li>
            </ul>
          </li>
        </ul>
      </section>
    </aside>

    <!-- Canvas -->
    <section ref="canvaAreaRef" id="canva-area">
      <svg id="arrows-layer" :width="canvaAreaRef?.clientWidth" :height="canvaAreaRef?.clientHeight">
        <defs>
          <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth">
            <path d="M0, 0 L0, 6 L9, 3 z" fill="#4f46e5" />
          </marker>
        </defs>
        <g v-for="(transition, index) in transitions" :key="transition.elementId" :id="transition.elementId"
          @contextmenu="(e) => { focusedTransition = transitions[index] as Transition; }"
          :ref="el => {
            if (el) transitionByElement.set(el as Element, transition as Transition)
            else {
              for (const [key, value] of transitionByElement.entries()) {
                if (value === transition) { transitionByElement.delete(key); break; }
              }
            }
          }">
          <path :d="transition.pathD" fill="none" stroke="#4f46e5" stroke-width="2" marker-end="url(#arrowhead)" />
          <text :x="transition.labelOffSet[0] - 10" :y="transition.labelOffSet[1] + 22" class="transition-label">
            {{ transition.label }}
          </text>
        </g>
      </svg>

      <ul ref="contextMenuRef" id="context-menu">
        <li v-if="focusedState" @click="setInicialState">Definir como inicial</li>
        <li v-if="focusedState" @click="toggleFinalState" class="ctx-checkbox">
          <input type="checkbox" id="final-state-cb" :checked="finalStates.includes(focusedState)" />
          <label for="final-state-cb">Estado final</label>
        </li>
        <li v-if="focusedState" @click="createNewTransition">Nova transição</li>
        <li v-if="focusedTransition" @click="editTransition">Editar transição</li>
        <li v-if="!focusedState && !focusedTransition" @click="createNewState">Novo estado</li>
        <li v-if="focusedState" @click="focusedState.destroy()" class="ctx-danger">Deletar estado</li>
        <li v-if="focusedTransition" @click="deleteTransition" class="ctx-danger">Excluir transição</li>
      </ul>
    </section>

    <!-- Fita -->
    <section id="machine-area">
      <div class="head-indicator">
        <svg viewBox="0 0 100 100" preserveAspectRatio="none">
          <polygon points="0,0 100,0 50,100" />
        </svg>
        <span>Cabeçote</span>
      </div>
      <div class="tape-window">
        <div class="tape" ref="tapeRef">
          <div v-for="(char, index) in tape" :key="index" class="tape-cell" :class="{ head: index === head }">
            {{ char }}
          </div>
        </div>
      </div>
    </section>
  </main>
</template>

<style scoped>
/* ── Variáveis ── */
main {
  --bg: #f8fafc;
  --surface: #ffffff;
  --border: #e2e8f0;
  --text: #1e293b;
  --muted: #64748b;
  --primary: #4f46e5;
  --primary-hover: #4338ca;
  --danger: #ef4444;
  --accent-active: #ffe082;
  --tape-bg: #1e293b;
  --tape-border: #334155;

  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: var(--text);
  background: var(--bg);
  min-height: 100vh;
  max-height: 100vh;
  overflow: hidden;
  display: grid;
  grid-template-columns: 300px 1fr;
  grid-template-rows: 1fr auto;
  grid-template-areas:
    "sidebar canvas"
    "sidebar tape";
}

/* ── Sidebar ── */
#machine-config-area {
  grid-area: sidebar;
  background: var(--surface);
  border-right: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  padding: 1.25rem;
  overflow-y: auto;
}

.brand h1 {
  font-size: 1.15rem;
  font-weight: 700;
  margin: 0 0 0.2rem 0;
  color: #0f172a;
}

.status-badge {
  font-size: 0.82rem;
  color: var(--muted);
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.input-group label {
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--muted);
}

.input-group input {
  padding: 0.45rem 0.7rem;
  border: 1px solid var(--border);
  border-radius: 6px;
  font-family: monospace;
  font-size: 0.95rem;
  outline: none;
  background: var(--bg);
  color: var(--text);
}

.input-group input:focus {
  border-color: var(--primary);
  box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.15);
}

.btn-primary {
  background: var(--primary);
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover { background: var(--primary-hover); }

/* ── Seções da sidebar ── */
.info-section h2 {
  font-size: 0.72rem;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--muted);
  margin: 0 0 0.6rem 0;
  padding-bottom: 0.25rem;
  border-bottom: 1px solid var(--border);
}

.meta-grid {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.meta-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
  background: var(--bg);
  padding: 0.3rem 0.55rem;
  border-radius: 4px;
}

.meta-label { color: var(--muted); font-weight: 500; }

/* ── Lista de estados ── */
.transitions-section {
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
  gap: 0.6rem;
  overflow-y: auto;
  flex: 1;
}

#states-list > li {
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 0.5rem 0.6rem;
  background: #fdfdfd;
  transition: border-color 0.15s, background 0.15s;
}

#states-list > li.active {
  border-color: #f59e0b;
  background: #fffbeb;
}

.state-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.3rem;
}

.state-node-name {
  font-size: 0.85rem;
  font-weight: 600;
  background: #e2e8f0;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
}

.state-badges { display: flex; gap: 0.25rem; }

.badge-final {
  background: #fee2e2;
  color: var(--danger);
  font-size: 0.72rem;
  font-weight: 700;
  padding: 0.1rem 0.35rem;
  border-radius: 4px;
}

.badge-inicial {
  background: #dcfce7;
  color: #16a34a;
  font-size: 0.72rem;
  font-weight: 700;
  padding: 0.1rem 0.35rem;
  border-radius: 4px;
}

.transition-sublist {
  list-style: none;
  padding: 0 0 0 0.4rem;
  margin: 0;
  font-family: monospace;
  font-size: 0.8rem;
  color: #475569;
}

.transition-sublist li { padding: 0.1rem 0; }

.t-trigger { color: #b45309; font-weight: 700; }
.t-action  { color: #1e1b4b; }
.t-target  { color: var(--primary); font-weight: 700; }
.t-arrow   { color: var(--muted); }
.empty-transitions { color: #94a3b8; font-style: italic; }

/* ── Canvas ── */
#canva-area {
  grid-area: canvas;
  position: relative;
  min-height: 0;
  min-width: 0;
  overflow: hidden;
  background: var(--surface);
  border-left: 1px solid var(--border);
}

#arrows-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;

  & g { pointer-events: all; }
}

.transition-label {
  font-family: monospace;
  font-size: 12px;
  fill: #334155;
}

/* Estados (divs criados por renderElement) */
:deep(.state-div) {
  position: fixed;
  aspect-ratio: 1/1;
  width: 80px;
  border: 2px solid #334155;
  border-radius: 50%;
  display: grid;
  place-items: center;
  cursor: move;
  background: var(--surface);
  font-family: system-ui, sans-serif;
  font-weight: 600;
  font-size: 0.9rem;
  color: var(--text);
  user-select: none;
  transition: background 0.15s, border-color 0.15s, box-shadow 0.15s;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
  z-index: 1;
}

:deep(.state-div:hover) {
  box-shadow: 0 2px 8px rgba(79, 70, 229, 0.18);
  border-color: var(--primary);
}

:deep(.current-state) {
  background: var(--accent-active) !important;
  border-color: #d97706 !important;
}

:deep(.inicial-state::before) {
  content: '';
  position: absolute;
  inset: -6px;
  border-radius: 50%;
  border: 2px dashed #16a34a;
  pointer-events: none;
}

:deep(.final-state) {
  border-width: 4px;
  border-color: var(--danger);
}

/* ── Menu de contexto ── */
#context-menu {
  list-style: none;
  padding: 0.25rem 0;
  margin: 0;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 7px;
  box-shadow: 0 4px 16px rgba(0,0,0,0.10);
  position: fixed;
  min-width: 170px;
  visibility: hidden;
  z-index: 100;
}

#context-menu li {
  padding: 0.45rem 1rem;
  font-size: 0.875rem;
  cursor: pointer;
  color: var(--text);
  transition: background 0.12s;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

#context-menu li:hover { background: #f1f5f9; }

#context-menu .ctx-danger { color: var(--danger); }
#context-menu .ctx-danger:hover { background: #fef2f2; }

#context-menu .ctx-checkbox input { cursor: pointer; }
#context-menu .ctx-checkbox label { cursor: pointer; }

/* ── Fita ── */
#machine-area {
  grid-area: tape;
  background: var(--bg);
  border-top: 1px solid var(--tape-border);
  padding: 1rem 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4rem;
}

.head-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: var(--danger);
  font-size: 0.7rem;
  font-weight: 700;
}

.head-indicator svg {
  width: 14px;
  height: 10px;
  fill: var(--danger);
}

.tape-window {
  width: 100%;
  max-width: 820px;
  overflow: hidden;
  border: 2px solid var(--tape-border);
  background: var(--tape-bg);
  border-radius: 8px;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.25);
}

.tape {
  display: flex;
  width: max-content;
  transition: transform 0.15s cubic-bezier(0.4, 0, 0.2, 1);
}

.tape-cell {
  width: 56px;
  min-width: 56px;
  height: 56px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: monospace;
  font-size: 1.8rem;
  font-weight: 700;
  color: #f8fafc;
  border-right: 1px solid var(--tape-border);
  transition: background 0.15s, color 0.15s;
}

.tape-cell.head {
  background: #eab308;
  color: #0f172a;
}
</style>
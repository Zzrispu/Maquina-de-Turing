<script setup lang="ts">
import { ref, onMounted, watch, computed, reactive } from "vue";

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

const currentState = ref<State>();
const focusedState = ref<State>();
const finalStates = ref<State[]>([]);
const auxiliaryAlphabet = ref<AuxiliaryAlphabet>(["*", "X", "B"]);

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
    return [this.geometry.labelPos[0], this.geometry.labelPos[1] + this.stackCount * 12];
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
    element.style.position = "fixed";
    element.style.aspectRatio = "1/1";
    element.style.width = `${this._radius * 2}px`;
    element.style.border = `2px solid #000000`;
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
    currentState.value = currentState.value?.executeTransition();
    moveTape();
  } catch (error) {
    console.error(error);
  }
}

function createNewState(e: PointerEvent): State {
  if (!canvaAreaRef.value) throw new Error("No defined Canvas");

  const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();
  const state = new State(undefined, [e.clientX - canvaX, e.clientY - canvaY]);
  state.renderElement();
  return state;
}

function createNewTransition(e: PointerEvent): Transition{
  if (!focusedState.value || !canvaAreaRef.value) throw new Error("No focused State");
  const {x: canvaX, y: canvaY} = canvaAreaRef.value.getBoundingClientRect();

  const statesElements = canvaAreaRef.value.querySelectorAll("div");

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

onMounted(() => {
  // Lógica do Custom Context Menu
  {
    canvaAreaRef.value!.addEventListener("contextmenu", (e) => {
      e.preventDefault();
      updateCtxMenuPos([e.clientX, e.clientY]);
      contextMenuRef.value!.style.visibility = "visible";
    });

    document.addEventListener("click", () => {
      contextMenuRef.value!.style.visibility = "hidden";
      focusedState.value = undefined;
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
  finalStates.value = [q6];

  q0.renderElement();
  q1.renderElement();
  q2.renderElement();
  q3.renderElement();
  q4.renderElement();
  q5.renderElement();
  q6.renderElement();

  // Quando o input da palavra muda, essa função é chamada
  watch(word, (newWord) => {
    tape.value = auxiliaryAlphabet.value[0] + newWord + auxiliaryAlphabet.value[2];
    head.value = 1;
    currentState.value = q0;
  });
})
</script>

<template>
  <main>
    <section id="machine-config-area" style="grid-area: machine-config-area; border: 1px solid blue;">
      <h1>Máquina de Turing</h1>
      <label for="i-word">Palavra:</label>
      <input id="i-word" v-model="word" />
      <button @click="iterate">▶</button>
      <p>Alfabeto: { {{ [...new Set(word)].join(", ") }} }</p>
      <p>Alfabeto Auxiliar: { {{ auxiliaryAlphabet.join(", ") }} }</p>
      <p>Estados Finais: {{ finalStates.map(s => s.label).join(", ") }}</p>
      <p>Estado Atual: {{ currentState?.label }}</p>
      <ul id="states-list">
        <li v-for="(state, index) of states" :key="index">
          <p>{{ state.label }}</p>
          <ul>
            <li v-for="(transition, index) of state.transitions" :key="index">
              ({{ transition.label }}), {{ transition.toState.label }}
            </li>
          </ul>
        </li> 
      </ul>
    </section>
    <section ref="canvaAreaRef" id="canva-area" style="grid-area: canva-area; border: 1px solid black;">
      <svg ref="arrowLayerRef" id="arrows-layer" :width="canvaAreaRef?.clientWidth" :height="canvaAreaRef?.clientHeight">
        <defs>
          <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto" markerUnits="strokeWidth">
            <path d="M0, 0 L0, 6 L9, 3 z" fill="black" />
          </marker>
        </defs>
        <g v-for="transition in transitions" :key="transition.label + transition.fromState.label + transition.toState.label">
          <path :d="transition.pathD" fill="none" stroke="black" stroke-width="2" marker-end="url(#arrowhead)" />
          <text :x="transition.labelOffSet[0]" :y="transition.labelOffSet[1]">
            {{ transition.label }}
          </text>
        </g>
      </svg>
      <ul ref="contextMenuRef" id="context-menu">
        <li @click="createNewState">Novo Estado</li>
        <li v-if="focusedState" @click="createNewTransition">Nova Transição</li>
        <li v-if="focusedState" @click="focusedState.destroy()">Deletar Estado</li>
      </ul>
    </section>
    <section id="machine-area" style="grid-area: machine-area; border: 1px solid yellow;">
      <svg width="24px" height="24px" viewBox="0 0 100 100" preserveAspectRatio="none">
        <polygon points="0,0 100,0 50,100" style="stroke:red; fill: transparent;" />
      </svg>
      <div class="tape" ref="tapeRef">
        <div v-for="(char, index) in tape" :key="index" :class="{ head: index === head }">
          {{ char }}
        </div>
      </div>
    </section>
  </main>
</template>

<style scoped>
main {
  min-height: 100vh;
  max-height: 100vh;
  overflow: hidden;
  display: grid;
  grid-template-columns: auto 1fr;
  grid-template-rows: 1fr auto;
  grid-template-areas:
    "machine-config-area canva-area"
    "machine-config-area machine-area";
}

#machine-config-area, #machine-area {
  padding: 1rem;
}

#canva-area {
  min-height: 0;
  min-width: 0;
  overflow: hidden;
  display: grid;

  &>#context-menu {
    border: 1px solid snow;
    background-color: #ffffff;
    width: fit-content;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    position: fixed;
    border-radius: 5px;
    overflow: hidden;
    visibility: hidden;
    z-index: 1;

    &>li {
      padding: .5rem 1rem;

      &:hover{
        cursor: pointer;
        background-color: #f7f7f7;
      }
    }
  }

  &>#arrow-layer {
    width: 100%;
    height: 100%;
    display: block;
  }
}

#machine-area {
  display: grid;
  justify-items: center;
  align-items: center;

  &>svg {
    width: 3rem;
    height: 4rem;
  }

  .tape {
    display: flex;
    border: 1px solid lime;
    font-size: 4rem;
    margin-bottom: 2rem;
    transition: transform 0.1s ease;
  }

  .head {
    background-color: yellow;
  }
}

#states-list {
  border: 1px solid purple;
  padding: 1rem;

  &>li>ul {
    margin-left: .5rem;
  }
}
</style>

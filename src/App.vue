<script setup lang="ts">
import { ref, onMounted, watch, computed } from "vue";

type Transition = [read: string, write: string, direction: "L" | "R", nextStage: State];
type AuxiliaryAlphabet = [startSymbol: string, markSymbol: string, blankSymbol: string];
type Coord = [X: number, Y: number];

const word = ref<string>("abbaba");
const head = ref<number>(1);

const states = ref<State[]>([]);
const currentState = ref<State>();
const finalStates = ref<State[]>([]);
const auxiliaryAlphabet = ref<AuxiliaryAlphabet>(["*", "X", "B"]);

const canvasRef = ref<HTMLCanvasElement | null>(null);
const ctx = computed(() => canvasRef.value?.getContext("2d") as CanvasRenderingContext2D);
const tapeRef = ref<HTMLDivElement | null>(null);

const tape = ref<string>(auxiliaryAlphabet.value[0] + word.value + auxiliaryAlphabet.value[2]);

class State {
  private _label: string;
  private _transitions: Transition[];

  private _pos: Coord;
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
  }

  get label(): string {
    return this._label;
  }

  set label(label: string) {
    this._label = label;
  }

  get transitions(): Transition[] {
    return this._transitions;
  }

  addTransition(transition: Transition) {
    this._transitions.push(transition);
  }

  removeTransition(transition: Transition | number) {
    if (typeof transition === "number") this.transitions.splice(transition, 1);
    else {
      const index = this._transitions.indexOf(transition);
      if (index > -1) this._transitions.splice(index, 1);
      else throw new Error("Transition not found");
    };
  }

  executeTransition(): State {
    for (const transition of this._transitions) {
      const [read, write, direction, nextState] = transition;
      if (tape.value[head.value] === read) {
        tape.value = tape.value.substring(0, head.value) + write + tape.value.substring(head.value + 1);
        head.value += direction === "L" ? -1: 1;
        if (head.value < 0) head.value = 0;
        if (head.value >= tape.value.length) head.value = tape.value.length - 1;
        return nextState;
      }
    }
    throw new Error("No applicable transition found");
  }

  drawState(): void {
    const [x, y] = this._pos;

    ctx.value.beginPath();
    ctx.value.arc(x, y, this._radius, 0, 2 * Math.PI);
    ctx.value.stroke();
    ctx.value.font = "18px Arial";
    ctx.value.fillText(this._label, x, y);

    let i: number = 0;
    for (const joint of this._joints) {
      ctx.value.font = "10px Arial";
      ctx.value.fillText(`${i}`, joint[0], joint[1]);
      i++;
    }
  }

  drawTransitions(): void {
    const [x, y] = this._pos;
    const transitionCounts = new Map<State, [number, Coord]>();

    for (const transition of this._transitions) {
      const [read, write, direction, nextState] = transition;
      const [nextX, nextY] = nextState._pos;

      if (transitionCounts.has(nextState)) {
        const coord = transitionCounts.get(nextState)?.[1] as Coord;
        const count = transitionCounts.get(nextState)?.[0] as number;
        ctx.value.font = "12px Arial";
        ctx.value.fillText(`${read} → ${write}, ${direction}`, coord[0], coord[1] - (12 * count));
        transitionCounts.set(nextState, [count + 1, coord]);
        continue;
      }

      let closestJoints: [Coord, Coord] = [this._pos, nextState._pos];
      let closestDistance: number = Number.POSITIVE_INFINITY;

      let startI: number;
      let endI: number;
      let startJ: number;
      let endJ: number;

      // Proximo estado está no primeiro quadrante em relação ao estado atual como origem
      if ((x < nextX) && (y >= nextY)) {
        startI = 6;
        endI = 9;
        startJ = 2;
        endJ = 5;
      }
      // Segundo quadrante
      else if ((x > nextX) && (y >= nextY)) {
        startI = 4;
        endI = 7;
        startJ = 0;
        endJ = 3;
      }
      // Terceiro quadrante
      else if ((x > nextX) && (y < nextY)) {
        startI = 2;
        endI = 5;
        startJ = 6;
        endJ = 9;
      }
      // Quarto quadrante
      else {
        startI = 0;
        endI = 3;
        startJ = 4;
        endJ = 7;
      }

      for (let i = startI; i < endI; i++) {
        console.log("i")
        const joint = i < this._joints.length ? this._joints[i] as Coord : this._joints[i - this._joints.length] as Coord;
        for (let j = startJ; j < endJ; j++) {
          console.log("j")
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
      ctx.value.stroke();

      const dx = closestJoints[1][0] - cpX;
      const dy = closestJoints[1][1] - cpY;
      const angle = Math.atan2(dy, dx);

      const arrowLength = 15;
      ctx.value.beginPath();
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

      ctx.value.font = "12px Arial";
      ctx.value.fillText(`${read} → ${write}, ${direction}`, cpX, cpY + 35);

      transitionCounts.set(nextState, [1, [cpX, cpY + 35]]);
    }
  }
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

onMounted(() => {
  // A função anonima abeixo executa uma vez e não gera reziduos
  (() => {
    const { width: canvaWidth, height: canvaHeight } = (canvasRef.value?.parentElement as HTMLElement).getBoundingClientRect();
    canvasRef.value!.width = canvaWidth;
    canvasRef.value!.height = canvaHeight;

    moveTape();
  })();

  const [startSymbol, markSymbol, blankSymbol] = auxiliaryAlphabet.value;

  const q0 = new State("q0", [200, 300]);
  const q1 = new State("q1", [400, 200]);
  const q2 = new State("q2", [600, 200]);
  const q3 = new State("q3", [800, 300]);
  const q4 = new State("q4", [600, 400]);
  const q5 = new State("q5", [400, 400]);
  const q6 = new State("q6", [200, 500]);

  states.value = [q0, q1, q2, q3, q4, q5, q6];

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

  q0.drawState();
  q1.drawState();
  q2.drawState();
  q3.drawState();
  q4.drawState();
  q5.drawState();
  q6.drawState();

  q2.drawTransitions();
  q3.drawTransitions();

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
              ({{ transition[0] }} → {{ transition[1] }}, {{ transition[2] }}), {{ transition[3].label }}
            </li>
          </ul>
        </li> 
      </ul>
    </section>
    <section id="canva-area" style="grid-area: canva-area; border: 1px solid black;">
      <canvas id="canva" ref="canvasRef"></canvas>
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

  &>#canva {
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

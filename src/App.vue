<script setup lang="ts">
import { ref, onMounted, watch } from "vue";

type Transition = [string, string, "L" | "R", State];
type AuxiliaryAlphabet = [string, string, string];

const word = ref<string>("abbaba");
const head = ref<number>(1);

const currentState = ref<State>();
const finalStates = ref<State[]>([]);
const auxiliaryAlphabet = ref<AuxiliaryAlphabet>(["*", "X", "B"]);

const tape = ref<string>(auxiliaryAlphabet.value[0] + word.value + auxiliaryAlphabet.value[2]);

class State {
  private _label: string;
  private transitions: Transition[];

  private counter: number = 0;

  constructor(label?: string) {
    if (label) this._label = label;
    else this._label = `q${this.counter++}`;

    this.transitions = [];
  }

  get label(): string {
    return this._label;
  }

  set label(label: string) {
    this._label = label;
  }

  addTransition(transition: Transition) {
    this.transitions.push(transition);
  }

  removeTransition(transition: Transition | number) {
    if (typeof transition === "number") this.transitions.splice(transition, 1);
    else {
      const index = this.transitions.indexOf(transition);
      if (index > -1) this.transitions.splice(index, 1);
      else throw new Error("Transition not found");
    };
  }

  executeTransition(): State {
    for (const transition of this.transitions) {
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
}

const canvasRef = ref<HTMLCanvasElement | null>(null);
const tapeRef = ref<HTMLDivElement | null>(null);

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

  const q0 = new State();
  const q1 = new State("q1");
  const q2 = new State("q2");
  const q3 = new State("q3");
  const q4 = new State("q4");
  const q5 = new State("q5");
  const q6 = new State("q6");

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
</style>

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
watch(word, (newWord) => {
  tape.value = auxiliaryAlphabet.value[0] + newWord + auxiliaryAlphabet.value[2];
  head.value = 1;
});

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

function iterate() {
  try {
    currentState.value = currentState.value?.executeTransition();
  } catch (error) {
    console.error(error);
  }
}

onMounted(() => {
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
})
</script>

<template>
  <h1>Máquina de Turing</h1>
  <label for="i-word">Palavra:</label>
  <input id="i-word" v-model="word" />
  <button @click="iterate">▶</button>
  <p>Alfabeto: { {{ [...new Set(word)].join(", ") }} }</p>
  <p>Alfabeto Auxiliar: { {{ auxiliaryAlphabet.join(", ") }} }</p>
  <p>Estados Finais: {{ finalStates.map(s => s.label).join(", ") }}</p>
  <p>Estado Atual: {{ currentState?.label }}</p>
  <div class="tape">
   <div v-for="(char, index) in tape" :key="index" :class="{ head: index === head }">
      {{ char }}
    </div>
  </div>
</template>

<style scoped>
.tape {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
}

.head {
  background-color: yellow;
}
</style>

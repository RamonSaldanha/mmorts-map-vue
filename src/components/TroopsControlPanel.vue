<template>
  <div class="troop-control-panel">
    <div class="panel-header">
      <h3>Controle de Tropas</h3>
      <button @click="closePanel" class="close-button">X</button>
    </div>
    <div v-for="(soldado, index) in soldados" :key="index" class="troop-item">
      <img :src="soldado.img" :alt="soldado.tipo" class="soldier-image">
      <span>{{ soldado.tipo }}</span>
      <input v-model.number="soldado.qtd" type="number" min="0" :max="soldado.maxQtd">
    </div>
    <button @click="enviarTropas" :disabled="!podeEnviar">Enviar Tropas</button>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

const props = defineProps(['soldiers', 'selectedTile', 'villagePosition']);
const emit = defineEmits(['send-troops', 'close-panel']);

const soldados = ref([
  {
    tipo: "Orc guerreiro",
    velocidade: 1,
    qtd: 0,
    maxQtd: 10,
    img: "https://i.imgur.com/L5dOVGA.png",
  },
  {
    tipo: "Centauro gatuno",
    velocidade: 2,
    qtd: 0,
    maxQtd: 5,
    img: "https://i.imgur.com/E78Z3Yd.png",
  }
]);

function closePanel() {
  emit('close-panel');
}

const podeEnviar = computed(() => {
  return props.selectedTile && soldados.value.some(s => s.qtd > 0);
});

function enviarTropas() {
  const tropasEnviadas = soldados.value.filter(s => s.qtd > 0).map(s => ({
    ...s,
    qtdEnviada: s.qtd
  }));
  
  emit('sendTroops', {
    troops: tropasEnviadas,
    destination: props.selectedTile,
    origin: props.villagePosition
  });
  
  soldados.value.forEach(s => s.qtd = 0);
}
</script>

<style scoped>

h3 {
  margin: 0;
  padding: 0;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.close-button {
  background: none;
  border: none;
  font-size: 18px;
  padding: 0px;
  margin: 0px;
  color: #000;
  cursor: pointer;
}

.troop-control-panel {
  margin: 20px;
  width: 230px;
  padding: 20px;
  background-color: #f0f0f0;
  border-left: 1px solid #ccc;
  position: absolute;
  bottom:0px;
}

.soldier-control {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.soldier-image {
  width: 20px;
  height: 20px;
  margin-right: 10px;
}

input[type="number"] {
  width: 30px;
}

button {
  margin-top: 20px;
  padding: 10px;
  background-color: #4CAF50;
  color: white;
  border: none;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
</style>
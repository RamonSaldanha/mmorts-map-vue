<template>
  <div class="map-wrapper">
    <div class="map-container" ref="mapContainer">
      <div 
        class="map" 
        :style="{ 
          transform: `translate(${mapPosition.x}px, ${mapPosition.y}px) scale(${zoomLevel})`
        }"
        @mousedown="startDragging"
        @mousemove="drag"
        @mouseup="stopDragging"
        @mouseleave="stopDragging"
        @click="handleMapClick"
      >
      <div 
        v-for="(tile, index) in tiles" 
        :key="index" 
        class="tile"
        :class="{ 'special-element': tile.url }"
        :style="getTileStyle(tile)"
      >
        <img v-if="tile.url" :src="tile.url" :alt="`Special element at (${tile.x}, ${tile.y})`" class="special-element-image">
        <div v-if="tile.x === villagePosition.x && tile.y === villagePosition.y" class="village">
          <img src="https://i.imgur.com/12Mm80B.png" alt="City Tile" class="special-element-image">
          <!-- https://i.imgur.com/dO1WS1V.png -->
          <!-- https://i.imgur.com/12Mm80B.png -->
        </div>
        <div class="coordinate-label">{{tile.y}},{{tile.x}}</div>
      </div>
        
        <!-- Tropas em movimento -->
        <div v-for="troop in movingTroops" :key="troop.id" class="troop"
             :style="getTroopStyle(troop)">
          <img :src="troop.img" :alt="troop.tipo" class="troop-icon">
        </div>
        

        <!-- Linhas de movimento -->
        <svg class="movement-layer" :viewBox="getLayerViewBox()">
          <line v-for="troop in movingTroops" :key="`line-${troop.id}`"
                :x1="getAdjustedX(troop.start.x, troop.start.y)"
                :y1="getAdjustedY(troop.start.x, troop.start.y)"
                :x2="getAdjustedX(troop.end.x, troop.end.y)"
                :y2="getAdjustedY(troop.end.x, troop.end.y)"
                stroke="#ff0000"
                stroke-width="2"
                stroke-dasharray="5,5" />
        </svg>

        <!-- Seta de seleção -->
        <svg class="arrow-layer" :viewBox="getLayerViewBox()">
          <defs>
            <marker id="arrowhead" markerWidth="10" markerHeight="7" 
              refX="10" refY="3.5" orient="auto">
              <polygon points="0 0, 10 3.5, 0 7" fill="#ff0000" />
            </marker>
          </defs>
          <line v-if="selectedTile"
            :x1="getAdjustedX(villagePosition.x, villagePosition.y)"
            :y1="getAdjustedY(villagePosition.x, villagePosition.y)"
            :x2="getAdjustedX(selectedTile.x, selectedTile.y)"
            :y2="getAdjustedY(selectedTile.x, selectedTile.y)"
            stroke="#ff0000"
            stroke-width="2"
            marker-end="url(#arrowhead)"
          />
        </svg>
      </div>
    </div>
    
    <div class="selected-tile" v-if="selectedTile">
      Selected Tile: { "y": {{ selectedTile.y }}, "x": {{ selectedTile.x }} }
      <span v-if="selectedTile.url">Special Element</span>
    </div>
    
    <TroopControlPanel 
      v-if="showTroopPanel"
      :soldiers="soldiers"
      :selectedTile="selectedTile"
      :villagePosition="villagePosition"
      @send-troops="sendTroops"
      @close-panel="closeTroopPanel"
    />
  </div>
</template>

<script setup>

import { ref, reactive, onMounted, onUnmounted } from 'vue';
//

import TroopControlPanel from './components/TroopsControlPanel.vue';

const mapSize = ref(30);
const tileSize = ref(50);
const TILE_WIDTH = tileSize.value * 2;
const TILE_HEIGHT = tileSize.value;

const mapPosition = reactive({ x: 0, y: 0 });
const isDragging = ref(false);
const lastMousePosition = reactive({ x: 0, y: 0 });
const startPosition = reactive({ x: 0, y: 0 });
const mapContainer = ref(null);

const zoomLevel = ref(1); // Nível de zoom inicial


const tiles = ref([]);
const selectedTile = ref(null);
const villagePosition = reactive({ x: 10, y: 10 });
const showTroopPanel = ref(false);

const soldiers = ref([
  {
    tipo: "Orc guerreiro",
    velocidade: 1,
    qtd: 0,
    img: "https://i.imgur.com/L5dOVGA.png",
  },
  {
    tipo: "Centauro gatuno",
    velocidade: 2,
    qtd: 0,
    img: "https://i.imgur.com/E78Z3Yd.png",
  }
]);

const specialElements = ref([
  {
    x: 0,
    y: 10,
    url: 'https://i.imgur.com/12Mm80B.png'
  },
  {
    x: 15,
    y: 2,
    url: 'https://i.imgur.com/dO1WS1V.png'
  }
]);


const movingTroops = ref([]);
let animationFrameId = null;

onMounted(() => {
  generateTiles();
  centerOnVillage();
  window.addEventListener('resize', centerOnVillage);
  if (mapContainer.value) {
    mapContainer.value.addEventListener('wheel', handleZoom); // Adiciona o evento de zoom
  }

  // Iniciar a animação
  animationFrameId = requestAnimationFrame(updateTroopPositions);
});

onUnmounted(() => {
  window.removeEventListener('resize', centerOnVillage);
  if (mapContainer.value) {
    mapContainer.value.removeEventListener('wheel', handleZoom); // Remove o evento de zoom
  }

  // Cancelar a animação
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});

function handleZoom(event) {
  event.preventDefault(); // Impede o comportamento de rolagem padrão

  const zoomFactor = 0.1; // Fator de zoom
  if (event.deltaY < 0) {
    // Rolagem para cima -> zoom in
    zoomLevel.value = Math.min(zoomLevel.value + zoomFactor, 3); // Limite de zoom máximo
  } else {
    // Rolagem para baixo -> zoom out
    zoomLevel.value = Math.max(zoomLevel.value - zoomFactor, 0.5); // Limite de zoom mínimo
  }
}

function closeTroopPanel() {
  showTroopPanel.value = false;
  selectedTile.value = null; // Opcional: desselecionar o tile quando o painel é fechado
}

function sendTroops(troopData) {
  console.log('Enviando tropas:', troopData);
  
  troopData.troops.forEach((troop, index) => {
    const id = Date.now() + index;
    const newTroop = {
      id,
      ...troop,
      position: { 
        x: troopData.origin.x * tileSize.value, 
        y: troopData.origin.y * tileSize.value 
      },
      start: { ...troopData.origin },
      end: { ...troopData.destination },
      isReturning: false,
      isMoving: true,
      startTime: Date.now(),
      duration: calculateTravelTime(troopData.origin, troopData.destination, troop.velocidade),
      waitTime: 5000 // 5 segundos de espera no destino
    };
    movingTroops.value.push(newTroop);
  });
}

function updateTroopPositions() {
  const now = Date.now();
  
  movingTroops.value.forEach((troop, index) => {
    if (!troop.isMoving) {
      if (now - troop.startTime > troop.waitTime) {
        // Iniciar viagem de retorno
        const temp = { ...troop.start };
        troop.start = { ...troop.end };
        troop.end = temp;
        troop.isReturning = true;
        troop.isMoving = true;
        troop.startTime = now;
        troop.duration = calculateTravelTime(troop.start, troop.end, troop.velocidade);
      }
      return;
    }

    const elapsed = now - troop.startTime;
    const progress = Math.min(elapsed / troop.duration, 1);

    troop.position.x = lerp(troop.start.x * tileSize.value, troop.end.x * tileSize.value, progress);
    troop.position.y = lerp(troop.start.y * tileSize.value, troop.end.y * tileSize.value, progress);

    if (progress === 1) {
      if (troop.isReturning) {
        // Remover tropa quando retornar à origem
        movingTroops.value.splice(index, 1);
      } else {
        // Parar no destino
        troop.isMoving = false;
        troop.startTime = now; // Reiniciar o temporizador para a espera
      }
    }
  });

  // Forçar a atualização do Vue
  if (movingTroops.value.length > 0) {
    movingTroops.value = [...movingTroops.value];
  }

  // Continuar a animação
  animationFrameId = requestAnimationFrame(updateTroopPositions);
}

function calculateTravelTime(start, end, speed) {
  const distance = Math.sqrt(
    Math.pow(end.x - start.x, 2) + 
    Math.pow(end.y - start.y, 2)
  );
  return (distance / speed) * 1000; // tempo em milissegundos
}

function lerp(start, end, t) {
  return start * (1 - t) + end * t;
}

function getLayerViewBox() {
  const width = mapSize.value * TILE_WIDTH * zoomLevel.value;
  const height = mapSize.value * TILE_HEIGHT * zoomLevel.value;
  return `0 0 ${width} ${height}`;
}

function getIsometricX(x, y) {
  return (x - y) * (TILE_WIDTH / 2);
}

function getIsometricY(x, y) {
  return (x + y) * (TILE_HEIGHT / 2);
}

function getAdjustedX(x, y) {
  // Ajusta o posicionamento considerando o zoom
  return (getIsometricX(x, y) + TILE_WIDTH / 2) * zoomLevel.value;
}

function getAdjustedY(x, y) {
  // Ajusta o posicionamento considerando o zoom
  return (getIsometricY(x, y) + TILE_HEIGHT / 2) * zoomLevel.value;
}


function getMapViewBox() {
  const width = mapSize.value * TILE_WIDTH;
  const height = mapSize.value * TILE_HEIGHT;
  return `${-width / 2} ${-height / 2} ${width * 2} ${height * 2}`;
}

function getTileStyle(tile) {
  const x = (tile.x - tile.y) * (TILE_WIDTH / 2);
  const y = (tile.x + tile.y) * (TILE_HEIGHT / 2);
  return {
    left: `${x}px`,
    top: `${y}px`,
    width: `${TILE_WIDTH}px`,
    height: `${TILE_HEIGHT}px`,
    zIndex: tile.y * mapSize.value + tile.x,
  };
}

function getIsometricPosition(x, y) {
  return {
    x: getIsometricX(x, y),
    y: getIsometricY(x, y)
  };
}

function getTroopStyle(troop) {
  const x = getIsometricX(troop.position.x / tileSize.value, troop.position.y / tileSize.value);
  const y = getIsometricY(troop.position.x / tileSize.value, troop.position.y / tileSize.value);
  return {
    left: `${x + TILE_WIDTH / 2}px`,
    top: `${y + TILE_HEIGHT / 2}px`,
    transform: 'translate(-50%, -50%)',
  };
}

function generateTiles() {
  for (let y = 0; y < mapSize.value; y++) {
    for (let x = 0; x < mapSize.value; x++) {
      tiles.value.push({ x, y });
    }
  }
  
  specialElements.value.forEach(element => {
    const tileIndex = tiles.value.findIndex(tile => tile.x === element.x && tile.y === element.y);
    if (tileIndex !== -1) {
      tiles.value[tileIndex] = { ...tiles.value[tileIndex], ...element };
    }
  });

  console.log('Generated tiles:', tiles.value);
}

function centerOnVillage() {
  if (!mapContainer.value) return;

  const containerWidth = mapContainer.value.clientWidth;
  const containerHeight = mapContainer.value.clientHeight;
  const villageX = getIsometricX(villagePosition.x, villagePosition.y);
  const villageY = getIsometricY(villagePosition.x, villagePosition.y);
  
  mapPosition.x = (containerWidth / 2) - villageX;
  mapPosition.y = (containerHeight / 2) - villageY;
}

function startDragging(event) {
  isDragging.value = true;
  lastMousePosition.x = event.clientX;
  lastMousePosition.y = event.clientY;
}

function drag(event) {
  if (!isDragging.value) return;

  const deltaX = event.clientX - lastMousePosition.x;
  const deltaY = event.clientY - lastMousePosition.y;

  mapPosition.x += deltaX;
  mapPosition.y += deltaY;

  lastMousePosition.x = event.clientX;
  lastMousePosition.y = event.clientY;
}

function stopDragging() {
  isDragging.value = false;
}

function handleMapClick(event) {
  if (isDragging.value) {
    isDragging.value = false;
    return;
  }

  const rect = mapContainer.value.getBoundingClientRect();
  const clickX = (event.clientX - rect.left - mapPosition.x) / zoomLevel.value;
  const clickY = (event.clientY - rect.top - mapPosition.y) / zoomLevel.value;

  // Ajuste o offset para centralizar o clique no tile
  const offsetX = TILE_WIDTH / 2;
  const offsetY = TILE_HEIGHT / 2;

  // Converter coordenadas do clique para coordenadas isométricas
  const isoX = ((clickX - offsetX) / (TILE_WIDTH / 2) + (clickY - offsetY) / (TILE_HEIGHT / 2)) / 2;
  const isoY = ((clickY - offsetY) / (TILE_HEIGHT / 2) - (clickX - offsetX) / (TILE_WIDTH / 2)) / 2;

  // Arredondar para o tile mais próximo e corrigir o deslocamento em X
  const tileX = Math.round(isoX);
  const tileY = Math.round(isoY);

  console.log('Clicked coordinates:', tileX, tileY);

  const clickedTile = tiles.value.find(tile => tile.x === tileX && tile.y === tileY);
  if (clickedTile) {
    selectedTile.value = clickedTile;
    showTroopPanel.value = true;
    console.log('Selected tile:', selectedTile.value);
  }
}



</script>

<style scoped>
.game-container {
  display: flex;
  width: 100%;
  height: 100vh;
}

.troop {
  position: absolute;
  z-index: 1000;
  transition: all 0.1s linear;
}

.troop-icon {
  width: 30px;
  height: 30px;
  transform: translate(-50%, -50%); /* Centraliza o ícone da tropa */
}

.movement-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 999;
}


.arrow-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1001;
}

.movement-layer,
.arrow-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1000;
  overflow: visible;
}

.map-wrapper {
  display: flex;
  flex-direction: column;
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

.map-container {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  border: 1px solid #000;
}

.map {
  position: absolute;
  width: calc(30 * 100px);
  height: calc(30 * 50px);
  transform-origin: top left;
  will-change: transform;
}

.tile {
  position: absolute;
  width: 100px; /* TILE_WIDTH */
  height: 50px; /* TILE_HEIGHT */
  background-color: #2b2829;
  clip-path: polygon(50% 0%, 100% 50%, 50% 100%, 0% 50%);
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s;
  z-index: 1;
}

.tile:hover {
  filter: brightness(1.2);
  z-index: 2;
}

.coordinate-label {
  position: absolute;
  top: 50%;
  left: 50%;
  opacity: 30%;
  transform: translate(-50%, -50%);
  font-size: 10px;
  color: #ffffff;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 1px 3px;
  border-radius: 2px;
  z-index: 3;
  pointer-events: none;
}

.village {
  width: 100%;
  height: 100%;
  /* background-color: red; */
  display: flex;
  justify-content: center;
  align-items: center;
}

.special-element {
  z-index: 1;
}

.special-element-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}


.coordinates {
  position: absolute;
  z-index: 10;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;
}

.x-coordinates {
  bottom: 0;
  left: 0;
  right: 0;
  height: 20px;
  display: flex;
}

.y-coordinates {
  top: 0;
  left: 0;
  bottom: 0;
  width: 20px;
  display: flex;
  flex-direction: column-reverse;
}

.selected-tile {
  position: fixed;
  bottom: 10px;
  right: 10px;
  background-color: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px 10px;
  border-radius: 5px;
  z-index: 1000;
}
</style>
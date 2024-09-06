<template>
    <div class="map-wrapper">
      <div class="map-container" ref="mapContainer">
        <div 
          class="map" 
          :style="{ transform: `translate(${mapPosition.x}px, ${mapPosition.y}px)` }"
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
            :style="{ left: `${tile.x * tileSize}px`, top: `${tile.y * tileSize}px` }"
          >
            <img v-if="tile.url" :src="tile.url" :alt="`Special element at (${tile.x}, ${tile.y})`" class="special-element-image">
            <div v-if="tile.x === villagePosition.x && tile.y === villagePosition.y" class="village">
              <img src="https://i.imgur.com/ArPQ6lM.jpeg" alt="City Tile" class="tile-image">
            </div>
          </div>
          
          <div class="coordinates y-coordinates" :style="{ left: `${mapPositionPx.left}px` }">
            <div v-for="y in mapSize" :key="'y-'+y" class="coordinate-label">
              {{ mapSize - y }}
            </div>
          </div>
          <div class="coordinates x-coordinates" :style="{ top: `${mapPositionPx.top}px` }">
            <div v-for="x in mapSize" :key="'x-'+x" class="coordinate-label">
              {{ x - 1 }}
            </div>
          </div>
  
          <!-- Tropas em movimento -->
          <div v-for="troop in movingTroops" :key="troop.id" class="troop"
                :style="{ left: `${troop.position.x}px`, top: `${troop.position.y}px` }">
            <img :src="troop.img" :alt="troop.tipo" class="troop-icon">
          </div>
          
          <!-- Linhas de movimento -->
          <svg class="movement-layer" :viewBox="`0 0 ${mapSize * tileSize} ${mapSize * tileSize}`">
            <line v-for="troop in movingTroops" :key="`line-${troop.id}`"
                  :x1="troop.start.x * tileSize + tileSize / 2"
                  :y1="troop.start.y * tileSize + tileSize / 2"
                  :x2="troop.end.x * tileSize + tileSize / 2"
                  :y2="troop.end.y * tileSize + tileSize / 2"
                  stroke="#ff0000"
                  stroke-width="2"
                  stroke-dasharray="5,5" />
          </svg>
          
          <svg class="arrow-layer" :viewBox="`0 0 ${mapSize * tileSize} ${mapSize * tileSize}`">
            <defs>
              <marker id="arrowhead" markerWidth="10" markerHeight="7" 
                refX="10" refY="3.5" orient="auto">
                <polygon points="0 0, 10 3.5, 0 7" fill="#ff0000" />
              </marker>
            </defs>
            <line v-if="selectedTile"
              :x1="(villagePosition.x + 0.5) * tileSize"
              :y1="(villagePosition.y + 0.5) * tileSize"
              :x2="(selectedTile.x + 0.5) * tileSize"
              :y2="(selectedTile.y + 0.5) * tileSize"
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
  import TroopControlPanel from './components/TroopsControlPanel.vue';
  
  const mapSize = ref(30);
  const tileSize = ref(50);
  const mapPosition = reactive({ x: 0, y: 0 });
  const mapPositionPx = reactive({ left: 0, top: 0 });
  const isDragging = ref(false);
  const startPosition = reactive({ x: 0, y: 0 });
  const tiles = ref([]);
  const selectedTile = ref(null);
  const villagePosition = reactive({ x: 10, y: 10 });
  const mapContainer = ref(null);
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
      url: 'https://i.imgur.com/ArPQ6lM.jpeg'
    },
    {
      x: 15,
      y: 2,
      url: 'https://i.imgur.com/wOFsUmN.jpeg'
    }
  ]);
  
  
  const movingTroops = ref([]);
  let animationFrameId = null;
  
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
    movingTroops.value = [...movingTroops.value];
  
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
  
  onMounted(() => {
    generateTiles();
    centerOnVillage();
    window.addEventListener('resize', centerOnVillage);
    
    // Iniciar a animação
    animationFrameId = requestAnimationFrame(updateTroopPositions);
  });
  
  onUnmounted(() => {
    window.removeEventListener('resize', centerOnVillage);
    
    // Cancelar a animação
    if (animationFrameId) {
      cancelAnimationFrame(animationFrameId);
    }
  });
  
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
    const containerWidth = mapContainer.value.clientWidth;
    const containerHeight = mapContainer.value.clientHeight;
    mapPosition.x = (containerWidth / 2) - ((villagePosition.x + 0.5) * tileSize.value);
    mapPosition.y = (containerHeight / 2) - ((villagePosition.y + 0.5) * tileSize.value);
  }
  
  function startDragging(event) {
    isDragging.value = true;
    startPosition.x = event.clientX - mapPosition.x;
    startPosition.y = event.clientY - mapPosition.y;
  }
  
  function drag(event) {
    if (!isDragging.value) return;
  
    const newX = event.clientX - startPosition.x;
    const newY = event.clientY - startPosition.y;
    const maxX = 0;
    const minX = -(mapSize.value * tileSize.value - mapContainer.value.clientWidth);
    const maxY = 0;
    const minY = -(mapSize.value * tileSize.value - mapContainer.value.clientHeight);
  
    mapPosition.x = Math.max(minX, Math.min(maxX, newX));
    mapPosition.y = Math.max(minY, Math.min(maxY, newY));
  
    mapPositionPx.left = -mapPosition.x;
    mapPositionPx.top = -mapPosition.y;
  }
  
  function stopDragging() {
    isDragging.value = false;
  }
  
  function handleMapClick(event) {
    if (isDragging.value) return;
    
    const rect = mapContainer.value.getBoundingClientRect();
    const x = Math.floor((event.clientX - rect.left - mapPosition.x) / tileSize.value);
    const y = Math.floor((event.clientY - rect.top - mapPosition.y) / tileSize.value);
    
    const clickedTile = tiles.value.find(tile => tile.x === x && tile.y === y);
    if (clickedTile) {
      selectedTile.value = clickedTile;
      showTroopPanel.value = true; // Mostrar o painel quando um tile é selecionado
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
    transition: all 0.1s linear;
    z-index: 1000;
  }
  
  .troop-icon {
    width: 30px;
    height: 30px;
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
  
  .movimento-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 999;
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
    width: calc(30 * 50px);
    height: calc(30 * 50px);
    cursor: move;
    pointer-events: auto;
  }
  
  .tile {
    width: 50px;
    height: 50px;
    position: absolute;
    border-right: 1px solid rgb(255,255,255,0.5);
    border-bottom: 1px solid rgb(255,255,255,0.5);
    background-color: #a8de5d;
    box-sizing: border-box;
    overflow: hidden;
    pointer-events: auto;
  }
  
  .village {
    width: 50px;
    height: 50px;
    background-color: red;
    border-radius: 50%;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
  
  .special-element {
    z-index: 1;
  }
  
  .special-element-image {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
  
  .arrow-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 9999;
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
  
  .coordinate-label {
    flex: 0 0 50px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    box-sizing: border-box;
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
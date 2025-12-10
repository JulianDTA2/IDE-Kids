<template>
  <div class="kids-ide-container">
    <div class="stars-background">
      <div 
        v-for="star in stars" 
        :key="star.id"
        class="star"
        :style="{
          left: star.left + '%',
          top: star.top + '%',
          animationDelay: star.delay + 's',
          animationDuration: star.duration + 's'
        }"
      ></div>
    </div>

    <div class="main-wrapper">
      <header class="clay-header animate-slideDown">
        <div class="header-content">
          <div class="logo-area animate-bounce-gentle">
            <div>
                <!--<img :src="logo1" alt="Logo RM" class="app-logo" />
                <span class="divider-logo">|</span>-->
                <img :src="logo2" alt="Logo Mark" class="app-logo" />
            </div>
            <!--//<div class="mobile-hide">
              <h1 class="logo-title">MarkRobot IDE</h1>
              <p class="logo-subtitle">¡Programa tu mundo!</p>
            </div>-->
          </div>

          <div class="hardware-controls">
            <select 
              v-model="selectedBoardFqbn" 
              class="clay-select board-select"
              title="Seleccionar Placa"
            >
              <option value="arduino:avr:uno">Arduino Uno</option>
              <option value="arduino:avr:nano">Arduino Nano</option>
              <option value="arduino:avr:mega">Arduino Mega</option>
              <option value="esp8266:esp8266:nodemcuv2">NodeMCU 1.0</option>
              <option value="esp32:esp32:esp32">ESP32 Dev Module</option>
              <option disabled>──────────────</option>
              <option v-for="b in allKnownBoards" :key="b.fqbn" :value="b.fqbn">
                  {{ b.name }}
              </option>
            </select>

            <div class="port-group">
              <select 
                v-model="selectedPort" 
                class="clay-select port-select"
                :title="selectedPort ? 'Puerto: ' + selectedPort : 'Sin puerto'"
              >
                <option value="" disabled>
                  {{ availablePorts.length === 0 ? 'Sin Puertos' : 'Seleccionar Puerto' }}
                </option>
                <option v-for="p in availablePorts" :key="p.address" :value="p.address">
                  {{ p.label }}
                </option>
              </select>
              
              <button @click="refreshPorts" class="clay-btn-icon btn-refresh" title="Refrescar Puertos">
                <svg class="icon-refresh" :class="{ 'spinning': isRefreshing }" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                  <path d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
              </button>
            </div>

            <input
              v-model="sketchName"
              type="text"
              class="clay-input sketch-name"
              placeholder="MiProyecto"
            />
          </div>

          <div class="action-buttons">
            <button 
              @click="verifyCode" 
              :disabled="isCompiling"
              class="clay-btn btn-verify"
              title="Verificar Código"
            >
              <svg v-if="!isCompiling" class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M5 13l4 4L19 7" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
              <svg v-else class="icon spinning" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M12 2v4m0 12v4M4.93 4.93l2.83 2.83m8.48 8.48l2.83 2.83M2 12h4m12 0h4M4.93 19.07l2.83-2.83m8.48-8.48l2.83-2.83" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
              <span class="btn-text">Verificar</span>
            </button>
            

            <button 
              @click="uploadCode"
              :disabled="isUploading"
              class="clay-btn btn-upload"
              title="Subir a la Placa"
            >
              <svg v-if="!isUploading" class="icon icon-upload" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
              <svg v-else class="icon spinning" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M12 2v4m0 12v4M4.93 4.93l2.83 2.83m8.48 8.48l2.83 2.83M2 12h4m12 0h4M4.93 19.07l2.83-2.83m8.48-8.48l2.83-2.83" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
              <span class="btn-text">Subir</span>
            </button>
            <button @click="openInArduino" class="clay-btn btn-extra" title="Abrir en Arduino IDE Nativo">
              <svg class="icon" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"></path></svg>
              <span class="btn-text">IDE</span>
            </button>

            <div class="divider"></div>

            <button @click="installAvrCore" :disabled="isInstalling" class="clay-btn-icon btn-extra" title="Instalar Drivers AVR">
                 <svg v-if="isInstalling" class="icon spinning" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"></path></svg>
                 <svg v-else class="icon" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4"></path></svg>
            </button>

            <button @click="saveSketch" class="clay-btn btn-save" title="Guardar Proyecto">
    <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
      <path d="M8 7H5a2 2 0 00-2 2v9a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-3m-1 4l-3 3m0 0l-3-3m3 3V4" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
    <span class="btn-text">Guardar</span>
  </button>

  <button @click="loadSketch" class="clay-btn btn-load" title="Abrir Proyecto">
    <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
      <path d="M5 19a2 2 0 01-2-2V7a2 2 0 012-2h4l2 2h4a2 2 0 012 2v1M5 19h14a2 2 0 002-2v-5a2 2 0 00-2-2H9a2 2 0 00-2 2v5a2 2 0 01-2 2z" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
    <span class="btn-text">Abrir</span>
  </button>

            <button @click="clearWorkspace" class="clay-btn-icon btn-delete" title="Limpiar Todo">
              <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </button>
          </div>
        </div>
      </header>

      <main class="main-content">
        <div class="workspace-area clay-card animate-slideUp">
          <div ref="blocklyDiv" class="blockly-area"></div>
        </div>

        <aside class="sidebar clay-card animate-slideLeft">
            <div class="tabs">
               <button class="tab-btn active">
                   <svg class="tab-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 20l4-16m4 4l4 4-4 4M6 16l-4-4 4-4"></path></svg>
                   Código C++
               </button>
            </div>

          <div class="code-section">
            <textarea
              class="clay-textarea code-display"
              readonly
              :value="generatedCode"
              placeholder="// Tu código aparecerá aquí..."
            ></textarea>
          </div>

          <div :class="['output-section', 'clay-card-inner', { collapsed: !showOutput }]">
            <button @click="showOutput = !showOutput" class="output-header">
              <div class="header-left">
                <svg class="header-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                  <path d="M9.75 17L9 20l-1 1h8l-1-1-.75-3M3 13h18M5 17h14a2 2 0 002-2V5a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                <span>Consola MarkRobot</span>
              </div>
              <svg class="chevron" :class="{ rotated: !showOutput }" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M19 9l-7 7-7-7" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </button>
            
            <div v-show="showOutput" class="console-content">
              <button @click.stop="outputLog = ''" class="clear-btn">Limpiar</button>
              <pre class="console-text">{{ outputLog }}</pre>
            </div>
          </div>
        </aside>
      </main>
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref, computed } from 'vue'; 
import * as Blockly from 'blockly';
import * as En from 'blockly/msg/en';
import 'blockly/blocks'; 
import ArduinoGenerator from '../arduino_core.js'; 
import logo2 from '../../../../resources/logo_m4rk.webp';

Blockly.setLocale(En);

const blocklyDiv = ref(null);
const generatedCode = ref("");
const generatedXml = ref("");
const outputLog = ref("¡Bienvenido a MarkRobot IDE! \nSelecciona tu placa y comienza a crear.");
const isCompiling = ref(false);
const isUploading = ref(false);
const isInstalling = ref(false);
const isRefreshing = ref(false); 
const showOutput = ref(true); 
const sketchName = ref("MiProyecto");
const stars = ref([]); 

const rawPorts = ref([]); 
const selectedPort = ref("");
const allKnownBoards = ref([]); 
const selectedBoardFqbn = ref("arduino:avr:uno"); 

let workspace = null;

const availablePorts = computed(() => {
    if (!rawPorts.value || !Array.isArray(rawPorts.value)) return [];

    return rawPorts.value.map(p => {
        let portName = p.address;
        if (!portName && p.port && p.port.address) {
            portName = p.port.address;
        }
        if (!portName) return null;

        return {
            address: portName,
            label: portName + (p.boardName ? ` (${p.boardName})` : ''), 
        };
    }).filter(p => p !== null); 
});

async function refreshPorts() {
  isRefreshing.value = true;
  if (!window.api) {
      setTimeout(() => { isRefreshing.value = false; }, 500);
      return;
  }
  
  outputLog.value += "\nBuscando puertos...";
  
  try {
    const ports = await window.api.listBoards(); 
    rawPorts.value = ports || [];
    
    setTimeout(() => {
        if (availablePorts.value.length > 0) {
            if (!selectedPort.value) {
                selectedPort.value = availablePorts.value[0].address;
            }
            outputLog.value += `\n¡Encontrados: ${availablePorts.value.length}!`;
        } else {
            outputLog.value += "\nNo se encontraron puertos (revisa el cable USB).";
        }
        isRefreshing.value = false;
    }, 100);
    
  } catch (e) {
    outputLog.value += `\nError al buscar: ${e.message}`;
    isRefreshing.value = false;
  }
}

async function verifyCode() {
  if (isCompiling.value) return;
  if (!window.api) { alert("Backend no conectado (Modo Diseño)"); return; }

  isCompiling.value = true;
  showOutput.value = true;
  outputLog.value = "Compilando tu código...\n";
  updateContent(); 

  try {
    const res = await window.api.compile({
      code: generatedCode.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    outputLog.value += res.log;
    if (res.success) outputLog.value += "\n✅ ¡Todo listo! Tu código está perfecto.";
  } catch (e) {
    outputLog.value += "\nError crítico: " + e.message;
  } finally {
    isCompiling.value = false;
  }
}

async function uploadCode() {
  if (isUploading.value) return;
  if (!window.api) { alert("Backend no conectado"); return; }
  
  if (!selectedPort.value) {
    alert("¡Espera! Necesitas seleccionar un puerto primero.");
    return;
  }
  
  isUploading.value = true;
  showOutput.value = true;
  outputLog.value = `📤 Subiendo a ${selectedPort.value}...\n¡Preparando el despegue!\n`;
  updateContent();
  
  try {
    const compileRes = await window.api.compile({
      code: generatedCode.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    
    if (!compileRes.success) {
      outputLog.value += compileRes.log + "\n⚠️ Subida cancelada por error de compilación.";
      isUploading.value = false;
      return;
    }

    outputLog.value += "\nCompilación OK. Subiendo...\n";
    
    const uploadRes = await window.api.upload({
      port: selectedPort.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    
    outputLog.value += uploadRes.log;
    if (uploadRes.success) outputLog.value += "\n🚀 ¡SUBIDA COMPLETADA! Tu robot está vivo.";
  } catch (e) {
    outputLog.value += "\nError crítico en subida: " + e.message;
  } finally {
    isUploading.value = false;
  }
}

async function installAvrCore() {
  if (isInstalling.value) return;
  if (!window.api) return;

  if(!confirm("Esto descargará las herramientas para Arduino. ¿Continuar?")) return;

  isInstalling.value = true;
  showOutput.value = true;
  outputLog.value += "\n⬇ Iniciando descarga de herramientas...\n";
  
  try {
    const res = await window.api.installCore('arduino:avr');
    outputLog.value += res.log;
    
    if (res.success) {
        outputLog.value += "\n✅ Herramientas instaladas correctamente.";
        window.api.listAllBoards().then(data => {
            if(data && data.boards) allKnownBoards.value = data.boards;
        });
    } else {
        outputLog.value += "\n⚠️ Hubo un problema en la instalación.";
    }
  } catch (e) {
    outputLog.value += "\nError crítico: " + e.message;
  } finally {
    isInstalling.value = false;
  }
}

async function openInArduino() {
  outputLog.value += "\nAbriendo en Arduino IDE nativo...";
  if(window.api) {
      await window.api.openIde({
          code: generatedCode.value, 
          sketchName: sketchName.value || 'Sketch'
      });
  }
}

async function saveSketch() {
  if (window.api) {
    const result = await window.api.saveFile(generatedXml.value, sketchName.value);
    if(result.success) {
        outputLog.value += `\n💾 Proyecto guardado en: ${result.path}`;
    }
  }
}

async function loadSketch() {
  if (window.api) {
    const result = await window.api.openFile();
    if(!result.canceled && result.content) {
        workspace.clear();
        try {
            const xml = Blockly.utils.xml.textToDom(result.content);
            Blockly.Xml.domToWorkspace(xml, workspace);
            outputLog.value += "\n📂 Proyecto cargado exitosamente.";
            if(result.fileName) {
                sketchName.value = result.fileName.replace(/\.[^/.]+$/, "");
            }
        } catch(e) {
            outputLog.value += "\nError al abrir el archivo.";
            console.error(e);
        }
    }
  }
}

function clearWorkspace() {
    if(confirm("¿Borrar todos los bloques y empezar de cero?")) {
        workspace.clear();
        insertStartBlock();
        outputLog.value += "\n✨ Espacio de trabajo limpio.";
    }
}

const toolbox = {
  kind: 'categoryToolbox',
  contents: [
    { kind: 'category', name: 'Variables', colour: '#a855f7', contents: [
      { kind: 'block', type: 'arduino_start' },
      { kind: 'button', text: 'Crear Variable', callbackKey: 'CREATE_VARIABLE' },
      { kind: 'block', type: 'variables_get' },
      { kind: 'block', type: 'variables_set' },
      { kind: 'block', type: 'variables_set_type' },
      { kind: 'block', type: 'type_cast' }
    ]},
    { kind: 'category', name: 'Lógica', colour: '#3b82f6', contents: [
      { kind: 'block', type: 'controls_if' },
      { kind: 'block', type: 'logic_compare' },
      { kind: 'block', type: 'logic_operation' },
      { kind: 'block', type: 'logic_boolean' },
      { kind: 'block', type: 'logic_negate' },
      { kind: 'block', type: 'logic_null' },
      { kind: 'block', type: 'logic_ternary' }
    ]},
    { kind: 'category', name: 'Bucles', colour: '#10b981', contents: [
      { kind: 'block', type: 'controls_repeat_ext', inputs: { TIMES: { shadow: { type: 'math_number', fields: { NUM: 10 } } } } },
      { kind: 'block', type: 'controls_whileUntil' },
      { kind: 'block', type: 'controls_for', inputs: { FROM: { shadow: { type: 'math_number', fields: { NUM: 1 } } }, TO: { shadow: { type: 'math_number', fields: { NUM: 10 } } }, BY: { shadow: { type: 'math_number', fields: { NUM: 1 } } } } },
      { kind: 'block', type: 'controls_flow_statements' }
    ]},
    { kind: 'category', name: 'Matemáticas', colour: '#f59e0b', contents: [
      { kind: 'block', type: 'math_number' },
      { kind: 'block', type: 'math_arithmetic' },
      { kind: 'block', type: 'math_random_int' },
      { kind: 'block', type: 'math_map' },
      { kind: 'block', type: 'math_single' },
      { kind: 'block', type: 'math_constrain' }
    ]},
    { kind: 'category', name: 'Texto', colour: '#ec4899', contents: [
      { kind: 'block', type: 'text' },
      { kind: 'block', type: 'text_print' },
      { kind: 'block', type: 'text_join' },
      { kind: 'block', type: 'text_length' }
    ]},
    { kind: 'sep' },
    { kind: 'category', name: 'Entrada/Salida', colour: '#6366f1', contents: [
      { kind: 'block', type: 'digital_write' },
      { kind: 'block', type: 'digital_read' },
      { kind: 'block', type: 'analog_read' },
      { kind: 'block', type: 'analog_write' },
      { kind: 'block', type: 'custom_delay' }
    ]},
    { kind: 'category', name: 'Motores', colour: '#ef4444', contents: [
      { kind: 'block', type: 'motor_setup' },
      { kind: 'block', type: 'motor_run' },
      { kind: 'block', type: 'motor_stop' }
    ]},
    { kind: 'category', name: 'Pantallas (8x8)', colour: '#D35400', contents: [
      { kind: 'block', type: 'display_8x8_setup' },
      { kind: 'block', type: 'display_8x8_draw' }
    ]},
    { kind: 'category', name: 'Sensores', colour: '#8b5cf6', contents: [
      { kind: 'block', type: 'ultrasonic_read' },
      { kind: 'block', type: 'color_sensor_read' },
      { kind: 'block', type: 'sound_sensor_read' }
    ]},
    { kind: 'category', name: 'Conexión', colour: '#06b6d4', contents: [
      { kind: 'block', type: 'wifi_connect' },
      { kind: 'block', type: 'wifi_is_connected' },
      { kind: 'block', type: 'bluetooth_setup' },
      { kind: 'block', type: 'bluetooth_read_string' },
      { kind: 'block', type: 'bluetooth_send_string' },
      { kind: 'block', type: 'rm_bluetooth_read' }
    ]}
  ]
};

function updateContent() {
  if (!workspace) return;
  try {
    const code = ArduinoGenerator.workspaceToCode(workspace);
    generatedCode.value = code;
    const xmlDom = Blockly.Xml.workspaceToDom(workspace);
    generatedXml.value = Blockly.Xml.domToPrettyText(xmlDom);
  } catch (e) {
    console.error(e);
  }
}

function insertStartBlock() {
    if(!workspace) return;
    const startBlock = workspace.newBlock('arduino_start');
    startBlock.initSvg();
    startBlock.render();
    workspace.centerOnBlock(startBlock.id);
    updateContent(); 
}

onMounted(async () => {
    stars.value = Array.from({ length: 25 }, (_, i) => ({
        id: i,
        left: Math.random() * 100,
        top: Math.random() * 100,
        delay: Math.random() * 5,
        duration: 3 + Math.random() * 4
    }));

  if (blocklyDiv.value) {
    workspace = Blockly.inject(blocklyDiv.value, {
      toolbox: toolbox,
      scrollbars: true,
      media: 'media/', 
      zoom: { controls: true, wheel: true, startScale: 1.0, maxScale: 3, minScale: 0.3, scaleSpeed: 1.2 },
      renderer: 'geras',
      theme: {
        'base': 'classic',
        'fontStyle': { 'family': 'Segoe UI, sans-serif', 'weight': 'bold', 'size': 12 },
        'componentStyles': { 
            'workspaceBackgroundColour': 'transparent', 
            'toolboxBackgroundColour': 'rgba(255, 255, 255, 0.5)', 
            'flyoutBackgroundColour': 'rgba(255, 255, 255, 0.8)',
            'scrollbarColour': '#a855f7'
        }
      }
    });

    workspace.registerButtonCallback('CREATE_VARIABLE', (button) => {
      Blockly.Variables.createVariableButtonHandler(button.getTargetWorkspace(), null, '');
    });
    
    workspace.addChangeListener(updateContent);
    window.addEventListener('resize', () => Blockly.svgResize(workspace));
    
    setTimeout(() => {
        insertStartBlock();
        Blockly.svgResize(workspace);
    }, 100);

    await refreshPorts();
    
    if(window.api) {
        window.api.listAllBoards().then(data => {
            if(data && data.boards) allKnownBoards.value = data.boards;
        }).catch(err => console.error("Error loading boards", err));
    }
  }
});
</script>

<style scoped>
/* Root Variables */
:root {
  --primary-purple: #a855f7;
  --primary-pink: #ec4899;
  --primary-blue: #3b82f6;
  --bg-light: #faf5ff;
  --shadow-light: rgba(255, 255, 255, 0.9);
  --shadow-dark: rgba(190, 190, 220, 0.4);
}

/* Main Container */
.kids-ide-container {
  position: relative;
  min-height: 100vh;
  background: linear-gradient(135deg, #faf5ff 0%, #fce7f3 50%, #dbeafe 100%);
  overflow: hidden;
  font-family: 'Segoe UI', 'Comic Sans MS', sans-serif;
}

.stars-background {
  position: absolute;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

.star {
  position: absolute;
  width: 8px;
  height: 8px;
  background: #fbbf24;
  border-radius: 50%;
  animation: twinkle 3s ease-in-out infinite;
  box-shadow: 0 0 10px #fbbf24;
}

.main-wrapper {
  position: relative;
  z-index: 10;
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 16px;
  gap: 16px;
}

/* Clay Morphism Styles */
.clay-card {
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  border-radius: 30px;
  box-shadow: 
    20px 20px 60px var(--shadow-dark),
    -20px -20px 60px var(--shadow-light),
    inset 2px 2px 4px rgba(255, 255, 255, 0.5);
  border: 3px solid rgba(255, 255, 255, 0.6);
}

.clay-card-inner {
  background: rgba(240, 240, 255, 0.6);
  border-radius: 20px;
  box-shadow: 
    inset 8px 8px 16px rgba(190, 190, 220, 0.3),
    inset -8px -8px 16px rgba(255, 255, 255, 0.7);
}

.clay-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  border-radius: 20px;
  border: none;
  color: white;
  font-weight: 900;
  cursor: pointer;
  box-shadow: 
    8px 8px 16px rgba(0, 0, 0, 0.15),
    -4px -4px 12px rgba(255, 255, 255, 0.7),
    inset 2px 2px 4px rgba(255, 255, 255, 0.3);
  transition: all 0.3s ease;
  font-size: 14px;
}

.clay-btn:hover:not(:disabled) {
  transform: translateY(-3px);
  box-shadow: 
    12px 12px 24px rgba(0, 0, 0, 0.2),
    -6px -6px 16px rgba(255, 255, 255, 0.8);
}

.clay-btn:active:not(:disabled) {
  transform: translateY(1px);
  box-shadow: 
    4px 4px 8px rgba(0, 0, 0, 0.2),
    inset 4px 4px 8px rgba(0, 0, 0, 0.1);
}

.clay-btn:disabled {
  opacity: 0.6;
  filter: grayscale(0.8);
  cursor: not-allowed;
}

.clay-btn-icon {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 18px;
  border: none;
  color: white;
  cursor: pointer;
  box-shadow: 
    8px 8px 16px rgba(0, 0, 0, 0.15),
    -4px -4px 12px rgba(255, 255, 255, 0.7),
    inset 2px 2px 4px rgba(255, 255, 255, 0.3);
  transition: all 0.3s ease;
}

.clay-btn-icon:hover:not(:disabled) {
  transform: scale(1.1) rotate(5deg);
}

.clay-btn-icon:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    background: #ccc !important;
}

.clay-select,
.clay-input {
  padding: 12px 16px;
  border-radius: 18px;
  border: 3px solid rgba(200, 200, 230, 0.5);
  background: rgba(255, 255, 255, 0.9);
  box-shadow: 
    inset 6px 6px 12px rgba(200, 200, 230, 0.3),
    inset -6px -6px 12px rgba(255, 255, 255, 0.9);
  outline: none;
  transition: all 0.3s ease;
  font-weight: 700;
  color: #4b5563;
}

.clay-select:focus,
.clay-input:focus {
  border-color: rgba(168, 85, 247, 0.6);
  box-shadow: 
    inset 8px 8px 16px rgba(168, 85, 247, 0.2),
    inset -8px -8px 16px rgba(255, 255, 255, 0.9),
    0 0 20px rgba(168, 85, 247, 0.3);
}

.clay-textarea {
  width: 100%;
  height: 100%;
  padding: 16px;
  border-radius: 18px;
  border: 3px solid rgba(200, 200, 230, 0.5);
  background: rgba(255, 255, 255, 0.9);
  box-shadow: 
    inset 6px 6px 12px rgba(200, 200, 230, 0.3),
    inset -6px -6px 12px rgba(255, 255, 255, 0.9);
  outline: none;
  resize: none;
  font-family: 'Consolas', monospace;
  font-size: 13px;
  color: #4b5563;
}

/* Header */
.clay-header {
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px);
  border-radius: 30px;
  padding: 16px 20px;
  box-shadow: 
    20px 20px 60px var(--shadow-dark),
    -20px -20px 60px var(--shadow-light);
}

.header-content {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 15px;
}

.logo-area {
  display: flex;
  align-items: center;
  gap: 15px;
}

.logo-container {
    display: flex;
    align-items: center;
    background: rgba(255,255,255,0.5);
    padding: 5px 15px;
    border-radius: 20px;
    box-shadow: inset 4px 4px 8px rgba(0,0,0,0.05);
}

.app-logo {
    height: 40px;
    width: auto;
    object-fit: contain;
}

.divider-logo {
    color: #a855f7;
    margin: 0 10px;
    font-size: 20px;
    opacity: 0.5;
}

.logo-title {
  font-size: 24px;
  font-weight: 900;
  background: linear-gradient(135deg, #a855f7, #ec4899);
  -webkit-background-clip: text;
  background-clip: text;
  -webkit-text-fill-color: transparent;
  margin: 0;
}

.logo-subtitle {
  font-size: 12px;
  color: #a855f7;
  font-weight: 700;
  margin: 0;
}

.hardware-controls {
  display: flex;
  gap: 12px;
  align-items: center;
  flex-wrap: wrap;
}

.board-select {
  min-width: 140px;
  color: #7c3aed;
}

.port-group {
  display: flex;
  gap: 8px;
}

.port-select {
  min-width: 120px;
  color: #2563eb;
}

.sketch-name {
  text-align: center;
  font-weight: 900;
  color: #ec4899;
  width: 140px;
}

.action-buttons {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}

.btn-verify { background: linear-gradient(135deg, #3b82f6, #2563eb); }
.btn-upload { background: linear-gradient(135deg, #10b981, #059669); }
.btn-refresh { background: linear-gradient(135deg, #f59e0b, #d97706); }
.btn-save { background: linear-gradient(135deg, #fbbf24, #f59e0b); }
.btn-load { background: linear-gradient(135deg, #8b5cf6, #7c3aed); }
.btn-delete { background: linear-gradient(135deg, #ef4444, #dc2626); }
.btn-extra { background: linear-gradient(135deg, #6366f1, #4f46e5); }

.divider {
  width: 2px;
  height: 32px;
  background: rgba(200, 200, 230, 0.5);
}

.icon { width: 20px; height: 20px; }
.icon-upload { transition: transform 0.3s ease; }
.btn-upload:hover .icon-upload { transform: translateY(-4px); }
.spinning { animation: spin 1s linear infinite; }

/* Main Content */
.main-content {
  flex: 1;
  display: flex;
  gap: 16px;
  overflow: hidden;
}

.workspace-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 10px; /* Menos padding para más espacio de bloques */
  overflow: hidden;
}

.blockly-area {
  flex: 1;
  width: 100%;
  height: 100%;
  border-radius: 20px;
  overflow: hidden;
}

/* Sidebar */
.sidebar {
  width: 350px;
  display: flex;
  flex-direction: column;
  padding: 20px;
  gap: 16px;
}

.tabs { display: flex; gap: 8px; margin-bottom: 8px; }
.tab-btn {
  display: flex; align-items: center; gap: 8px;
  padding: 10px 20px; border-radius: 16px; border: none;
  background: rgba(230, 230, 250, 0.6);
  font-weight: 800; color: #6b7280; font-size: 14px;
}
.tab-btn.active {
  background: linear-gradient(135deg, #a855f7, #ec4899);
  color: white;
  box-shadow: 4px 4px 10px rgba(168, 85, 247, 0.3);
}
.tab-icon { width: 18px; height: 18px; }

.code-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.code-display { flex: 1; font-size: 12px; line-height: 1.4; }

.output-section {
  display: flex; flex-direction: column;
  transition: all 0.3s ease; overflow: hidden;
}
.output-section.collapsed { max-height: 56px; }

.output-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 12px 16px; background: transparent; border: none;
  cursor: pointer; font-weight: 800; color: #7c3aed;
  transition: color 0.3s ease; width: 100%;
}
.output-header:hover { color: #ec4899; }

.header-left { display: flex; align-items: center; gap: 8px; font-size: 14px; }
.header-icon { width: 20px; height: 20px; }
.chevron { width: 20px; height: 20px; transition: transform 0.3s ease; }
.chevron.rotated { transform: rotate(180deg); }

.console-content {
  flex: 1;
  background: #1f2937;
  border-radius: 16px;
  margin: 0 12px 12px 12px;
  padding: 16px;
  overflow: auto;
  position: relative;
  max-height: 200px;
  min-height: 150px;
}

.clear-btn {
  position: absolute; top: 8px; right: 12px;
  padding: 4px 8px; background: rgba(239, 68, 68, 0.2);
  border: none; border-radius: 8px;
  color: #f87171; font-size: 11px; font-weight: 700;
  cursor: pointer;
}
.clear-btn:hover { background: rgba(239, 68, 68, 0.4); color: #fee2e2; }

.console-text {
  margin: 0; color: #10b981;
  font-family: 'Consolas', monospace; font-size: 12px;
  white-space: pre-wrap; padding-top: 20px;
}

/* Animations */
@keyframes slideDown { from { opacity: 0; transform: translateY(-30px); } to { opacity: 1; transform: translateY(0); } }
@keyframes slideUp { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: translateY(0); } }
@keyframes slideLeft { from { opacity: 0; transform: translateX(30px); } to { opacity: 1; transform: translateX(0); } }
@keyframes bounce-gentle { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-5px); } }
@keyframes twinkle { 0%, 100% { opacity: 0.3; transform: scale(1); } 50% { opacity: 1; transform: scale(1.5); } }
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }

.animate-slideDown { animation: slideDown 0.6s ease-out; }
.animate-slideUp { animation: slideUp 0.6s ease-out 0.2s both; }
.animate-slideLeft { animation: slideLeft 0.6s ease-out 0.3s both; }
.animate-bounce-gentle { animation: bounce-gentle 2s ease-in-out infinite; }

/* Responsive */
@media (max-width: 1024px) {
  .sidebar { width: 300px; }
  .btn-text { display: none; }
  .clay-btn { padding: 12px; }
  .logo-title, .logo-subtitle { display: none; } /* Ocultar texto de logo en tablets */
}

@media (max-width: 768px) {
  .main-content { flex-direction: column; }
  .sidebar { width: 100%; max-height: 400px; }
  .header-content { justify-content: center; }
  .hardware-controls, .action-buttons { justify-content: center; width: 100%; }
  .sketch-name { width: 100%; margin-top: 10px; order: -1; }
}
</style>
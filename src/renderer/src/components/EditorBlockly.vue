<template>
  <div class="kids-layout">
    
    <header class="top-bar-kids">
      <div class="brand-area">
          <div class="logo-container bouncy">
            <img :src="logo1" alt="Logo RM" class="app-logo" />
          </div>
          <div class="project-title-wrapper">
             <span class="label-tiny">MI PROYECTO:</span>
             <input 
                type="text" 
                v-model="sketchName" 
                class="kids-input-title" 
                placeholder="¡Nombra tu invento!" 
            />
          </div>
      </div>
      
      <div class="hardware-island">
        <div class="select-wrapper">
            <span class="icon-label">🤖</span>
            <select v-model="selectedBoardFqbn" class="kids-select" title="¿Qué cerebro usaremos?">
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
        </div>

        <div class="select-wrapper">
            <span class="icon-label">🔌</span>
            <select v-model="selectedPort" class="kids-select port-select">
                <option value="" disabled>
                  {{ availablePorts.length === 0 ? '🚫 Sin conexión' : 'Seleccionar Cable' }}
                </option>
                <option v-for="p in availablePorts" :key="p.address" :value="p.address">
                    {{ p.label }}
                </option>
            </select>
            <button @click="refreshPorts" class="kids-btn-mini rotate-hover" title="Buscar">🔄</button>
        </div>
      </div>

      <div class="big-actions">
        <button class="kids-big-btn btn-blue" @click="verifyCode" :class="{ 'loading': isCompiling }">
            <div class="btn-icon">
                 <span v-if="!isCompiling">🔍</span>
                 <span v-else class="spin">⚙️</span>
            </div>
            <span class="btn-label">Revisar</span>
        </button>
        
        <button class="kids-big-btn btn-green" @click="uploadCode" :class="{ 'loading': isUploading }">
             <div class="btn-icon">
                 <span v-if="!isUploading">🚀</span>
                 <span v-else class="spin">🔥</span>
            </div>
            <span class="btn-label">¡Dar Vida!</span>
        </button>
      </div>
    </header>

    <main class="main-stage">
      <div class="blockly-frame">
          <div ref="blocklyDiv" class="blockly-canvas"></div>
      </div>

      <aside class="side-tools">
        
        <div class="code-preview-card">
            <div class="card-header" @click="showOutput = !showOutput">
                <span>💻 Modo Hacker</span>
                <span class="arrow">{{ showOutput ? '▼' : '▶' }}</span>
            </div>
            <div v-show="showOutput" class="code-content">
                 <textarea readonly :value="generatedCode"></textarea>
            </div>
        </div>

        <div class="robot-assistant" :class="assistantMood">
            <div class="robot-avatar">
                <span v-if="assistantMood === 'happy'">🤖👍</span>
                <span v-else-if="assistantMood === 'thinking'">🤔💭</span>
                <span v-else-if="assistantMood === 'error'">😵💥</span>
                <span v-else>🤖</span>
            </div>
            <div class="speech-bubble">
                <p>{{ assistantMessage }}</p>
            </div>
        </div>

        <div class="extra-actions-grid">
            <button class="kids-tool-btn btn-yellow" @click="saveSketch" title="Guardar">💾</button>
            <button class="kids-tool-btn btn-purple" @click="loadSketch" title="Abrir">📂</button>
            <button class="kids-tool-btn btn-orange" @click="openInArduino" title="IDE Pro">🛠️</button>
            <button class="kids-tool-btn btn-red" @click="installAvrCore" :disabled="isInstalling" title="Instalar Soporte">
                <span v-if="isInstalling" class="spin">⏳</span>
                <span v-else>📦</span>
            </button>
            <button class="kids-tool-btn btn-danger-zone" @click="clearWorkspace" title="Borrar Todo">🗑️</button>
        </div>

      </aside>
    </main>
  </div>
</template>

<script setup>
import { onMounted, ref, computed } from 'vue'; 
import * as Blockly from 'blockly';
import * as En from 'blockly/msg/en';
import 'blockly/blocks'; 

// Importación unificada (Bloques + Generador)
import ArduinoGenerator from '../arduino_core.js'; 

// Ajusta estas rutas a tus logos reales
import logo2 from '../../../../resources/logo_m4rk.webp'; 
import logo1 from '../../../../resources/logo_RM.png';

// --- NUEVO: SONIDOS (Ruta corregida: subimos 2 niveles para llegar a public) ---
import clickSound from '../../public/media/click.mp3';
import deleteSound from '../../public/media/delete.mp3';

Blockly.setLocale(En);

// Refs de UI
const blocklyDiv = ref(null);
const generatedCode = ref("");
const generatedXml = ref("");
// outputLog se mantiene internamente pero visualmente usamos el Asistente
const outputLog = ref("Bienvenido a MarkRobot IDE.\nSelecciona una placa y puerto para comenzar.");
const isCompiling = ref(false);
const isUploading = ref(false);
const isInstalling = ref(false);
const showOutput = ref(false); // Por defecto cerrado para niños
const sketchName = ref("MiInvento");

// Refs de Hardware
const rawPorts = ref([]); 
const selectedPort = ref("");
const allKnownBoards = ref([]); 
const selectedBoardFqbn = ref("arduino:avr:uno"); 

// Refs de "Gamificación" (Robot)
const assistantMessage = ref("¡Hola! Soy Mark. ¡Arrastra bloques para empezar!");
const assistantMood = ref("normal"); // normal, happy, thinking, error

// Audio Players
const audioClick = new Audio(clickSound);
const audioDelete = new Audio(deleteSound);

let workspace = null;

// --- COMPUTADA: FORMATO DE PUERTOS LIMPIO Y ROBUSTO ---
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
            label: portName, 
        };
    }).filter(p => p !== null); 
});

// Helper para controlar al robot
function setRobotStatus(msg, mood = 'normal') {
    assistantMessage.value = msg;
    assistantMood.value = mood;
    // También guardamos en el log técnico por si acaso
    outputLog.value += `\n[ROBOT]: ${msg}`;
}

async function refreshPorts() {
  if (!window.api) return;
  setRobotStatus("Buscando cables conectados...", "thinking");
  
  try {
    const ports = await window.api.listBoards(); 
    rawPorts.value = ports || [];
    
    setTimeout(() => {
        if (availablePorts.value.length > 0) {
            if (!selectedPort.value) {
                selectedPort.value = availablePorts.value[0].address;
            }
            setRobotStatus(`¡Encontré ${availablePorts.value.length} placa(s)! Listo.`, "happy");
        } else {
            setRobotStatus("No veo ninguna placa. ¿Conectaste el USB?", "normal");
        }
    }, 500);
    
  } catch (e) {
    setRobotStatus("Error al buscar puertos.", "error");
    outputLog.value += `\nError: ${e.message}`;
  }
}

async function verifyCode() {
  if (isCompiling.value) return;
  if (!window.api) { alert("Backend no conectado"); return; }

  isCompiling.value = true;
  setRobotStatus("Leyendo tus instrucciones...", "thinking");
  updateContent(); 

  try {
    const res = await window.api.compile({
      code: generatedCode.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    outputLog.value += res.log;
    
    if (res.success) {
        setRobotStatus("¡Código perfecto! Todo se ve bien.", "happy");
    } else {
        setRobotStatus("¡Ups! Algo está mal conectado en los bloques.", "error");
    }
  } catch (e) {
    setRobotStatus("Error grave del sistema.", "error");
    outputLog.value += "\nError crítico: " + e.message;
  } finally {
    isCompiling.value = false;
  }
}

async function uploadCode() {
  if (isUploading.value) return;
  if (!window.api) { alert("Backend no conectado"); return; }
  
  if (!selectedPort.value) {
    setRobotStatus("¡Espera! Necesito saber el PUERTO USB.", "error");
    alert("Por favor selecciona un PUERTO primero.");
    return;
  }
  
  isUploading.value = true;
  setRobotStatus("¡Enviando poderes a la placa! ⚡", "thinking");
  updateContent();
  
  try {
    const compileRes = await window.api.compile({
      code: generatedCode.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    
    if (!compileRes.success) {
      outputLog.value += compileRes.log;
      setRobotStatus("No puedo subirlo, hay errores en los bloques.", "error");
      isUploading.value = false;
      return;
    }

    setRobotStatus("Compilado OK. Subiendo...", "thinking");
    
    const uploadRes = await window.api.upload({
      port: selectedPort.value,
      fqbn: selectedBoardFqbn.value,
      sketchName: sketchName.value || 'Sketch'
    });
    
    outputLog.value += uploadRes.log;
    if (uploadRes.success) {
        setRobotStatus("¡ÉXITO! Tu robot tiene vida. 🚀", "happy");
    } else {
        setRobotStatus("Falló la subida. Revisa el cable.", "error");
    }
  } catch (e) {
    setRobotStatus("Error crítico en subida.", "error");
    outputLog.value += "\nError crítico: " + e.message;
  } finally {
    isUploading.value = false;
  }
}

async function installAvrCore() {
  if (isInstalling.value) return;
  if (!window.api) return;

  if(!confirm("Esto descargará el soporte para Arduino. ¿Continuar?")) return;

  isInstalling.value = true;
  setRobotStatus("Descargando herramientas...", "thinking");
  
  try {
    const res = await window.api.installCore('arduino:avr');
    outputLog.value += res.log;
    
    if (res.success) {
        setRobotStatus("¡Instalación completada!", "happy");
        window.api.listAllBoards().then(data => {
            if(data && data.boards) allKnownBoards.value = data.boards;
        });
    } else {
        setRobotStatus("Hubo un error instalando.", "error");
    }
  } catch (e) {
    outputLog.value += "\nError crítico: " + e.message;
  } finally {
    isInstalling.value = false;
  }
}

async function openInArduino() {
  setRobotStatus("Abriendo IDE avanzado...", "normal");
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
        setRobotStatus("¡Proyecto guardado!", "happy");
        outputLog.value += `\nGuardado en: ${result.path}`;
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
            setRobotStatus("¡Proyecto cargado!", "happy");
            if(result.fileName) {
                sketchName.value = result.fileName.replace(/\.[^/.]+$/, "");
            }
        } catch(e) {
            setRobotStatus("No pude leer ese archivo.", "error");
            console.error(e);
        }
    }
  }
}

function clearWorkspace() {
    audioClick.play();
    if(confirm("¿Estás seguro de borrar todos los bloques?")) {
        workspace.clear();
        audioDelete.play(); // Sonido extra
        insertStartBlock();
        setRobotStatus("Lienzo limpio. ¡A crear!", "normal");
    }
}

// =========================================================
// DEFINICIÓN DE TOOLBOX (MENÚ LATERAL)
// =========================================================
const toolbox = {
  kind: 'categoryToolbox',
  contents: [
    { kind: 'category', name: 'Variables', colour: '#FF6B6B', contents: [
      { kind: 'block', type: 'arduino_start' },
      { kind: 'button', text: 'Crear Variable', callbackKey: 'CREATE_VARIABLE' },
      { kind: 'block', type: 'variables_get' },
      { kind: 'block', type: 'variables_set' },
      { kind: 'block', type: 'variables_set_type' },
      { kind: 'block', type: 'type_cast' }
    ]},
    { kind: 'category', name: 'Decisiones', colour: '#FFD93D', contents: [
      { kind: 'block', type: 'controls_if' },
      { kind: 'block', type: 'logic_compare' },
      { kind: 'block', type: 'logic_operation' },
      { kind: 'block', type: 'logic_boolean' },
      { kind: 'block', type: 'logic_negate' },
      { kind: 'block', type: 'logic_null' },
      { kind: 'block', type: 'logic_ternary' }
    ]},
    { kind: 'category', name: 'Repetir', colour: '#4D96FF', contents: [
      { kind: 'block', type: 'controls_repeat_ext', inputs: { TIMES: { shadow: { type: 'math_number', fields: { NUM: 10 } } } } },
      { kind: 'block', type: 'controls_whileUntil' },
      { kind: 'block', type: 'controls_for', inputs: { FROM: { shadow: { type: 'math_number', fields: { NUM: 1 } } }, TO: { shadow: { type: 'math_number', fields: { NUM: 10 } } }, BY: { shadow: { type: 'math_number', fields: { NUM: 1 } } } } },
      { kind: 'block', type: 'controls_flow_statements' }
    ]},
    { kind: 'category', name: 'Números', colour: '#6BCB77', contents: [
      { kind: 'block', type: 'math_number' },
      { kind: 'block', type: 'math_arithmetic' },
      { kind: 'block', type: 'math_random_int' },
      { kind: 'block', type: 'math_map' },
      { kind: 'block', type: 'math_single' },
      { kind: 'block', type: 'math_trig' },
      { kind: 'block', type: 'math_constant' },
      { kind: 'block', type: 'math_number_property' },
      { kind: 'block', type: 'math_round' },
      { kind: 'block', type: 'math_modulo' },
      { kind: 'block', type: 'math_constrain' }
    ]},
    { kind: 'category', name: 'Texto', colour: '#A65C81', contents: [
      { kind: 'block', type: 'text' },
      { kind: 'block', type: 'text_print' },
      { kind: 'block', type: 'text_join' },
      { kind: 'block', type: 'text_append' },
      { kind: 'block', type: 'text_length' },
      { kind: 'block', type: 'text_isEmpty' }
    ]},
    { kind: 'sep' },
    { kind: 'category', name: 'Pines I/O', colour: '#5C81A6', contents: [
      { kind: 'block', type: 'digital_write' },
      { kind: 'block', type: 'digital_read' },
      { kind: 'block', type: 'analog_read' },
      { kind: 'block', type: 'analog_write' },
      { kind: 'block', type: 'custom_delay' }
    ]},
    { kind: 'category', name: 'Motores', colour: '#E67E22', contents: [
      { kind: 'block', type: 'motor_setup' },
      { kind: 'block', type: 'motor_run' },
      { kind: 'block', type: 'motor_stop' }
    ]},
    { kind: 'category', name: 'Pantalla 8x8', colour: '#D35400', contents: [
      { kind: 'block', type: 'display_8x8_setup' },
      { kind: 'block', type: 'display_8x8_draw' }
    ]},
    { kind: 'category', name: 'Sensores', colour: '#8E44AD', contents: [
      { kind: 'block', type: 'ultrasonic_read' },
      { kind: 'block', type: 'color_sensor_read' },
      { kind: 'block', type: 'sound_sensor_read' }
    ]},
    { kind: 'category', name: 'Wifi/BT', colour: '#2980B9', contents: [
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
  if (blocklyDiv.value) {
    workspace = Blockly.inject(blocklyDiv.value, {
      toolbox: toolbox,
      scrollbars: true,
      media: 'media/', 
      zoom: { controls: true, wheel: true, startScale: 1.0, maxScale: 3, minScale: 0.3, scaleSpeed: 1.2 },
      renderer: 'geras',
      theme: {
        'base': 'classic',
        'fontStyle': { 'family': 'Fredoka One, sans-serif', 'weight': 'normal', 'size': 12 },
        'componentStyles': { 
            'workspaceBackgroundColour': '#FFFFFF', 
            'toolboxBackgroundColour': '#FFF4E0', 
            'flyoutBackgroundColour': '#FFEacc',
            'flyoutOpacity': 1
        }
      }
    });

    workspace.registerButtonCallback('CREATE_VARIABLE', (button) => {
      Blockly.Variables.createVariableButtonHandler(button.getTargetWorkspace(), null, '');
    });
    
    workspace.addChangeListener(updateContent);
    // Sonido al mover bloques
    workspace.addChangeListener((e) => {
        if(e.type === Blockly.Events.BLOCK_MOVE && !e.oldParentId && e.newParentId) {
            audioClick.play();
        }
    });

    window.addEventListener('resize', () => Blockly.svgResize(workspace));
    
    insertStartBlock();

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
@import url('https://fonts.googleapis.com/css2?family=Fredoka+One&family=Nunito:wght@400;700;900&display=swap');

/* --- ESTILO CARTOON / KIDS --- */
:root {
    --bg-sunny: #FFFAE5;
    --primary-blue: #4D96FF;
    --primary-green: #6BCB77;
    --primary-red: #FF6B6B;
    --primary-yellow: #FFD93D;
    --border-dark: #2C3E50;
    --font-head: 'Fredoka One', cursive;
    --font-body: 'Nunito', sans-serif;
}

.kids-layout {
    display: flex;
    flex-direction: column;
    height: 100vh;
    background-color: var(--bg-sunny);
    font-family: var(--font-body);
    color: var(--border-dark);
    overflow: hidden;
}

/* TOP BAR */
.top-bar-kids {
    background: #ffffff;
    padding: 10px 20px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    border-bottom: 4px solid var(--border-dark);
    box-shadow: 0 6px 0 rgba(0,0,0,0.1);
    z-index: 100;
    gap: 20px;
}

.brand-area {
    display: flex;
    align-items: center;
    gap: 15px;
}

.bouncy { animation: bounce 2s infinite; }
@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-5px); }
}

.app-logo { height: 50px; }

.project-title-wrapper {
    display: flex;
    flex-direction: column;
}

.label-tiny {
    font-size: 0.7rem;
    font-weight: 900;
    color: #95a5a6;
    margin-left: 5px;
}

.kids-input-title {
    background: #F3F4F6;
    border: 3px solid var(--border-dark);
    border-radius: 12px;
    padding: 5px 10px;
    font-family: var(--font-head);
    font-size: 1.2rem;
    color: var(--primary-blue);
    width: 200px;
    transition: transform 0.2s;
}
.kids-input-title:focus {
    transform: scale(1.05);
    outline: none;
    border-color: var(--primary-blue);
}

/* HARDWARE CONTROLS */
.hardware-island {
    display: flex;
    gap: 15px;
    background: #EEE;
    padding: 8px 15px;
    border-radius: 15px;
    border: 3px solid #DDD;
}

.select-wrapper {
    position: relative;
    display: flex;
    align-items: center;
    gap: 5px;
}

.icon-label { font-size: 1.5rem; }

.kids-select {
    appearance: none;
    background: white;
    border: 3px solid var(--border-dark);
    border-radius: 10px;
    padding: 8px 30px 8px 10px;
    font-weight: 800;
    font-family: var(--font-body);
    cursor: pointer;
    box-shadow: 4px 4px 0 rgba(0,0,0,0.2);
}
.kids-select:active { box-shadow: 2px 2px 0 rgba(0,0,0,0.2); transform: translate(2px, 2px); }

.kids-btn-mini {
    background: var(--primary-yellow);
    border: 3px solid var(--border-dark);
    border-radius: 8px;
    cursor: pointer;
    font-size: 1.2rem;
    padding: 5px;
    box-shadow: 3px 3px 0 rgba(0,0,0,0.2);
    transition: all 0.2s;
}
.kids-btn-mini:active { transform: translate(3px, 3px); box-shadow: none; }
.rotate-hover:hover { transform: rotate(180deg); }

/* BIG ACTIONS */
.big-actions {
    display: flex;
    gap: 15px;
}

.kids-big-btn {
    border: 3px solid var(--border-dark);
    border-radius: 15px;
    padding: 10px 20px;
    font-family: var(--font-head);
    font-size: 1.1rem;
    color: white;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 10px;
    box-shadow: 6px 6px 0 rgba(0,0,0,0.2);
    transition: all 0.1s;
    text-transform: uppercase;
    letter-spacing: 1px;
}

.kids-big-btn:hover { transform: translateY(-2px); box-shadow: 8px 8px 0 rgba(0,0,0,0.2); }
.kids-big-btn:active { transform: translateY(4px); box-shadow: 0px 0px 0 rgba(0,0,0,0.2); }

.btn-blue { background: var(--primary-blue); }
.btn-green { background: var(--primary-green); }

.spin { animation: spin 1s linear infinite; display: inline-block; }
@keyframes spin { 100% { transform: rotate(360deg); } }

/* MAIN STAGE */
.main-stage {
    display: flex;
    flex: 1;
    padding: 20px;
    gap: 20px;
    overflow: hidden;
}

.blockly-frame {
    flex: 1;
    background: white;
    border: 4px solid var(--border-dark);
    border-radius: 20px;
    box-shadow: 10px 10px 0 rgba(0,0,0,0.1);
    overflow: hidden;
    position: relative;
}

.blockly-canvas { width: 100%; height: 100%; }

/* SIDEBAR TOOLS */
.side-tools {
    width: 300px;
    display: flex;
    flex-direction: column;
    gap: 20px;
}

/* ROBOT ASSISTANT */
.robot-assistant {
    background: white;
    border: 4px solid var(--border-dark);
    border-radius: 20px;
    padding: 15px;
    display: flex;
    flex-direction: column;
    align-items: center;
    box-shadow: 8px 8px 0 rgba(0,0,0,0.1);
    transition: all 0.3s;
}

.robot-assistant.error { background: #FFEEEE; border-color: var(--primary-red); }
.robot-assistant.happy { background: #EEFFEE; border-color: var(--primary-green); }

.robot-avatar { font-size: 4rem; animation: float 3s ease-in-out infinite; }
@keyframes float { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-10px); } }

.speech-bubble {
    background: #f0f0f0;
    border-radius: 15px;
    padding: 15px;
    margin-top: 10px;
    position: relative;
    border: 2px solid var(--border-dark);
    font-weight: bold;
    text-align: center;
    width: 100%;
}
.speech-bubble:after {
    content: ''; position: absolute; top: -10px; left: 50%;
    border-width: 0 10px 10px; border-style: solid;
    border-color: transparent transparent var(--border-dark) transparent;
}

/* EXTRA ACTIONS GRID */
.extra-actions-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
}

.kids-tool-btn {
    border: 3px solid var(--border-dark);
    border-radius: 12px;
    padding: 15px;
    font-size: 1.5rem;
    cursor: pointer;
    box-shadow: 4px 4px 0 rgba(0,0,0,0.2);
    transition: transform 0.1s;
    background: white;
}
.kids-tool-btn:active { transform: translate(4px, 4px); box-shadow: none; }

.btn-yellow { background: var(--primary-yellow); }
.btn-purple { background: #9B59B6; color: white; }
.btn-orange { background: #E67E22; color: white; }
.btn-red { background: #3498db; color: white; }
.btn-danger-zone { background: var(--primary-red); color: white; }

/* CODE PREVIEW */
.code-preview-card {
    background: #34495E;
    border-radius: 15px;
    overflow: hidden;
    border: 3px solid var(--border-dark);
    flex-shrink: 0;
}
.card-header {
    background: #2C3E50;
    color: white;
    padding: 10px;
    font-weight: bold;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
}
.code-content textarea {
    width: 100%;
    height: 150px;
    background: #34495E;
    color: #2ECC71;
    border: none;
    padding: 10px;
    font-family: 'Courier New', monospace;
    resize: none;
}
</style>
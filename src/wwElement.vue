<template>
  <div class="color-picker-wrapper">
    <div class="color-picker-trigger" @click="togglePicker">
      <div class="color-preview" :style="{ backgroundColor: color }"></div>
    </div>
    
    <!-- Overlay -->
    <div v-if="isOpen" class="overlay" @click="togglePicker"></div>
    
    <!-- Picker Container -->
    <div v-if="isOpen" class="picker-container" @click.stop @mousedown.stop>
      <div 
        class="saturation-panel" 
        ref="saturationPanel"
        :style="saturationPanelStyle"
        @mousedown.prevent.stop="startSaturationDrag"
        @touchstart.prevent.stop="startSaturationDrag"
      >
        <div class="saturation-gradient"></div>
        <div class="brightness-gradient"></div>
        <div class="saturation-cursor" :style="cursorPosition"></div>
      </div>
      
      <!-- Hue Slider -->
      <div 
        class="hue-slider" 
        ref="hueSlider"
        @mousedown.prevent.stop="startHueDrag"
        @touchstart.prevent.stop="startHueDrag"
      >
        <div class="hue-cursor" :style="{ left: huePosition }"></div>
      </div>
      
      <!-- Opacity Slider -->
      <div 
        class="opacity-slider" 
        ref="opacitySlider"
        @mousedown.prevent.stop="startOpacityDrag"
        @touchstart.prevent.stop="startOpacityDrag"
      >
        <div class="opacity-gradient" :style="opacityGradientStyle"></div>
        <div class="opacity-cursor" :style="{ left: opacityPosition }"></div>
      </div>
      
      <!-- Bottom Row -->
      <div class="bottom-row">
        <div class="format-selector-wrapper">
          <select 
            class="format-selector" 
            v-model="colorFormat"
            @click.stop
            @mousedown.stop
          >
            <option value="hex">HEX</option>
            <option value="rgb">RGB</option>
            <option value="hsl">HSL</option>
          </select>
          <span class="dropdown-arrow">▼</span>
        </div>
        
        <div class="color-input-wrapper">
          <span v-if="colorFormat === 'hex'" class="input-prefix">#</span>
          <input 
            class="color-input" 
            type="text"
            :value="formattedColorValue"
            @input="handleColorInput"
            @click.stop
            @mousedown.stop
            @focus="$event.target.select()"
          />
        </div>
        
        <input 
          class="opacity-input" 
          type="text"
          :value="opacityPercentage"
          @input="handleOpacityInput"
          @click.stop
          @mousedown.stop
          @focus="$event.target.select()"
        />
        
        <div class="color-preview-box" :style="{ '--preview-color': previewColor }"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { computed } from 'vue';

export default {
  props: {
    content: { type: Object, required: true },
    uid: { type: String, required: true },
  },
  setup(props) {
    const initValue = computed(() => props.content.initValue || '#1677ff');
    
    const { value: selectedColor, setValue: setSelectedColor } = wwLib.wwVariable.useComponentVariable({
      uid: props.uid,
      name: 'value',
      type: 'string',
      defaultValue: initValue,
    });
    
    return { selectedColor, setSelectedColor, initValue };
  },
  data() {
    return {
      isOpen: false,
      hue: 220,
      saturation: 100,
      brightness: 100,
      opacity: 1,
      colorFormat: 'hex',
      activeCleanup: null,
    };
  },
  watch: {
    initValue(newValue) {
      // When initValue changes, update selectedColor
      this.setSelectedColor(newValue);
    },
  },
  computed: {
    color() {
      return this.selectedColor || '#1677ff';
    },
    cursorPosition() {
      return {
        left: `${this.saturation}%`,
        top: `${100 - this.brightness}%`,
      };
    },
    huePosition() {
      return `${(this.hue / 360) * 100}%`;
    },
    opacityPosition() {
      return `${this.opacity * 100}%`;
    },
    saturationPanelStyle() {
      return {
        backgroundColor: `hsl(${this.hue}, 100%, 50%)`,
      };
    },
    opacityGradientStyle() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      return {
        background: `linear-gradient(to right, rgba(${r}, ${g}, ${b}, 0), rgba(${r}, ${g}, ${b}, 1))`,
      };
    },
    formattedColorValue() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      
      if (this.colorFormat === 'hex') {
        return this.hsbToHex(this.hue, this.saturation, this.brightness).substring(1).toUpperCase();
      } else if (this.colorFormat === 'rgb') {
        return `${r}, ${g}, ${b}`;
      } else if (this.colorFormat === 'hsl') {
        const h = Math.round(this.hue);
        const s = Math.round(this.saturation);
        const l = Math.round((this.brightness * (200 - this.saturation)) / 200);
        return `${h}, ${s}%, ${l}%`;
      }
      return '';
    },
    opacityPercentage() {
      return Math.round(this.opacity * 100) + '%';
    },
    previewColor() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      return `rgba(${r}, ${g}, ${b}, ${this.opacity})`;
    },
  },
  beforeUnmount() {
    if (this.activeCleanup) {
      this.activeCleanup();
      this.activeCleanup = null;
    }
  },
  methods: {
    togglePicker() {
      this.isOpen = !this.isOpen;
      console.log('🎨 Picker toggled:', this.isOpen ? 'OPEN' : 'CLOSED');
    },
    startSaturationDrag(e) {
      console.log('🎯 startSaturationDrag called', {
        type: e.type,
        button: e.button,
        touches: e.touches?.length,
        target: e.target.className
      });
      
      // CRITICAL: Only allow left mouse button (button 0) or touch
      if (e.button && e.button !== 0) {
        console.log('❌ Rejected: Not left mouse button', e.button);
        return;
      }
      if (e.touches && e.touches.length > 1) {
        console.log('❌ Rejected: Multi-touch detected', e.touches.length);
        return;
      }
      
      console.log('✅ Drag started');
      e.preventDefault();
      e.stopPropagation();
      
      // Force reflow
      void this.$refs.saturationPanel.offsetHeight;
      
      if (this.activeCleanup) {
        console.log('🧹 Cleaning up previous drag');
        this.activeCleanup();
      }
      
      // DO NOT update position on mousedown — only on move!
      
      let lastMoveTime = 0;
      let moveCount = 0;
      const throttleDelay = 16; // ~60fps, prevents wwLib re-render lag
      
      const onMove = (moveEvent) => {
        moveEvent.preventDefault(); // Critical: Stop browser defaults mid-drag
        const now = Date.now();
        if (now - lastMoveTime < throttleDelay) return;
        lastMoveTime = now;
        moveCount++;
        console.log(`🖱️ Move #${moveCount}`, {
          type: moveEvent.type,
          x: moveEvent.clientX,
          y: moveEvent.clientY
        });
        this.handleSaturationDrag(moveEvent);
        this.updateSelectedColor(); // Update color live during drag
      };
      
      const cleanup = () => {
        console.log('🛑 Drag ended', {
          totalMoves: moveCount,
          saturation: this.saturation.toFixed(1),
          brightness: this.brightness.toFixed(1)
        });
        
        console.log('🧹 Removing listeners...');
        
        // Remove all possible listeners
        window.removeEventListener('pointermove', onMove);
        window.removeEventListener('pointerup', cleanup);
        window.removeEventListener('pointercancel', cleanup);
        window.removeEventListener('mousemove', onMove);
        window.removeEventListener('mouseup', cleanup);
        document.removeEventListener('mousemove', onMove);
        document.removeEventListener('mouseup', cleanup);
        // Touch fallbacks
        window.removeEventListener('touchmove', onMove);
        window.removeEventListener('touchend', cleanup);
        
        if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
          const frontDoc = wwLib.getFrontDocument();
          frontDoc.removeEventListener('mousemove', onMove);
          frontDoc.removeEventListener('mouseup', cleanup);
        }
        
        console.log('✅ Listeners removed');
        
        this.activeCleanup = null;
        this.updateSelectedColor();
        console.log('💾 Color updated');
      };
      
      this.activeCleanup = cleanup;
      
      console.log('🎧 Adding listeners');
      
      // Hybrid: Pointer + Mouse + Touch (WeWeb iframe loves this combo)
      window.addEventListener('pointermove', onMove);
      window.addEventListener('pointerup', cleanup);
      window.addEventListener('pointercancel', cleanup);
      window.addEventListener('mousemove', onMove);
      window.addEventListener('mouseup', cleanup);
      // Touch: Passive false to allow preventDefault
      window.addEventListener('touchmove', onMove, { passive: false });
      window.addEventListener('touchend', cleanup);
      
      // WeWeb-specific: Use getFrontDocument if in editor mode
      if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
        const frontDoc = wwLib.getFrontDocument();
        frontDoc.addEventListener('mousemove', onMove);
        frontDoc.addEventListener('mouseup', cleanup);
      }
      
      console.log('✅ Listeners added');
    },
    handleSaturationDrag(e) {
      if (!this.$refs.saturationPanel) {
        console.warn('⚠️ saturationPanel ref not found');
        return;
      }
      
      // Support touch events
      const event = e.touches?.[0] || e;
      const rect = this.$refs.saturationPanel.getBoundingClientRect();
      
      const x = Math.max(0, Math.min(event.clientX - rect.left, rect.width));
      const y = Math.max(0, Math.min(event.clientY - rect.top, rect.height));
      
      this.saturation = (x / rect.width) * 100;
      this.brightness = 100 - (y / rect.height) * 100;
      
      console.log('📍 Position updated:', {
        s: this.saturation.toFixed(1) + '%',
        b: this.brightness.toFixed(1) + '%'
      });
    },
    startHueDrag(e) {
      // CRITICAL: Only allow left mouse button (button 0) or touch
      if (e.button && e.button !== 0) {
        console.log('❌ Rejected: Not left mouse button', e.button);
        return;
      }
      if (e.touches && e.touches.length > 1) {
        console.log('❌ Rejected: Multi-touch detected', e.touches.length);
        return;
      }
      
      console.log('✅ Hue drag started');
      e.stopPropagation();
      
      // Force layout reflow to fix Chrome drag bug
      void this.$refs.hueSlider.offsetHeight;
      
      if (this.activeCleanup) {
        console.log('🧹 Cleaning up previous drag');
        this.activeCleanup();
      }
      
      // DO NOT update position on mousedown — only on move!
      
      let lastMoveTime = 0;
      let moveCount = 0;
      const throttleDelay = 16; // ~60fps, prevents wwLib re-render lag
      
      const onMove = (moveEvent) => {
        moveEvent.preventDefault();
        const now = Date.now();
        if (now - lastMoveTime < throttleDelay) return;
        lastMoveTime = now;
        moveCount++;
        this.handleHueDrag(moveEvent);
        this.updateSelectedColor(); // Update color live during drag
      };
      
      const cleanup = () => {
        console.log('🛑 Hue drag ended', { totalMoves: moveCount, hue: this.hue.toFixed(1) });
        console.log('🧹 Removing listeners...');
        
        // Remove all possible listeners
        window.removeEventListener('pointermove', onMove);
        window.removeEventListener('pointerup', cleanup);
        window.removeEventListener('pointercancel', cleanup);
        window.removeEventListener('mousemove', onMove);
        window.removeEventListener('mouseup', cleanup);
        window.removeEventListener('touchmove', onMove);
        window.removeEventListener('touchend', cleanup);
        
        if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
          const frontDoc = wwLib.getFrontDocument();
          frontDoc.removeEventListener('mousemove', onMove);
          frontDoc.removeEventListener('mouseup', cleanup);
        }
        
        console.log('✅ Listeners removed');
        
        this.activeCleanup = null;
        this.updateSelectedColor();
        console.log('💾 Color updated');
      };
      
      this.activeCleanup = cleanup;
      
      console.log('🎧 Adding listeners');
      
      // Hybrid: Pointer + Mouse + Touch (WeWeb iframe loves this combo)
      window.addEventListener('pointermove', onMove);
      window.addEventListener('pointerup', cleanup);
      window.addEventListener('pointercancel', cleanup);
      window.addEventListener('mousemove', onMove);
      window.addEventListener('mouseup', cleanup);
      // Touch: Passive false to allow preventDefault
      window.addEventListener('touchmove', onMove, { passive: false });
      window.addEventListener('touchend', cleanup);
      
      // WeWeb-specific: Use getFrontDocument if in editor mode
      if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
        const frontDoc = wwLib.getFrontDocument();
        frontDoc.addEventListener('mousemove', onMove);
        frontDoc.addEventListener('mouseup', cleanup);
      }
      
      console.log('✅ Listeners added');
    },
    handleHueDrag(e) {
      if (!this.$refs.hueSlider) {
        console.warn('⚠️ hueSlider ref not found');
        return;
      }
      
      // Support touch events
      const event = e.touches?.[0] || e;
      const rect = this.$refs.hueSlider.getBoundingClientRect();
      
      const x = Math.max(0, Math.min(event.clientX - rect.left, rect.width));
      this.hue = (x / rect.width) * 360;
      
      console.log('🌈 Hue updated:', this.hue.toFixed(1) + '°');
    },
    startOpacityDrag(e) {
      // CRITICAL: Only allow left mouse button (button 0) or touch
      if (e.button && e.button !== 0) {
        console.log('❌ Rejected: Not left mouse button', e.button);
        return;
      }
      if (e.touches && e.touches.length > 1) {
        console.log('❌ Rejected: Multi-touch detected', e.touches.length);
        return;
      }
      
      console.log('✅ Opacity drag started');
      e.stopPropagation();
      
      // Force layout reflow to fix Chrome drag bug
      void this.$refs.opacitySlider.offsetHeight;
      
      if (this.activeCleanup) {
        console.log('🧹 Cleaning up previous drag');
        this.activeCleanup();
      }
      
      // DO NOT update position on mousedown — only on move!
      
      let lastMoveTime = 0;
      let moveCount = 0;
      const throttleDelay = 16; // ~60fps, prevents wwLib re-render lag
      
      const onMove = (moveEvent) => {
        moveEvent.preventDefault();
        const now = Date.now();
        if (now - lastMoveTime < throttleDelay) return;
        lastMoveTime = now;
        moveCount++;
        this.handleOpacityDrag(moveEvent);
        this.updateSelectedColor(); // Update color live during drag
      };
      
      const cleanup = () => {
        console.log('🛑 Opacity drag ended', { totalMoves: moveCount, opacity: this.opacity.toFixed(2) });
        console.log('🧹 Removing listeners...');
        
        // Remove all possible listeners
        window.removeEventListener('pointermove', onMove);
        window.removeEventListener('pointerup', cleanup);
        window.removeEventListener('pointercancel', cleanup);
        window.removeEventListener('mousemove', onMove);
        window.removeEventListener('mouseup', cleanup);
        window.removeEventListener('touchmove', onMove);
        window.removeEventListener('touchend', cleanup);
        
        if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
          const frontDoc = wwLib.getFrontDocument();
          frontDoc.removeEventListener('mousemove', onMove);
          frontDoc.removeEventListener('mouseup', cleanup);
        }
        
        console.log('✅ Listeners removed');
        
        this.activeCleanup = null;
        this.updateSelectedColor();
        console.log('💾 Color updated');
      };
      
      this.activeCleanup = cleanup;
      
      console.log('🎧 Adding listeners');
      
      // Hybrid: Pointer + Mouse + Touch (WeWeb iframe loves this combo)
      window.addEventListener('pointermove', onMove);
      window.addEventListener('pointerup', cleanup);
      window.addEventListener('pointercancel', cleanup);
      window.addEventListener('mousemove', onMove);
      window.addEventListener('mouseup', cleanup);
      // Touch: Passive false to allow preventDefault
      window.addEventListener('touchmove', onMove, { passive: false });
      window.addEventListener('touchend', cleanup);
      
      // WeWeb-specific: Use getFrontDocument if in editor mode
      if (typeof wwLib !== 'undefined' && wwLib.getFrontDocument) {
        const frontDoc = wwLib.getFrontDocument();
        frontDoc.addEventListener('mousemove', onMove);
        frontDoc.addEventListener('mouseup', cleanup);
      }
      
      console.log('✅ Listeners added');
    },
    handleOpacityDrag(e) {
      if (!this.$refs.opacitySlider) {
        console.warn('⚠️ opacitySlider ref not found');
        return;
      }
      
      // Support touch events
      const event = e.touches?.[0] || e;
      const rect = this.$refs.opacitySlider.getBoundingClientRect();
      
      const x = Math.max(0, Math.min(event.clientX - rect.left, rect.width));
      this.opacity = x / rect.width;
      
      console.log('💧 Opacity updated:', (this.opacity * 100).toFixed(0) + '%');
    },
    updateSelectedColor() {
      const hex = this.hsbToHex(this.hue, this.saturation, this.brightness);
      console.log('🎨 Setting color:', hex);
      this.setSelectedColor(hex);
    },
    handleColorInput(e) {
      const value = e.target.value.trim();
      
      if (this.colorFormat === 'hex') {
        // Parse hex color
        const hex = value.startsWith('#') ? value : `#${value}`;
        if (/^#[0-9A-Fa-f]{6}$/.test(hex)) {
          const hsb = this.hexToHsb(hex);
          this.hue = hsb.h;
          this.saturation = hsb.s;
          this.brightness = hsb.b;
          this.updateSelectedColor();
        }
      } else if (this.colorFormat === 'rgb') {
        // Parse RGB format: "r, g, b"
        const parts = value.split(',').map(p => parseInt(p.trim()));
        if (parts.length === 3 && parts.every(p => p >= 0 && p <= 255)) {
          const hsb = this.rgbToHsb(parts[0], parts[1], parts[2]);
          this.hue = hsb.h;
          this.saturation = hsb.s;
          this.brightness = hsb.b;
          this.updateSelectedColor();
        }
      } else if (this.colorFormat === 'hsl') {
        // Parse HSL format: "h, s%, l%"
        const parts = value.split(',').map(p => p.trim().replace('%', ''));
        if (parts.length === 3) {
          const h = parseFloat(parts[0]);
          const s = parseFloat(parts[1]);
          const l = parseFloat(parts[2]);
          if (!isNaN(h) && !isNaN(s) && !isNaN(l)) {
            const hsb = this.hslToHsb(h, s, l);
            this.hue = hsb.h;
            this.saturation = hsb.s;
            this.brightness = hsb.b;
            this.updateSelectedColor();
          }
        }
      }
    },
    handleOpacityInput(e) {
      const value = e.target.value.trim().replace('%', '');
      const opacity = parseFloat(value);
      if (!isNaN(opacity) && opacity >= 0 && opacity <= 100) {
        this.opacity = opacity / 100;
        this.updateSelectedColor();
      }
    },
    hsbToHex(h, s, b) {
      s = s / 100;
      b = b / 100;
      const k = (n) => (n + h / 60) % 6;
      const f = (n) => b * (1 - s * Math.max(0, Math.min(k(n), 4 - k(n), 1)));
      const r = Math.round(255 * f(5));
      const g = Math.round(255 * f(3));
      const bl = Math.round(255 * f(1));
      return `#${[r, g, bl].map((x) => x.toString(16).padStart(2, '0')).join('')}`;
    },
    hsbToRgb(h, s, b) {
      s = s / 100;
      b = b / 100;
      const k = (n) => (n + h / 60) % 6;
      const f = (n) => b * (1 - s * Math.max(0, Math.min(k(n), 4 - k(n), 1)));
      return {
        r: Math.round(255 * f(5)),
        g: Math.round(255 * f(3)),
        b: Math.round(255 * f(1)),
      };
    },
    hexToHsb(hex) {
      const r = parseInt(hex.slice(1, 3), 16) / 255;
      const g = parseInt(hex.slice(3, 5), 16) / 255;
      const b = parseInt(hex.slice(5, 7), 16) / 255;
      return this.rgbToHsb(r * 255, g * 255, b * 255);
    },
    rgbToHsb(r, g, b) {
      r /= 255;
      g /= 255;
      b /= 255;
      const max = Math.max(r, g, b);
      const min = Math.min(r, g, b);
      const delta = max - min;
      
      let h = 0;
      if (delta !== 0) {
        if (max === r) h = ((g - b) / delta + (g < b ? 6 : 0)) * 60;
        else if (max === g) h = ((b - r) / delta + 2) * 60;
        else h = ((r - g) / delta + 4) * 60;
      }
      
      const s = max === 0 ? 0 : (delta / max) * 100;
      const br = max * 100;
      
      return { h, s, b: br };
    },
    hslToHsb(h, s, l) {
      const brightness = l + (s * Math.min(l, 100 - l)) / 100;
      const saturation = brightness === 0 ? 0 : 2 * (1 - l / brightness) * 100;
      return { h, s: saturation, b: brightness };
    },
  },
};
</script>

<style scoped>
.color-picker-wrapper {
  position: relative;
  display: inline-block;
}

.color-picker-trigger {
  width: 32px;
  height: 32px;
  cursor: pointer;
  border: 2px solid #3a3a3a;
  background: #2a2a2a;
  border-radius: 4px;
  padding: 4px;
}

.color-preview {
  width: 100%;
  height: 100%;
  border-radius: 2px;
}

.overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: transparent;
  z-index: 98;
}

.picker-container {
  position: absolute;
  top: calc(100% + 8px);
  left: 0;
  width: 280px;
  padding: 12px;
  background: #1e1e1e;
  border: 1px solid #3a3a3a;
  border-radius: 8px;
  z-index: 99;
}

.saturation-panel {
  position: relative;
  width: 100%;
  height: 180px;
  cursor: crosshair;
  border-radius: 4px;
  user-select: none !important;
  -webkit-user-select: none !important;
  touch-action: none !important;
  overflow: hidden !important;
  transform: translateZ(0);
  -webkit-transform: translateZ(0);
}

.saturation-gradient {
  position: absolute;
  inset: 0;
  background: linear-gradient(to right, #fff, transparent);
  pointer-events: none !important;
  transform: translateZ(0);
}

.brightness-gradient {
  position: absolute;
  inset: 0;
  background: linear-gradient(to bottom, transparent, #000);
  pointer-events: none !important;
  transform: translateZ(0);
}

.saturation-cursor {
  position: absolute;
  width: 12px;
  height: 12px;
  border: 2px solid white;
  border-radius: 50%;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 3;
  will-change: transform;
  transform: translateZ(0) translate(-50%, -50%);
}

.hue-slider {
  position: relative;
  width: 100%;
  height: 12px;
  margin-top: 10px;
  border-radius: 6px;
  cursor: pointer;
  background: linear-gradient(to right, 
    #ff0000 0%, 
    #ffff00 17%, 
    #00ff00 33%, 
    #00ffff 50%, 
    #0000ff 67%, 
    #ff00ff 83%, 
    #ff0000 100%
  );
  user-select: none;
  touch-action: none;
  overflow: hidden;
  transform: translateZ(0);
}

.hue-cursor {
  position: absolute;
  top: 50%;
  width: 4px;
  height: 16px;
  background: white;
  border-radius: 2px;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
  transform: translate(-50%, -50%);
  pointer-events: none;
  will-change: transform;
  transform: translateZ(0) translate(-50%, -50%);
}

.opacity-slider {
  position: relative;
  width: 100%;
  height: 12px;
  margin-top: 10px;
  border-radius: 6px;
  cursor: pointer;
  background: 
    linear-gradient(45deg, #ccc 25%, transparent 25%), 
    linear-gradient(-45deg, #ccc 25%, transparent 25%), 
    linear-gradient(45deg, transparent 75%, #ccc 75%), 
    linear-gradient(-45deg, transparent 75%, #ccc 75%);
  background-size: 8px 8px;
  background-position: 0 0, 0 4px, 4px -4px, -4px 0px;
  user-select: none;
  touch-action: none;
  overflow: hidden;
  transform: translateZ(0);
}

.opacity-gradient {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border-radius: 6px;
  pointer-events: none;
  will-change: background;
  transform: translateZ(0);
}

.opacity-cursor {
  position: absolute;
  top: 50%;
  width: 4px;
  height: 16px;
  background: white;
  border-radius: 2px;
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.5);
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 1;
  will-change: transform;
  transform: translateZ(0) translate(-50%, -50%);
}

.bottom-row {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 12px;
}

.format-selector-wrapper {
  position: relative;
  display: inline-block;
}

.format-selector {
  padding: 6px 24px 6px 8px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  border-radius: 4px;
  color: #fff;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  outline: none;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
}

.format-selector:hover {
  background: #333;
}

.format-selector:focus {
  border-color: #555;
}

.dropdown-arrow {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 8px;
  color: #888;
  pointer-events: none;
}

.color-input-wrapper {
  position: relative;
  flex: 1;
  display: flex;
  align-items: center;
}

.input-prefix {
  position: absolute;
  left: 8px;
  color: #888;
  font-size: 11px;
  font-family: monospace;
  pointer-events: none;
}

.color-input {
  width: 100%;
  padding: 6px 8px;
  padding-left: 20px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  border-radius: 4px;
  color: #fff;
  font-size: 11px;
  font-family: monospace;
  outline: none;
  text-transform: uppercase;
}

.color-input:hover {
  background: #333;
}

.color-input:focus {
  border-color: #555;
  background: #333;
}

.opacity-input {
  width: 48px;
  padding: 6px 8px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  border-radius: 4px;
  color: #fff;
  font-size: 11px;
  font-family: monospace;
  text-align: center;
  outline: none;
}

.opacity-input:hover {
  background: #333;
}

.opacity-input:focus {
  border-color: #555;
  background: #333;
}

.color-preview-box {
  width: 32px;
  height: 32px;
  border-radius: 4px;
  border: 1px solid #3a3a3a;
  flex-shrink: 0;
  background: 
    linear-gradient(45deg, #555 25%, transparent 25%), 
    linear-gradient(-45deg, #555 25%, transparent 25%), 
    linear-gradient(45deg, transparent 75%, #555 75%), 
    linear-gradient(-45deg, transparent 75%, #555 75%);
  background-size: 8px 8px;
  background-position: 0 0, 0 4px, 4px -4px, -4px 0px;
  position: relative;
}

.color-preview-box::after {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: 4px;
  background: var(--preview-color);
}
</style>

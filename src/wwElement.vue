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
    opacityGradientStyle() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      return {
        background: `linear-gradient(to right, rgba(${r}, ${g}, ${b}, 0), rgba(${r}, ${g}, ${b}, 1))`,
      };
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
  background: hsl(220, 100%, 50%);
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
</style>

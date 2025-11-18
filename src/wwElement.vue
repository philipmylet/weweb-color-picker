<template>
  <div class="color-picker-wrapper" :style="wrapperStyle">
    <div class="color-picker-trigger" @click="togglePicker" :style="triggerStyle">
      <div class="color-preview" :style="{ backgroundColor: selectedColor }"></div>
    </div>
    
    <div v-if="isOpen" class="color-picker-panel" :style="panelStyle">
      <!-- Saturation/Brightness Gradient -->
      <div class="saturation-panel" 
           ref="saturationPanel"
           @mousedown="startSaturationDrag"
           :style="saturationPanelStyle">
        <div class="saturation-gradient"></div>
        <div class="brightness-gradient"></div>
        <div class="saturation-cursor" :style="cursorPosition"></div>
      </div>
      
      <!-- Hue Slider -->
      <div class="hue-slider" 
           ref="hueSlider"
           @mousedown="startHueDrag"
           :style="sliderStyle">
        <div class="hue-cursor" :style="{ left: huePosition }"></div>
      </div>
      
      <!-- Opacity Slider -->
      <div v-if="content.showOpacity" 
           class="opacity-slider" 
           ref="opacitySlider"
           @mousedown="startOpacityDrag"
           :style="sliderStyle">
        <div class="opacity-gradient" :style="opacityGradientStyle"></div>
        <div class="opacity-cursor" :style="{ left: opacityPosition }"></div>
      </div>
      
      <!-- Color Info -->
      <div class="color-info" :style="infoStyle">
        <select v-model="colorFormat" class="format-select">
          <option value="hex">HEX</option>
          <option value="rgb">RGB</option>
          <option value="hsl">HSL</option>
        </select>
        <input 
          type="text" 
          v-model="colorInput" 
          @input="handleColorInput"
          class="color-input"
        />
        <div class="opacity-display" v-if="content.showOpacity">
          {{ Math.round(opacity * 100) }}%
        </div>
        <div class="color-swatch" :style="{ backgroundColor: selectedColor }"></div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ColorPickerElement',
  props: {
    content: { type: Object, required: true },
  },
  emits: ['update:content', 'trigger-event'],
  data() {
    return {
      isOpen: false,
      hue: 220,
      saturation: 100,
      brightness: 100,
      opacity: 1,
      colorFormat: 'hex',
      isDragging: false,
      dragType: null,
      activeCleanup: null,
    };
  },
  computed: {
    wrapperStyle() {
      return {
        width: this.content.width || 'auto',
        height: this.content.height || 'auto',
      };
    },
    triggerStyle() {
      return {
        width: this.content.triggerWidth || '32px',
        height: this.content.triggerHeight || '32px',
        borderRadius: this.content.triggerBorderRadius || '4px',
        padding: this.content.triggerPadding || '4px',
      };
    },
    panelStyle() {
      return {
        width: this.content.panelWidth || '280px',
        padding: this.content.panelPadding || '12px',
        borderRadius: this.content.panelBorderRadius || '8px',
      };
    },
    saturationPanelStyle() {
      return {
        height: this.content.saturationHeight || '180px',
        borderRadius: this.content.saturationBorderRadius || '4px',
        background: `hsl(${this.hue}, 100%, 50%)`,
      };
    },
    sliderStyle() {
      return {
        height: this.content.sliderHeight || '12px',
        borderRadius: this.content.sliderBorderRadius || '6px',
      };
    },
    infoStyle() {
      return {
        padding: this.content.infoPadding || '8px 0 0 0',
      };
    },
    selectedColor() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      return `rgba(${r}, ${g}, ${b}, ${this.opacity})`;
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
    colorInput: {
      get() {
        const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
        if (this.colorFormat === 'hex') {
          return this.rgbToHex(r, g, b);
        } else if (this.colorFormat === 'rgb') {
          return `${r}, ${g}, ${b}`;
        } else if (this.colorFormat === 'hsl') {
          const { h, s, l } = this.rgbToHsl(r, g, b);
          return `${h}, ${s}%, ${l}%`;
        }
        return '';
      },
      set(value) {
        // Handle input changes
      },
    },
  },
  mounted() {
    this.initializeFromDefaultColor();
  },
  beforeUnmount() {
    // Clean up any active drag listeners
    if (this.activeCleanup) {
      this.activeCleanup();
      this.activeCleanup = null;
    }
  },
  methods: {
    initializeFromDefaultColor() {
      if (this.content.defaultColor) {
        const color = this.parseColor(this.content.defaultColor);
        if (color) {
          const hsb = this.rgbToHsb(color.r, color.g, color.b);
          this.hue = hsb.h;
          this.saturation = hsb.s;
          this.brightness = hsb.b;
          this.opacity = color.a !== undefined ? color.a : 1;
        }
      }
    },
    togglePicker() {
      this.isOpen = !this.isOpen;
      if (!this.isOpen) {
        this.emitChange();
      }
    },
    startSaturationDrag(e) {
      e.preventDefault();
      
      // Clean up any existing drag operation
      if (this.activeCleanup) {
        this.activeCleanup();
      }
      
      this.isDragging = true;
      this.dragType = 'saturation';
      this.handleSaturationDrag(e);
      
      const onMouseMove = (moveEvent) => {
        moveEvent.preventDefault();
        this.handleSaturationDrag(moveEvent);
      };
      
      const cleanup = () => {
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', cleanup);
        this.isDragging = false;
        this.dragType = null;
        this.activeCleanup = null;
        this.emitChange();
      };
      
      this.activeCleanup = cleanup;
      document.addEventListener('mousemove', onMouseMove);
      document.addEventListener('mouseup', cleanup);
    },
    startHueDrag(e) {
      e.preventDefault();
      
      // Clean up any existing drag operation
      if (this.activeCleanup) {
        this.activeCleanup();
      }
      
      this.isDragging = true;
      this.dragType = 'hue';
      this.handleHueDrag(e);
      
      const onMouseMove = (moveEvent) => {
        moveEvent.preventDefault();
        this.handleHueDrag(moveEvent);
      };
      
      const cleanup = () => {
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', cleanup);
        this.isDragging = false;
        this.dragType = null;
        this.activeCleanup = null;
        this.emitChange();
      };
      
      this.activeCleanup = cleanup;
      document.addEventListener('mousemove', onMouseMove);
      document.addEventListener('mouseup', cleanup);
    },
    startOpacityDrag(e) {
      e.preventDefault();
      
      // Clean up any existing drag operation
      if (this.activeCleanup) {
        this.activeCleanup();
      }
      
      this.isDragging = true;
      this.dragType = 'opacity';
      this.handleOpacityDrag(e);
      
      const onMouseMove = (moveEvent) => {
        moveEvent.preventDefault();
        this.handleOpacityDrag(moveEvent);
      };
      
      const cleanup = () => {
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', cleanup);
        this.isDragging = false;
        this.dragType = null;
        this.activeCleanup = null;
        this.emitChange();
      };
      
      this.activeCleanup = cleanup;
      document.addEventListener('mousemove', onMouseMove);
      document.addEventListener('mouseup', cleanup);
    },
    handleSaturationDrag(e) {
      if (!this.$refs.saturationPanel) return;
      const rect = this.$refs.saturationPanel.getBoundingClientRect();
      const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
      const y = Math.max(0, Math.min(e.clientY - rect.top, rect.height));
      
      this.saturation = (x / rect.width) * 100;
      this.brightness = 100 - (y / rect.height) * 100;
    },
    handleHueDrag(e) {
      if (!this.$refs.hueSlider) return;
      const rect = this.$refs.hueSlider.getBoundingClientRect();
      const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
      this.hue = (x / rect.width) * 360;
    },
    handleOpacityDrag(e) {
      if (!this.$refs.opacitySlider) return;
      const rect = this.$refs.opacitySlider.getBoundingClientRect();
      const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
      this.opacity = x / rect.width;
    },
    handleColorInput(e) {
      const value = e.target.value;
      const color = this.parseColor(value);
      if (color) {
        const hsb = this.rgbToHsb(color.r, color.g, color.b);
        this.hue = hsb.h;
        this.saturation = hsb.s;
        this.brightness = hsb.b;
        if (color.a !== undefined) {
          this.opacity = color.a;
        }
      }
    },
    emitChange() {
      const { r, g, b } = this.hsbToRgb(this.hue, this.saturation, this.brightness);
      const hex = this.rgbToHex(r, g, b);
      const rgba = `rgba(${r}, ${g}, ${b}, ${this.opacity})`;
      const { h, s, l } = this.rgbToHsl(r, g, b);
      
      this.$emit('update:content', {
        ...this.content,
        selectedColor: rgba,
      });
      
      this.$emit('trigger-event', {
        name: 'change',
        event: {
          hex,
          rgb: `rgb(${r}, ${g}, ${b})`,
          rgba,
          hsl: `hsl(${h}, ${s}%, ${l}%)`,
          hsla: `hsla(${h}, ${s}%, ${l}%, ${this.opacity})`,
        },
      });
    },
    parseColor(color) {
      if (color.startsWith('#')) {
        const hex = color.slice(1);
        if (hex.length === 6) {
          return {
            r: parseInt(hex.slice(0, 2), 16),
            g: parseInt(hex.slice(2, 4), 16),
            b: parseInt(hex.slice(4, 6), 16),
            a: 1,
          };
        }
      }
      const rgbMatch = color.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)(?:,\s*([\d.]+))?\)/);
      if (rgbMatch) {
        return {
          r: parseInt(rgbMatch[1]),
          g: parseInt(rgbMatch[2]),
          b: parseInt(rgbMatch[3]),
          a: rgbMatch[4] ? parseFloat(rgbMatch[4]) : 1,
        };
      }
      return null;
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
    rgbToHsb(r, g, b) {
      r /= 255;
      g /= 255;
      b /= 255;
      const max = Math.max(r, g, b);
      const min = Math.min(r, g, b);
      const d = max - min;
      const s = max === 0 ? 0 : d / max;
      const v = max;
      let h = 0;
      
      if (max !== min) {
        switch (max) {
          case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break;
          case g: h = ((b - r) / d + 2) / 6; break;
          case b: h = ((r - g) / d + 4) / 6; break;
        }
      }
      
      return {
        h: Math.round(h * 360),
        s: Math.round(s * 100),
        b: Math.round(v * 100),
      };
    },
    rgbToHsl(r, g, b) {
      r /= 255;
      g /= 255;
      b /= 255;
      const max = Math.max(r, g, b);
      const min = Math.min(r, g, b);
      let h, s, l = (max + min) / 2;
      
      if (max === min) {
        h = s = 0;
      } else {
        const d = max - min;
        s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
        switch (max) {
          case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break;
          case g: h = ((b - r) / d + 2) / 6; break;
          case b: h = ((r - g) / d + 4) / 6; break;
        }
      }
      
      return {
        h: Math.round(h * 360),
        s: Math.round(s * 100),
        l: Math.round(l * 100),
      };
    },
    rgbToHex(r, g, b) {
      return '#' + [r, g, b].map(x => {
        const hex = x.toString(16);
        return hex.length === 1 ? '0' + hex : hex;
      }).join('');
    },
  },
};
</script>

<style lang="scss" scoped>
.color-picker-wrapper {
  position: relative;
  display: inline-block;
}

.color-picker-trigger {
  cursor: pointer;
  border: 2px solid #3a3a3a;
  background: #2a2a2a;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.2s;
  
  &:hover {
    border-color: #4a4a4a;
  }
}

.color-preview {
  width: 100%;
  height: 100%;
  border-radius: 2px;
}

.color-picker-panel {
  position: absolute;
  top: calc(100% + 8px);
  left: 0;
  background: #1e1e1e;
  border: 1px solid #3a3a3a;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
  z-index: 1000;
}

.saturation-panel {
  position: relative;
  cursor: crosshair;
  margin-bottom: 12px;
  border: 1px solid #3a3a3a;
  user-select: none;
}

.saturation-gradient {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(to right, #fff, transparent);
}

.brightness-gradient {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(to bottom, transparent, #000);
}

.saturation-cursor {
  position: absolute;
  width: 12px;
  height: 12px;
  border: 2px solid #fff;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
  box-shadow: 0 0 0 1px rgba(0, 0, 0, 0.3);
}

.hue-slider,
.opacity-slider {
  position: relative;
  cursor: pointer;
  margin-bottom: 12px;
  border: 1px solid #3a3a3a;
  user-select: none;
}

.hue-slider {
  background: linear-gradient(to right, 
    #ff0000 0%, 
    #ffff00 17%, 
    #00ff00 33%, 
    #00ffff 50%, 
    #0000ff 67%, 
    #ff00ff 83%, 
    #ff0000 100%
  );
}

.opacity-slider {
  background-image: 
    linear-gradient(45deg, #808080 25%, transparent 25%),
    linear-gradient(-45deg, #808080 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #808080 75%),
    linear-gradient(-45deg, transparent 75%, #808080 75%);
  background-size: 8px 8px;
  background-position: 0 0, 0 4px, 4px -4px, -4px 0px;
}

.opacity-gradient {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.hue-cursor,
.opacity-cursor {
  position: absolute;
  top: 50%;
  width: 4px;
  height: calc(100% + 4px);
  background: #fff;
  border: 1px solid #000;
  transform: translate(-50%, -50%);
  pointer-events: none;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.5);
}

.color-info {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  box-sizing: border-box;
}

.format-select {
  background: #2a2a2a;
  color: #fff;
  border: 1px solid #3a3a3a;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  cursor: pointer;
  flex-shrink: 0;
  
  &:focus {
    outline: none;
    border-color: #4a9eff;
  }
}

.color-input {
  flex: 1;
  min-width: 0;
  background: #2a2a2a;
  color: #fff;
  border: 1px solid #3a3a3a;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-family: monospace;
  
  &:focus {
    outline: none;
    border-color: #4a9eff;
  }
}

.opacity-display {
  color: #999;
  font-size: 12px;
  min-width: 35px;
  text-align: right;
  flex-shrink: 0;
}

.color-swatch {
  width: 24px;
  height: 24px;
  border-radius: 4px;
  border: 1px solid #3a3a3a;
  flex-shrink: 0;
}
</style>

<!-- ColorPicker - Color picker used when editing widget properties.
     Can show material-design colors, 20-distinct colors, and arbitrary colors.
     Copyright ©2021 Thorsten von Eicken, MIT license, see LICENSE file
-->

<template>
  <div style="display: content;">
    <v-text-field dense :label="label" :hint="hint" :value="value"
                  @input="$emit('input',$event.hex)">
      <template v-slot:append-outer>
        <v-btn icon x-small @click="show_picker=!show_picker">
          <v-icon>mdi-palette</v-icon>
        </v-btn>
      </template>
    </v-text-field>
    <v-overlay v-model="show_picker" class="color-picker" :dark="false">
      <v-card class="d-flex flex-column">
        <v-card-title class="d-flex align-baseline width100 pt-0 pb-0">
          <span>{{label}}</span>
          <v-spacer></v-spacer>
          <v-btn elevation=0 icon @click="show_picker=false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        <v-color-picker hide-canvas hide-sliders mode="hexa" show-swatches swatches-max-height="auto"
                        width="100%"
                        :swatches="swatches" :value="value" @update:color="changeColor">
        </v-color-picker>
      </v-card>
    </v-overlay>
  </div>
</template>

<style>
.color-picker .v-overlay__content { height: 95%; width: 95%; max-width: 500px;  }
.color-picker .v-card { height: 100%; width: 100%; }
.color-picker .v-color-picker {
  flex-grow: 1;  min-height: 0px;
  display: flex; flex-direction: column;
}
.color-picker .v-color-picker__edit { margin-top: 0px; }
.color-picker .v-color-picker__controls { padding-bottom: 0px; }
.color-picker .v-color-picker__input > input { margin-bottom: 0px; }
.color-picker .v-color-picker__swatches { min-height: 0px; max-height: 80% !important; }
</style>

<script scoped>
import md_colors from 'vuetify/es5/util/colors.js'

export default {
  name: "ColorPicker",

  props: {
    label: { default: null },
    hint: { default: null },
    value: { default: null },
  },

  data() {
    const swatches = []
    for (let c in md_colors) {
      const color = md_colors[c]
      if (c == 'shades') {
        swatches.push( Object.values(color) )
      } else {
        swatches.push( ['lighten3', 'base', 'darken3'].map(l => color[l]) )
      }
    }
    swatches[swatches.length-1][2] = '#00000000' // transparent -> text color

    return {
      show_picker: false,
      swatches,
    }
  },

  methods: {
    changeColor(c) {
      if (c && c.hexa == "#00000000") {
        this.$emit('input', "") // gives us text color with theme support
      } else {
        this.$emit('input', c.hex)
      }
    },
  },

}
</script>

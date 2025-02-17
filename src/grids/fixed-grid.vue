<!-- Grid - Container to manage a grid of widgets dynamically placed on the page.
     Handles the layout of the widgets as well as the editing mode.
     Copyright ©2021 Thorsten von Eicken, MIT license, see LICENSE file
-->

<template>
  <div class="u-tooltip-attach">

    <!-- Hacky roll-up/roll-down icon at the top-center of the grid if there's no title -->
    <div v-if="rollupMini" :class="rollerClasses">
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-btn x-small icon height="24px" class="mx-auto" @click="toggleRoll" v-on="on">
            <v-icon>mdi-arrow-{{rolledup ? 'down' : 'up'}}-drop-circle</v-icon>
          </v-btn>
        </template>
        <span>Roll widgets up/down</span>
      </v-tooltip>
    </div>

    <!-- Normal-mode title bar, when we have a title -->
    <v-toolbar dense flat v-if="rollupMaxi" height=36 color="background"
               class="d-flex justify-start">
      <!-- roll-up/down button -->
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-btn x-small icon height="24px" class="mx-auto" @click="toggleRoll" v-on="on">
            <v-icon>mdi-arrow-{{rolledup ? 'down' : 'up'}}-drop-circle</v-icon>
          </v-btn>
        </template>
        <span>Roll widgets up/down</span>
      </v-tooltip>
      <!-- grid title -->
      <v-toolbar-title>{{grid.title}}</v-toolbar-title>
    </v-toolbar>

    <!-- Editing toolbar above grid proper -->
    <v-toolbar v-if="$root.editMode" dense flat color="background" class="editmode">
      <!-- roll-up/down button -->
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-btn small icon color="grey" @click="toggleRoll" class="mr-4" v-on="on">
            <v-icon>mdi-arrow-{{rolledup ? 'down' : 'up'}}-drop-circle</v-icon>
          </v-btn>
        </template>
        <span>Roll widgets up/down</span>
      </v-tooltip>

      <!-- grid title text field -->
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-text-field single-line dense hide-details label="grid title" class="mr-6 flex-grow-0"
                        v-on="on" :value="grid.title" @change="changeTitle" style="width: 20ex">
          </v-text-field>
        </template>
        <span>Title to show at top of grid, if empty the grid bar is thinner</span>
      </v-tooltip>

      <!-- Menu to add widget -->
      <widget-menu @select="addWidget"></widget-menu>

      <!-- Paste button/text field -->
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-btn small icon @click="pasting=!pasting" class="mc-auto" v-on="on">
            <v-icon>mdi-content-paste</v-icon>
          </v-btn>
        </template>
        <span>Paste a widget, adding it to the grid</span>
      </v-tooltip>
      <div ref="pasteDiv" class="d-flex">
        <input type="text" v-if="pasting" size="15"
               placeholder="paste widget here" class="pasteinput">
      </div>

      <v-spacer></v-spacer>
      <!-- Button to delete the grid -->
      <v-tooltip bottom>
        <template v-slot:activator="{ on }">
          <v-btn small @click="$emit('delete')" class="mc-auto" v-on="on">
            Delete grid
          </v-btn>
        </template>
        <span>Delete this grid and all its widgets</span>
      </v-tooltip>

    </v-toolbar>

    <!-- Grid of widgets -->
    <v-container fluid v-if="!rolledup" class="g-grid-small pt-0 px-2">
      <component v-for="(w,ix) in grid.widgets" :key="w" :id="w" :is="editComponent[w]"
                 :edit_active="ix == edit_ix" @edit="toggleEdit(ix, $event)"
                 @move="moveWidget(ix, $event)" @delete="deleteWidget(ix)"
                 @clone="cloneWidget(ix)">
      </component>
    </v-container>
  </div>
</template>

<style scoped>
/* style for button groups in the edit toolbar */
.v-toolbar__content div { margin-right: 4ex; }

/* style for roll-up/roll-down bar when there's no toolbar */
.roller { width: 100% }
.roller.roller__minimal { height: 22px; }

/* style to make grid happen */
.g-grid-small {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  grid-auto-rows: 78px;
  gap: 8px;
  grid-auto-flow: dense;
}
.g-grid-margin { margin: 0.5em; }

.pasteinput {
  margin: 0 4px; padding: 0 2px; min-width: 20ex;
  color: #888;
  border: 1px solid #888; border-radius: 4px;
}
</style>
<style>
.editmode.theme--light .v-toolbar__content { border-top: 1px solid #e0e0e0; }
.editmode.theme--dark .v-toolbar__content { border-top: 1px solid #111; }
</style>

<script scoped>

import PanelEdit from '/src/components/panel-edit.vue'
import WidgetEdit from '/src/components/widget-edit.vue'
import WidgetMenu from '/src/components/widget-menu.vue'

export default {
  name: 'FixedGrid',

  components: { PanelEdit, WidgetEdit, WidgetMenu },
  inject: [ '$store', '$config', 'palette' ],

  props: {
    id: { type: String }, // this grid's ID
  },

  data() { return {
    edit_ix: null, // widget being edited
    rolledup: false, // whether grid is rolled-up
    pasting: false, // controls display of paste div
  }},

  computed: {
    // grid config: {id, kind, icon, widgets}
    grid() { return this.$store.gridByID(this.id) },
    rollupMini() { return !this.$root.editMode && !this.grid.title },
    rollupMaxi() { return !this.$root.editMode &&  this.grid.title },
    rollerClasses() { // classes for mini roll-up div
      const rm = this.grid.widgets.length>0 &&  !this.rolledup && 'roller__minimal'
      return [ 'd-flex', 'roller', rm ]
    },

    // editComponent returns the component used to edit a widget: widget-edit or panel-edit
    editComponent() {
      return Object.fromEntries(this.grid.widgets.map(wid =>
        [ wid, this.$store.widgetByID(wid).kind.endsWith("Panel") ? "PanelEdit" : "WidgetEdit" ]
      ))
    },
  },

  watch: {
    pasting(nv) {
      if (nv) {
        this.$refs.pasteDiv.addEventListener('paste', this.pasteWidget)
        this.$nextTick(()=>this.$refs.pasteDiv.firstChild.focus())
      } else {
        this.$refs.pasteDiv.removeEventListener('paste', this.pasteWidget)
      }
    },
  },

  methods: {
    toggleEdit(ix, on) { this.edit_ix = on ? ix : null },

    addWidget(kind) {
      const widget_ix = this.$store.addWidget(this.id, kind)
      this.edit_ix = widget_ix // start editing the new widget
    },

    // handle widget delete event coming up from widget-edit
    deleteWidget(ix) {
      this.$store.deleteWidget(this.id, ix)
      this.edit_ix = null
    },

    // handle widget clone event coming up from widget-edit
    cloneWidget(ix) {
      // start by adding a new widget of the same kind to the end of the grid
      const old_w = this.$store.widgetByID(this.$store.widgetIDByIX(this.grid, ix))
      const widget_ix = this.$store.addWidget(this.id, old_w.kind)
      const widget_id = this.$store.widgetIDByIX(this.grid, widget_ix)
      // copy the properties over
      const props = JSON.parse(JSON.stringify(old_w)) // clone and clean of observers
      delete props.id
      this.$store.updateWidget(widget_id, props)
      // move clone up to be just behind original
      if (widget_ix != ix+1) {
        let ww = [ ...this.grid.widgets ] // clone
        ww.copyWithin(ix+2, ix+1) // shift widgets up
        ww[ix+1] = widget_id
        this.$store.updateGrid(this.id, { widgets: ww })
      }
      this.edit_ix = ix+1
    },

    // move a widget up/down (dir=-1/1)
    moveWidget(ix, dir) {
      console.log(`Moving widget #${ix} by ${dir}`)
      if (!(ix+dir >= 0 && ix+dir < this.grid.widgets.length)) return
      let ww = [ ...this.grid.widgets ] // clone
      let w = ww[ix]; ww[ix] = ww[ix+dir]; ww[ix+dir] = w // swap
      this.$store.updateGrid(this.id, { widgets: ww })
      this.edit_ix += dir
    },

    // paste a widget
    pasteWidget(ev) {
      // Stop data actually being pasted into div
      ev.stopPropagation()
      ev.preventDefault()
      // Get pasted data via clipboard API
      let clipboardData = ev.clipboardData || window.clipboardData
      let pastedData = clipboardData.getData('Text')
      this.pasting = false
      // Validate pasted text
      console.log(pastedData)
      try {
        let w = JSON.parse(pastedData)
        if ('id' in w && 'kind' in w) {
          if (w.kind in this.palette.widgets) {
            const widget_ix = this.$store.addWidget(this.id, w.kind)
            delete w.id
            delete w.kind
            this.$store.updateWidget(this.$store.widgetIDByIX(this.grid, widget_ix), w)
          } else {
            console.log(`Widget kind '${w.kind}' not found`)
          }
        }
      } catch(e) {
        console.log(e)
      }
    },
    clearPasteDiv() { this.$refs.pasteDiv.firstChild.innerHTML = "" },

    toggleRoll() { this.rolledup = !this.rolledup },
    changeTitle(ev) { this.$store.updateGrid(this.id, { title: ev }) },

  },
}

</script>

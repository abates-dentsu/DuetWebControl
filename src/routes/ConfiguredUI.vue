<template>
  <v-card>
    <v-card-title v-if="currentPanel" style="padding-block: 0.2em; margin-block: 0">
        <v-card-text v-for="(panelRow, rowIndex) in currentPanel.interface" :key="currentPanel.idx + '-btn-' + rowIndex" style="margin-block: 0; padding-block: 0">
          <v-row dense>
            <v-col v-for="(moveBtn, butIndex) in panelRow" :key="currentPanel.idx + '-move-btn-' + rowIndex + '-' + butIndex">
              <v-btn v-if="moveBtn.name && moveBtn.name.startsWith('mdi-')"
                     :color="moveBtn.color"
                     :disabled="uiFrozen"
                     :title="moveBtn.name"
                     class="ml-0"
                     style="width:100%;" @click="customClick(moveBtn)"
                     :outlined="moveBtn.outline"
              >
                {{ moveBtn.prefix || '' }}<v-icon>{{ moveBtn.name }}</v-icon>{{ moveBtn.suffix || '' }}
              </v-btn>
              <v-btn v-if="moveBtn.name && !moveBtn.name.startsWith('mdi-') && moveBtn.name.toLowerCase() !== 'null'"
                     :color="moveBtn.color"
                     :disabled="uiFrozen"
                     :title="moveBtn.name"
                     class="ml-0"
                     style="width:100%;"
                     @click="customClick(moveBtn)"
                     :outlined="moveBtn.outline"
              >
                {{ moveBtn.name }}
              </v-btn>
              <div v-if="!moveBtn.name && moveBtn.val" style="outline: 1px solid blue; height:100%; display: flex; align-items: center; justify-content: center;">{{ customValues[moveBtn.val] }}</div>
            </v-col>
          </v-row>
        </v-card-text>
    </v-card-title>
  </v-card>
</template>

<script>
'use strict';

import Vue from 'vue';
import { registerRoute } from './';
import {mapActions, mapGetters, mapState} from 'vuex';
import store from '../store';

// store the needs of the registration call
let singletonReference = null;

export function reloadConfiguredUI() {
  if (singletonReference) singletonReference.install();
}

export default {
  data() {
    return { path: null, customValues: {} };
  },
  computed: {
    ...mapGetters(['isConnected', 'uiFrozen']),
    ...mapState('machine/model', ['move', 'model']),
    ...mapState('machine/settings', ['moveFeedrate']),
    ...mapState('mainMenu', ['configuredUI']),
    currentPanel() {
      if (!(this.configuredUI.panels || []).length) return null;
      return this.configuredUI.panels.find(p => p.path === this.path);
    }
  },
  mounted() {
    if (!window.moveitmoveit) window.moveitmoveit = this;
    if (!this.moveitFixtures) this.moveitFixtures = {};

    let loc = ('' + document.location);
    this.path = loc.substring(loc.indexOf('/', 10));
  },
  watch:{
    'configuredUI.panels': function(to) {
      if (Array.isArray(to)) {
        to.forEach(panel => {
          registerRoute(singletonReference, panel);
        });
      }
    },
    $route (to) {
      this.path = to.path;
    }
  },
  methods: {
    ...mapActions('machine', ['sendCode', 'download', 'upload']),

    templates() {
      return this.configuredUI.templates;
    },

    scripts() {
      return this.configuredUI.scripts;
    },

    async customClick(btn) {

      let sendIt = async (c) => {
        if (c.length > 257) {
          await this.upload({
            filename: '0:/macros/gcodeCache_temp.g',
            content: c,
            showProgress: false,
            showSuccess: false,
            showError: true
          });

          // now run the temp file...
          c = 'M98 P"0:/macros/gcodeCache_temp.g"';
        }
        // console.log(c);
        await this.sendCode(c);
      }

      const scripts = this.scripts();
      const templates = this.templates();

      if (btn.set && btn.val) {
        // value setting button
        this[btn.val] = btn.set;
        this.$set(this.customValues, btn.val, btn.set);

      } else if (!btn.code && btn.val && btn.axis) {
        // running gcode based on a reference value...
        let v = this.customValues[btn.val];
        if (!v) {
          this.$makeNotification('info', 'Missing value', 'Nothing has moved as the reference value was not set', 0);
          return;
        }

        let axisStr = (a) => {
          a = a.trim();
          let neg = a.startsWith('-');
          a = a[a.length - 1];
          return a.toUpperCase() + (neg ? '-' : '') + v;
        };

        await sendIt(`M120\nG91\nG1 ${btn.axis.split(',').map(axisStr).join(' ')} F${this.moveFeedrate}\nG90\nM121`);

      } else if (btn.code && btn.code.length > 2) {
        await sendIt(btn.code);

      } else if (btn.fixture) {
        let { fixture } = btn;
        let over = fixture.endsWith(':over');
        fixture = over ? fixture.substring(0, fixture.indexOf(':')) : fixture;

        if (btn.set) {
          // set a fixture value (record machine position for all axes)
          let axes = this.model.move.axes;
          let pv = this.moveitFixtures[fixture] = {};
          axes.forEach(a => (pv[a.letter.toUpperCase()] = a.machinePosition));

          this.saveFixtures();

        } else if (this.moveitFixtures[fixture]) {
          // go to a fixture
          let pos = this.moveitFixtures[fixture];

          let axes = this.model.move.axes;

          // going to hop up to 5mm below max on Z...
          let zax = axes.find(a => a.letter === 'Z');
          if (!zax) return;

          let gcode = 'G53 G1 Z' + (zax.max - 5) + '\n'
              + 'G53 G1 X' + pos.X + ' Y' + pos.Y;

          if (!over) gcode += '\nG53 G1 Z' + pos.Z;

          sendIt(gcode);
        }

      } else if ((btn.template && templates[btn.template]) || (btn.script && scripts[btn.script])) {
        let type = btn.template ? 'template' : 'script';

        let tempStr = this[type + 's']()[btn[type]];
        if (tempStr.startsWith('0:/')) {
          tempStr = await this.download({
            filename: tempStr,
            type: 'text',
            showProgress: false,
            showSuccess: false,
            showError: false
          });
        }

        // api for the script
        let params = btn.params; // eslint-disable-line
        let gcode = null;  // eslint-disable-line
        let model = this.model; // eslint-disable-line
        let sendCode = async (c) => this.sendCode(c); // eslint-disable-line
        let fixtures = this.moveitFixtures;  // eslint-disable-line
        let saveFixtures = async () => this.saveFixtures();  // eslint-disable-line

        try {
          let func = null;
          // make the functions
          if (type === 'template') {
            // make the template string evaulate as a function
            eval('func = () => {\ngcode = `' + tempStr + '`;\n}');
          } else {
            // wrap in async running function
            eval('func = async ({ params, gcode, model, sendCode, fixtures, saveFixtures, scripts, templates }) => {\n' + tempStr + '\n;return gcode;\n}');
          }

          // run the script function...
          gcode = await func({ params, gcode, model, sendCode, fixtures, saveFixtures, scripts, templates });

        } catch (e) {
          console.log(e);
        }

        if (gcode && gcode.length) {
          await sendIt(gcode);
        } else {
          console.log('NO GCODE GENERATED FROM TEMPLATE!');
        }
      }
    }
  },

  install() {
    singletonReference = this;
    const panels = ((store.getters["mainMenu/configuredUI"] || {}).panels || []);
    panels.forEach(panel => registerRoute(this, panel));

    // make sure there's an instance, so it can receive state updates
    new (Vue.extend(this))({ store });
  }
};

</script>

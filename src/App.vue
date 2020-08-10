<template>
  <div id="app">
    <p v-show="dmnIsValid">
      <el-button type="success" icon="el-icon-check" circle />
      DMN IS VALID
    </p>
    <p v-show="!dmnIsValid">
      <el-button type="danger" icon="el-icon-close" circle />
      DMN IS NOT VALID
    </p>
    <el-tabs type="card">
      <el-tab-pane label="Search and Filter">
        <h2>Custom Interface for Search and Filter</h2>
        <p>DMN Name: {{ this.dmnName }}</p>
      </el-tab-pane>
      <el-tab-pane label="Preview">
        <h2>Camunda DMN Preview</h2>
        <div class="camunda-dmn-container" ref="camundaDmnContainer" />
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script>
import LineItemMappingDmn from '!raw-loader!./assets/dmn/LINE_ITEM_MAPPING_DMN.dmn';
import DmnJS from 'dmn-js';
import DmnModdle from 'dmn-moddle';

const DMN_ROOT = 'dmn:Definitions';

export default {
  name: 'App',
  computed: {
    dmnName() {
      return this.moddleDefinitions?.name;
    },
  },
  data() {
    return {
      dmnIsValid: true,
      camundaDmnViewer: new DmnJS({
        height: '100%',
        width: '100%',
      }),
      moddle: new DmnModdle(),
      xmlData: LineItemMappingDmn,
      moddleDefinitions: undefined,
    };
  },
  mounted() {
    this.camundaDmnViewer.attachTo(this.$refs.camundaDmnContainer);

    this.syncCamundaXml();
    this.syncModdleXml();
  },
  methods: {
    async syncCamundaXml() {
      return new Promise((resolve, reject) => {
        this.camundaDmnViewer.importXML(this.xmlData, err => {
          if (err) {
            console.error('error rendering', err);
            reject(err);
          } else {
            console.log('rendered');

            var activeEditor = this.camundaDmnViewer.getActiveViewer();

            // access active editor components
            var canvas = activeEditor.get('canvas');

            // zoom to fit full viewport
            canvas.zoom('fit-viewport');

            resolve();
          }
        });
      });
    },
    async syncModdleXml() {
      return new Promise((resolve, reject) => {
        this.moddle.fromXML(this.xmlData, DMN_ROOT, (err, definitions) => {
          if (err) {
            console.error('error initing moddle', err);
            reject(err);
          }

          console.log('initialized', definitions);
          this.moddleDefinitions = definitions;
          resolve(definitions);
        });
      });
    },
  },
};
</script>

<style lang="scss">
#app {
  margin-top: 60px;

  .camunda-dmn-container {
    width: 100%;
    height: 1000px;
  }
}
</style>

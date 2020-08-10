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
        <p>Hit Policy: {{ this.hitPolicy }}</p>
        <el-table
          :data="customTableData"
          :header-cell-class-name="cellClassNameAssigner"
          :cell-class-name="cellClassNameAssigner"
          style="width: 100%"
        >
          <el-table-column
            v-for="column in customTableColumns"
            :key="column.id"
            :prop="column.id"
            :label="column.label"
            :filterMethod="filterMethod(column.id)"
            :filters="getFiltersForColumn(column.id)"
            filter-multiple
          />
        </el-table>
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

const ColumnTypes = Object.freeze({
  INPUT: 'dmn:InputClause',
  OUTPUT: 'dmn:OutputClause',
});

export default {
  ColumnTypes,
  name: 'App',
  computed: {
    dmnName() {
      return this.moddleDefinitions?.name;
    },
    hitPolicy() {
      return this.dmnDecisionLogic?.hitPolicy;
    },
    dmnDecisionLogic() {
      return this.moddleDefinitions?.drgElement?.[0].decisionLogic;
    },
    /**
     * Combines DMN input and output columns definitions to be used as column definitions in el-table
     *
     * @returns ModdleElement[]
     */
    customTableColumns() {
      if (this.dmnDecisionLogic === undefined) {
        return [];
      }

      return [...this.dmnDecisionLogic.input, ...this.dmnDecisionLogic.output];
    },
    /**
     * Converts DMN rules into an array of objects. Each rule's inputEntry and outEntry are transformed into
     *  a map that is keyed by the associated column definition id so they can be linked to el-table-column
     *
     *  @returns Object[] [{ 'columnDefinitionId': { ...ruleData } }]
     */
    customTableData() {
      const rules = this.dmnDecisionLogic?.rule;
      if (rules === undefined) {
        return [];
      }

      return rules.map(rule => {
        const ruleColumns = [...rule.inputEntry, ...rule.outputEntry];

        if (ruleColumns.length !== this.customTableColumns.length) {
          throw new Error(
            'Rule column length and decision table column length should always match.'
          );
        }

        return ruleColumns.reduce((resultingObject, ruleColumn, index) => {
          const objectKeyName = this.customTableColumns[index].id;
          resultingObject[objectKeyName] = ruleColumn;

          return resultingObject;
        }, {});
      });
    },
    /**
     * Produces a map with the key being the column definition id and the values being a Set of
     * the text for each rule in the column.
     *
     * @returns Object { 'columnDefinitionId': [...ruleTextValues]}
     */
    columnFiltersMap() {
      return this.customTableColumns.reduce((resultingObject, column) => {
        const items = new Set();

        this.customTableData.forEach(data => {
          items.add(data[column.id].text);
        });

        resultingObject[column.id] = items;
        return resultingObject;
      }, {});
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
    cellClassNameAssigner({ columnIndex }) {
      const column = this.customTableColumns?.[columnIndex];
      if (ColumnTypes.INPUT === column?.$type) {
        return 'input-column';
      }

      if (ColumnTypes.OUTPUT === column?.$type) {
        return 'output-column';
      }
    },
    getFiltersForColumn(columnId) {
      const columnData = this.columnFiltersMap?.[columnId] ?? [];

      return [...columnData].map(val => ({
        text: val.length > 0 ? val : '(empty)',
        value: val,
      }));
    },
    filterMethod(columnId) {
      return (value, row) => row[columnId].text === value;
    },
  },
};
</script>

<style lang="scss">
$--font-path: '~element-ui/lib/theme-chalk/fonts';

@import '~element-ui/lib/theme-chalk/reset.css';
@import '~element-ui/packages/theme-chalk/src/index.scss';

#app {
  margin-top: 60px;

  .camunda-dmn-container {
    width: 100%;
    height: 1000px;
  }

  .output-column {
    background: #eee;
  }
}
</style>

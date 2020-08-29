<template>
  <div id="app">
    <el-row style="background: #EEE; margin-bottom: 18px;">
      DOCUMENT:
      <el-select
        v-model="document"
        placeholder="Select a Document"
        @change="onDocumentChange"
      >
        <el-option
          v-for="document in Object.keys($options.Document)"
          :label="document"
          :key="document"
          :value="document"
        />
      </el-select>
    </el-row>
    <el-tabs type="card">
      <el-tab-pane label="Search and Filter">
        <h2>Custom Interface for Search and Filter</h2>
        <p>DMN Name: {{ this.dmnName }}</p>
        <p>Hit Policy: {{ this.hitPolicy }}</p>
        <el-button
          type="primary"
          style="float: right; margin-bottom: 18px;"
          @click="onDisplayAddRule"
        >
          Add Rule
        </el-button>
        <el-input
          prefix-icon="el-icon-search"
          type="primary"
          style="float: right; margin-bottom: 18px; margin-right: 9px; width: 200px"
          placeholder="Full-text Search"
          v-model="search"
        />
        <el-table
          :data="customTableDataFiltered"
          :header-cell-class-name="cellClassNameAssigner"
          :cell-class-name="cellClassNameAssigner"
          highlight-current-row
          @current-change="onCurrentRowChange"
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
            sortable
            :sort-method="columnSortMethod(column.id)"
          />
        </el-table>
        <el-dialog :visible.sync="ruleEditorVm.visible">
          <el-form>
            <el-form-item
              v-for="column in customTableColumns"
              :key="column.id"
              :label="column.label"
            >
              <el-input v-model="ruleEditorVm.rule[column.id]" />
            </el-form-item>
            <el-button type="primary" @click="onAddRule">Add Rule</el-button>
          </el-form>
        </el-dialog>
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
import LineItemMappingDmnBicBoi from '!raw-loader!./assets/dmn/LINE_ITEM_MAPPING_DMN_BIC_BOI.dmn';
import DmnJS from 'dmn-js';
import DmnModdle from 'dmn-moddle';

const Document = Object.freeze({
  LineItemMappingDmn,
  LineItemMappingDmnBicBoi,
});

const DMN_ROOT = 'dmn:Definitions';

const ColumnTypes = Object.freeze({
  INPUT: 'dmn:InputClause',
  OUTPUT: 'dmn:OutputClause',
});

const MODDLE_INSTANCE = new DmnModdle();

// TODO: Element id attributes need a unique value. This is good enough for this playground
function goodEnoughUidGenerator(length = 8) {
  var result = '';
  var characters =
    'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  var charactersLength = characters.length;
  for (var i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
  }
  return result;
}

export default {
  Document,
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
    customTableDataFiltered() {
      if (!this.search?.length) {
        return this.customTableData;
      }

      return this.customTableData.filter(
        row =>
          Object.values(row).filter(column =>
            (column || '')?.text?.includes(this.search)
          ).length
      );
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
      camundaDmnViewer: new DmnJS({
        height: '100%',
        width: '100%',
      }),
      document: 'LineItemMappingDmn',
      xmlData: LineItemMappingDmn,
      moddleDefinitions: undefined,
      currentRow: undefined,
      search: undefined,
      ruleEditorVm: {
        visible: false,
        rule: {},
      },
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
            console.info('rendered');

            const activeEditor = this.camundaDmnViewer.getActiveViewer();

            try {
              // access active editor components
              const canvas = activeEditor.get('canvas');

              // zoom to fit full viewport
              canvas.zoom('fit-viewport');
            } catch (e) {
              console.warn('fit to canvas failed', e);
            }

            const dmnTableView = this.camundaDmnViewer.getViews()?.[1];
            if (dmnTableView) {
              console.info('switching to DMN table view');
              this.camundaDmnViewer.open(dmnTableView);
            }

            resolve();
          }
        });
      });
    },
    async syncModdleXml() {
      return new Promise((resolve, reject) => {
        MODDLE_INSTANCE.fromXML(this.xmlData, DMN_ROOT, (err, definitions) => {
          if (err) {
            console.error('error initing moddle', err);
            reject(err);
          }

          console.info('initialized', definitions);
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
    onCurrentRowChange(currentRow) {
      this.currentRow = currentRow;
    },
    onDisplayAddRule() {
      this.ruleEditorVm.visible = true;

      // initialize ruleEditor object properties so they are reactive
      this.ruleEditorVm.rule = this.customTableColumns.reduce(
        (resultingObject, column) => {
          resultingObject[column.id] = '';
          return resultingObject;
        },
        {}
      );
    },
    columnSortMethod(columnId) {
      return (rowA, rowB) => {
        return (rowA?.[columnId]?.text || '').localeCompare(
          rowB?.[columnId]?.text
        );
      };
    },
    onAddRule() {
      const rule = this.ruleEditorVm.rule;

      const inputEntries = this.customTableColumns
        .filter(column => column.$type === ColumnTypes.INPUT)
        .map(column =>
          MODDLE_INSTANCE.create('dmn:UnaryTests', {
            id: `UnaryTests_${goodEnoughUidGenerator()}`,
            text: rule[column.id],
          })
        );

      const outputEntries = this.customTableColumns
        .filter(column => column.$type === ColumnTypes.OUTPUT)
        .map(column =>
          MODDLE_INSTANCE.create('dmn:LiteralExpression', {
            id: `LiteralExpression_${goodEnoughUidGenerator()}`,
            text: rule[column.id],
          })
        );

      const ruleElement = MODDLE_INSTANCE.create('dmn:DecisionRule', {
        id: `DecisionRule_${goodEnoughUidGenerator()}`,
        inputEntry: inputEntries,
        outputEntry: outputEntries,
      });

      console.info('rule', ruleElement);
      this.moddleDefinitions?.drgElement?.[0].decisionLogic.rule.push(
        ruleElement
      );
      // not the right way to do it, but good enough for playground
      this.updateCamundaOfModdleDefinitionChange();
    },
    async updateCamundaOfModdleDefinitionChange() {
      return new Promise((resolve, reject) => {
        MODDLE_INSTANCE.toXML(this.moddleDefinitions, {}, (err, xml) => {
          if (err) {
            console.error('error writing moddle definitions to xml', err);
            reject(err);
          }

          console.info('generated xml', xml);
          this.xmlData = xml;
          this.syncCamundaXml();
          resolve(xml);
        });
      });
    },
    onDocumentChange(documentId) {
      this.xmlData = Document[documentId];

      this.syncCamundaXml();
      this.syncModdleXml();
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

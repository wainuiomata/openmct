<!--
 Open MCT, Copyright (c) 2014-2024, United States Government
 as represented by the Administrator of the National Aeronautics and Space
 Administration. All rights reserved.

 Open MCT is licensed under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
 http://www.apache.org/licenses/LICENSE-2.0.

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 License for the specific language governing permissions and limitations
 under the License.

 Open MCT includes source code licensed under additional open source
 licenses. See the Open Source Licenses file (LICENSES.md) included with
 this source code distribution or the Licensing information page available
 at runtime from the About dialog for additional information.
-->
<template>
  <tr
    :style="{ top: rowTop }"
    class="noselect"
    :class="[rowClass, { 'is-selected': marked }]"
    v-on="listeners"
  >
    <component
      :is="componentList[key]"
      v-for="(title, key) in headers"
      :key="key"
      :column-key="key"
      :style="rowStyle(key)"
      :class="[cellLimitClasses[key], selectableColumns[key] ? 'is-selectable' : '']"
      :object-path="objectPath"
      :row="row"
    />
  </tr>
</template>

<script>
import TableCell from './TableCell.vue';

export default {
  components: {
    TableCell
  },
  inject: ['openmct', 'currentView'],
  props: {
    headers: {
      type: Object,
      required: true
    },
    row: {
      type: Object,
      required: true
    },
    columnWidths: {
      type: Object,
      required: true
    },
    objectPath: {
      type: Array,
      required: true
    },
    rowIndex: {
      type: Number,
      required: false,
      default: undefined
    },
    rowOffset: {
      type: Number,
      required: false,
      default: 0
    },
    rowHeight: {
      type: Number,
      required: false,
      default: 0
    },
    marked: {
      type: Boolean,
      required: false,
      default: false
    }
  },
  emits: ['mark-multiple-concurrent', 'unmark', 'mark', 'row-context-click'],
  data: function () {
    return {
      rowTop: (this.rowOffset + this.rowIndex) * this.rowHeight + 'px',
      rowClass: this.row.getRowClass(),
      cellLimitClasses: this.row.getCellLimitClasses(),
      selectableColumns: Object.keys(this.row.columns).reduce((selectable, columnKeys) => {
        selectable[columnKeys] = this.row.columns[columnKeys].selectable;

        return selectable;
      }, {})
    };
  },
  computed: {
    listeners() {
      let listenersObject = {
        click: this.markRow
      };

      if (this.row.getContextMenuActions().length) {
        listenersObject.contextmenu = this.showContextMenu;
      }

      return listenersObject;
    },
    componentList() {
      return Object.keys(this.headers).reduce((components, header) => {
        components[header] = this.row.getCellComponentName(header) || 'table-cell';

        return components;
      }, {});
    }
  },
  // TODO: use computed properties
  watch: {
    rowOffset: 'calculateRowTop',
    row: {
      handler: 'formatRow',
      deep: true
    }
  },
  methods: {
    calculateRowTop: function (rowOffset) {
      this.rowTop = (rowOffset + this.rowIndex) * this.rowHeight + 'px';
    },
    formatRow: function (row) {
      this.rowClass = row.getRowClass();
      this.cellLimitClasses = row.getCellLimitClasses();
    },
    markRow: function (event) {
      let keyCtrlModifier = false;

      if (event.ctrlKey || event.metaKey) {
        keyCtrlModifier = true;
      }

      if (event.shiftKey) {
        this.$emit('mark-multiple-concurrent', this.rowIndex);
      } else {
        if (this.marked) {
          this.$emit('unmark', this.rowIndex, keyCtrlModifier);
        } else {
          this.$emit('mark', this.rowIndex, keyCtrlModifier);
        }
      }
    },
    selectCell(element, columnKey) {
      if (this.selectableColumns[columnKey]) {
        //TODO: This is a hack. Cannot get parent this way.
        this.openmct.selection.select(
          [
            {
              element: element,
              context: {
                type: 'table-cell',
                row: this.row.objectKeyString,
                column: columnKey
              }
            },
            {
              element: this.openmct.layout.$refs.browseObject.$el,
              context: {
                item: this.openmct.router.path[0]
              }
            }
          ],
          false
        );
        event.stopPropagation();
      }
    },
    getDatum() {
      return this.row.fullDatum;
    },
    rowStyle(key) {
      return this.columnWidths[key] === undefined
        ? {}
        : { width: this.columnWidths[key] + 'px', 'max-width': this.columnWidths[key] + 'px' };
    },
    showContextMenu: async function (event) {
      event.preventDefault();

      this.updateViewContext();
      this.markRow(event);

      const contextualDomainObject = await this.row.getContextualDomainObject?.(
        this.openmct,
        this.row.objectKeyString
      );

      let objectPath = this.objectPath;
      if (contextualDomainObject) {
        objectPath = objectPath.slice();
        objectPath.unshift(contextualDomainObject);
      }

      const actions = this.row
        .getContextMenuActions()
        .map((key) => this.openmct.actions.getAction(key));
      const menuItems = this.openmct.menus.actionsToMenuItems(
        actions,
        objectPath,
        this.currentView
      );
      if (menuItems.length) {
        this.openmct.menus.showMenu(event.x, event.y, menuItems);
      }
    },
    updateViewContext() {
      this.$emit('row-context-click', {
        viewHistoricalData: true,
        viewDatumAction: true,
        getDatum: this.getDatum
      });
    }
  }
};
</script>

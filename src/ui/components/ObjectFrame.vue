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
  <div
    ref="soView"
    class="c-so-view js-notebook-snapshot-item-wrapper"
    :class="[
      statusClass,
      widthClass,
      'c-so-view--' + domainObject.type,
      {
        'c-so-view--no-frame': !hasFrame,
        'has-complex-content': complexContent
      }
    ]"
  >
    <div class="c-so-view__header">
      <div class="c-object-label" :class="[statusClass]">
        <div class="c-object-label__type-icon" :class="cssClass">
          <span
            class="is-status__indicator"
            :aria-label="`This item is ${status}`"
            :title="`This item is ${status}`"
          ></span>
        </div>
        <div
          ref="objectName"
          class="c-object-label__name"
          @mouseover.ctrl="showToolTip"
          @mouseleave="hideToolTip"
        >
          {{ domainObject && domainObject.name }}
        </div>
      </div>

      <div
        class="c-so-view__frame-controls"
        :class="{
          'c-so-view__frame-controls--no-frame': !hasFrame,
          'has-complex-content': complexContent
        }"
      >
        <div v-if="supportsIndependentTime" class="c-conductor-holder--compact">
          <independent-time-conductor :domain-object="domainObject" :object-path="objectPath" />
        </div>
        <NotebookMenuSwitcher
          v-if="notebookEnabled"
          :domain-object="domainObject"
          :object-path="objectPath"
          class="c-notebook-snapshot-menubutton"
        />
        <div v-if="statusBarItems.length > 0" class="c-so-view__frame-controls__btns">
          <button
            v-for="(item, index) in statusBarItems"
            :key="index"
            class="c-icon-button"
            :class="item.cssClass"
            :title="item.name"
            :aria-label="item.name"
            @click="item.onItemClicked"
          >
            <span class="c-icon-button__label">{{ item.name }}</span>
          </button>
        </div>
        <button
          class="c-icon-button icon-3-dots c-so-view__frame-controls__more"
          aria-label="View menu items"
          title="View menu items"
          @click.prevent.stop="showMenuItems($event)"
        ></button>
      </div>
    </div>

    <object-view
      ref="objectView"
      class="c-so-view__object-view js-object-view js-notebook-snapshot-item"
      :show-edit-view="showEditView"
      :object-path="objectPath"
      :layout-font-size="layoutFontSize"
      :layout-font="layoutFont"
      @change-action-collection="setActionCollection"
    />
  </div>
</template>

<script>
import NotebookMenuSwitcher from '@/plugins/notebook/components/NotebookMenuSwitcher.vue';
import IndependentTimeConductor from '@/plugins/timeConductor/independent/IndependentTimeConductor.vue';

import tooltipHelpers from '../../api/tooltips/tooltipMixins.js';
import { SupportedViewTypes } from '../../utils/constants.js';
import ObjectView from './ObjectView.vue';

const SIMPLE_CONTENT_TYPES = ['clock', 'timer', 'summary-widget', 'hyperlink', 'conditionWidget'];
const CSS_WIDTH_LESS_STR = '--width-less-than-';

export default {
  components: {
    ObjectView,
    NotebookMenuSwitcher,
    IndependentTimeConductor
  },
  mixins: [tooltipHelpers],
  inject: ['openmct'],
  props: {
    domainObject: {
      type: Object,
      required: true
    },
    objectPath: {
      type: Array,
      required: true
    },
    hasFrame: Boolean,
    showEditView: {
      type: Boolean,
      default: true
    },
    layoutFontSize: {
      type: String,
      default: ''
    },
    layoutFont: {
      type: String,
      default: ''
    }
  },
  data() {
    let objectType = this.openmct.types.get(this.domainObject.type);

    let cssClass =
      objectType && objectType.definition ? objectType.definition.cssClass : 'icon-object-unknown';

    let complexContent = !SIMPLE_CONTENT_TYPES.includes(this.domainObject.type);

    return {
      cssClass,
      widthClass: '',
      complexContent,
      notebookEnabled: this.openmct.types.get('notebook'),
      statusBarItems: [],
      status: '',
      supportsIndependentTime: false
    };
  },
  computed: {
    statusClass() {
      return this.status ? `is-status--${this.status}` : '';
    }
  },
  mounted() {
    this.status = this.openmct.status.get(this.domainObject.identifier);
    this.removeStatusListener = this.openmct.status.observe(
      this.domainObject.identifier,
      this.setStatus
    );
    const provider = this.openmct.objectViews.get(this.domainObject, this.objectPath)[0];
    if (provider) {
      this.$refs.objectView.show(this.domainObject, provider.key, false, this.objectPath);
    }

    if (this.$refs.soView) {
      this.soViewResizeObserver = new ResizeObserver(this.resizeSoView);
      this.soViewResizeObserver.observe(this.$refs.soView);
    }

    const viewKey = this.getViewKey();
    this.supportsIndependentTime = this.domainObject && SupportedViewTypes.includes(viewKey);
  },
  beforeUnmount() {
    this.removeStatusListener();

    if (this.actionCollection) {
      this.unlistenToActionCollection();
    }

    if (this.soViewResizeObserver) {
      this.soViewResizeObserver.disconnect();
    }
  },
  methods: {
    getSelectionContext() {
      return this.$refs.objectView.getSelectionContext();
    },
    setActionCollection(actionCollection) {
      if (this.actionCollection) {
        this.unlistenToActionCollection();
      }

      this.actionCollection = actionCollection;
      this.actionCollection.on('update', this.updateActionItems);
      this.updateActionItems();
    },
    unlistenToActionCollection() {
      this.actionCollection.off('update', this.updateActionItems);
      delete this.actionCollection;
    },
    updateActionItems() {
      const statusBarItems = this.actionCollection.getStatusBarActions();
      this.statusBarItems = this.openmct.menus.actionsToMenuItems(
        statusBarItems,
        this.actionCollection.objectPath,
        this.actionCollection.view
      );
      this.menuActionItems = this.actionCollection.getVisibleActions();
    },
    showMenuItems(event) {
      const sortedActions = this.openmct.actions._groupAndSortActions(this.menuActionItems);
      if (sortedActions.length) {
        const menuItems = this.openmct.menus.actionsToMenuItems(
          sortedActions,
          this.actionCollection.objectPath,
          this.actionCollection.view
        );
        this.openmct.menus.showMenu(event.x, event.y, menuItems);
      }
    },
    setStatus(status) {
      this.status = status;
    },
    resizeSoView() {
      let cW = this.$refs.soView.offsetWidth;
      let widths = [220, 600];
      let wClass = '';

      for (let width of widths) {
        if (cW < width) {
          wClass = wClass.concat(' ', CSS_WIDTH_LESS_STR, width);
        }
      }

      this.widthClass = wClass.trimStart();
    },
    getViewKey() {
      return this.$refs.objectView?.viewKey;
    },
    async showToolTip() {
      const { BELOW } = this.openmct.tooltips.TOOLTIP_LOCATIONS;
      this.buildToolTip(await this.getObjectPath(), BELOW, 'objectName');
    }
  }
};
</script>

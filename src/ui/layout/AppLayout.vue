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
    class="l-shell"
    :class="{
      'is-editing': isEditing
    }"
  >
    <div id="splash-screen"></div>

    <div
      class="l-shell__head"
      :class="{
        'l-shell__head--expanded': headExpanded,
        'l-shell__head--minify-indicators': !headExpanded
      }"
    >
      <CreateButton class="l-shell__create-button" />
      <GrandSearch ref="grand-search" />
      <StatusIndicators />
      <button
        class="l-shell__head__collapse-button c-icon-button"
        :class="
          headExpanded
            ? 'l-shell__head__collapse-button--collapse'
            : 'l-shell__head__collapse-button--expand'
        "
        :aria-label="`Click to ${headExpanded ? 'collapse' : 'expand'} items`"
        :title="`Click to ${headExpanded ? 'collapse' : 'expand'} items`"
        @click="toggleShellHead"
      ></button>
      <notification-banner />
      <div class="l-shell__head-section l-shell__controls">
        <button
          class="c-icon-button c-icon-button--major icon-new-window"
          title="Open in a new browser tab"
          target="_blank"
          @click="openInNewTab"
        ></button>
        <button
          :class="[
            'c-icon-button c-icon-button--major',
            fullScreen ? 'icon-fullscreen-collapse' : 'icon-fullscreen-expand'
          ]"
          :aria-label="`${fullScreen ? 'Exit' : 'Enable'} full screen mode`"
          :title="`${fullScreen ? 'Exit' : 'Enable'} full screen mode`"
          @click="fullScreenToggle"
        ></button>
      </div>
      <app-logo />
    </div>

    <div class="l-shell__drawer c-drawer c-drawer--push c-drawer--align-top"></div>

    <multipane class="l-shell__main" :class="[resizingClass]" type="horizontal">
      <pane
        class="l-shell__pane-tree"
        handle="after"
        label="Browse"
        hide-param="hideTree"
        :persist-position="true"
        @start-resizing="onStartResizing"
        @end-resizing="onEndResizing"
      >
        <template #controls>
          <button
            class="c-icon-button l-shell__reset-tree-button icon-folders-collapse"
            aria-label="Collapse all tree items"
            title="Collapse all tree items"
            @click="handleTreeReset"
          ></button>
          <button
            class="c-icon-button l-shell__sync-tree-button icon-target"
            aria-label="Show selected item in tree"
            title="Show selected item in tree"
            @click="handleSyncTreeNavigation"
          ></button>
        </template>
        <multipane type="vertical">
          <pane>
            <mct-tree
              ref="mctTree"
              :sync-tree-navigation="triggerSync"
              :reset-tree-navigation="triggerReset"
              class="l-shell__tree"
            />
          </pane>
          <pane
            handle="before"
            label="Recently Viewed"
            :persist-position="true"
            collapse-type="horizontal"
            hide-param="hideRecents"
          >
            <RecentObjectsList
              ref="recentObjectsList"
              class="l-shell__tree"
              @open-and-scroll-to="openAndScrollTo($event)"
              @set-clear-button-disabled="setClearButtonDisabled"
            />
            <template #controls>
              <button
                class="c-icon-button icon-clear-data"
                aria-label="Clear Recently Viewed"
                title="Clear Recently Viewed"
                :disabled="disableClearButton"
                @click="handleClearRecentObjects"
              ></button>
            </template>
          </pane>
        </multipane>
      </pane>
      <pane class="l-shell__pane-main" role="main">
        <browse-bar
          ref="browseBar"
          class="l-shell__main-view-browse-bar"
          :action-collection="actionCollection"
          @sync-tree-navigation="handleSyncTreeNavigation"
        />
        <toolbar v-if="toolbar" class="l-shell__toolbar" />
        <object-view
          ref="browseObject"
          class="l-shell__main-container js-main-container js-notebook-snapshot-item"
          data-selectable
          :show-edit-view="true"
          @change-action-collection="setActionCollection"
        />
        <component :is="conductorComponent" class="l-shell__time-conductor" />
      </pane>
      <pane
        class="l-shell__pane-inspector l-pane--holds-multipane"
        handle="before"
        label="Inspect"
        hide-param="hideInspector"
        :persist-position="true"
        @start-resizing="onStartResizing"
        @end-resizing="onEndResizing"
      >
        <Inspector ref="inspector" :is-editing="isEditing" />
      </pane>
    </multipane>
  </div>
</template>

<script>
import ObjectView from '../components/ObjectView.vue';
import Inspector from '../inspector/InspectorPanel.vue';
import Toolbar from '../toolbar/ToolbarContainer.vue';
import AppLogo from './AppLogo.vue';
import BrowseBar from './BrowseBar.vue';
import CreateButton from './CreateButton.vue';
import MctTree from './MctTree.vue';
import Multipane from './MultipaneContainer.vue';
import Pane from './PaneContainer.vue';
import RecentObjectsList from './RecentObjectsList.vue';
import GrandSearch from './search/GrandSearch.vue';
import NotificationBanner from './status-bar/NotificationBanner.vue';
import StatusIndicators from './status-bar/StatusIndicators.vue';

export default {
  components: {
    Inspector,
    MctTree,
    ObjectView,
    CreateButton,
    GrandSearch,
    Multipane,
    Pane,
    BrowseBar,
    Toolbar,
    AppLogo,
    StatusIndicators,
    NotificationBanner,
    RecentObjectsList
  },
  inject: ['openmct'],
  data: function () {
    let storedHeadProps = window.localStorage.getItem('openmct-shell-head');
    let headExpanded = true;
    if (storedHeadProps) {
      headExpanded = JSON.parse(storedHeadProps).expanded;
    }

    return {
      fullScreen: false,
      conductorComponent: undefined,
      isEditing: false,
      hasToolbar: false,
      actionCollection: undefined,
      triggerSync: false,
      triggerReset: false,
      headExpanded,
      isResizing: false,
      disableClearButton: false
    };
  },
  computed: {
    toolbar() {
      return this.hasToolbar && this.isEditing;
    },
    resizingClass() {
      return this.isResizing ? 'l-shell__resizing' : '';
    }
  },
  mounted() {
    this.openmct.editor.on('isEditing', (isEditing) => {
      this.isEditing = isEditing;
    });

    this.openmct.selection.on('change', this.toggleHasToolbar);
  },
  methods: {
    enterFullScreen() {
      let docElm = document.documentElement;

      if (docElm.requestFullscreen) {
        docElm.requestFullscreen();
      } else if (docElm.mozRequestFullScreen) {
        /* Firefox */
        docElm.mozRequestFullScreen();
      } else if (docElm.webkitRequestFullscreen) {
        /* Chrome, Safari and Opera */
        docElm.webkitRequestFullscreen();
      } else if (docElm.msRequestFullscreen) {
        /* IE/Edge */
        docElm.msRequestFullscreen();
      }
    },
    exitFullScreen() {
      if (document.exitFullscreen) {
        document.exitFullscreen();
      } else if (document.mozCancelFullScreen) {
        document.mozCancelFullScreen();
      } else if (document.webkitCancelFullScreen) {
        document.webkitCancelFullScreen();
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen();
      }
    },
    toggleShellHead() {
      this.headExpanded = !this.headExpanded;

      window.localStorage.setItem(
        'openmct-shell-head',
        JSON.stringify({
          expanded: this.headExpanded
        })
      );
    },
    fullScreenToggle() {
      if (this.fullScreen) {
        this.fullScreen = false;
        this.exitFullScreen();
      } else {
        this.fullScreen = true;
        this.enterFullScreen();
      }
    },
    openInNewTab(event) {
      window.open(window.location.href);
    },
    toggleHasToolbar(selection) {
      let structure = undefined;

      if (!selection || !selection[0]) {
        structure = [];
      } else {
        structure = this.openmct.toolbars.get(selection);
      }

      this.hasToolbar = structure.length > 0;
    },
    openAndScrollTo(navigationPath) {
      this.$refs.mctTree.openAndScrollTo(navigationPath);
      this.$refs.mctTree.targetedPath = navigationPath;
    },
    setActionCollection(actionCollection) {
      this.actionCollection = actionCollection;
    },
    handleSyncTreeNavigation() {
      this.triggerSync = !this.triggerSync;
    },
    handleTreeReset() {
      this.triggerReset = !this.triggerReset;
    },
    handleClearRecentObjects() {
      this.$refs.recentObjectsList.clearRecentObjects();
    },
    onStartResizing() {
      this.isResizing = true;
    },
    onEndResizing() {
      this.isResizing = false;
    },
    setClearButtonDisabled(isDisabled) {
      this.disableClearButton = isDisabled;
    }
  }
};
</script>

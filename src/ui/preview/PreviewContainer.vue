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
  <div role="dialog" aria-label="Preview Container" class="l-preview-window js-preview-window">
    <PreviewHeader
      ref="previewHeader"
      :current-view="currentViewProvider"
      :action-collection="actionCollection"
      :domain-object="domainObject"
      :views="viewProviders"
    />
    <div class="l-preview-window__object-view js-notebook-snapshot-item">
      <div ref="objectView"></div>
    </div>
  </div>
</template>

<script>
import { nextTick } from 'vue';

import StyleRuleManager from '@/plugins/condition/StyleRuleManager';
import { STYLE_CONSTANTS } from '@/plugins/condition/utils/constants';

import PreviewHeader from './PreviewHeader.vue';

export default {
  components: {
    PreviewHeader
  },
  inject: ['openmct', 'objectPath'],
  props: {
    viewOptions: {
      type: Object,
      default() {
        return {};
      }
    },
    existingView: {
      type: Object,
      default() {
        return undefined;
      }
    }
  },
  data() {
    let domainObject = this.objectPath[0];

    return {
      domainObject: domainObject,
      viewKey: undefined,
      viewProviders: [],
      currentViewProvider: {},
      actionCollection: undefined,
      existingViewIndex: 0
    };
  },
  mounted() {
    this.viewProviders = this.openmct.objectViews.get(this.domainObject, this.objectPath);
    this.viewProviders.forEach((provider, index) => {
      provider.onItemClicked = () => {
        if (this.existingView && provider.key === this.existingView.key) {
          this.existingViewIndex = index;
        }

        this.setView(provider);
      };
    });

    this.setView(this.viewProviders[0]);
  },
  beforeUnmount() {
    if (this.stopListeningStyles) {
      this.stopListeningStyles();
    }

    if (this.styleRuleManager) {
      this.styleRuleManager.destroy();
      delete this.styleRuleManager;
    }

    if (this.actionCollection) {
      this.actionCollection.destroy();
    }
  },
  unmounted() {
    if (!this.existingView) {
      this.view.destroy();
    } else if (this.existingViewElement) {
      // if the existing view element is populated, it's the currently viewed view
      // in preview and we need to add it back to the parent.
      this.addExistingViewBackToParent();
    }
  },
  methods: {
    clear() {
      if (this.view) {
        if (this.view !== this.existingView) {
          this.view.destroy();
        } else {
          this.addExistingViewBackToParent();
        }

        this.$refs.objectView.innerHTML = '';
        delete this.view;
        delete this.viewContainer;
      }
    },
    setView(viewProvider) {
      if (this.viewKey === viewProvider.key) {
        return;
      }

      const isExistingView = viewProvider.key === this.existingView?.key;

      this.clear();

      this.viewKey = viewProvider.key;
      this.initializeViewContainer();

      if (isExistingView) {
        this.view = this.existingView;
        this.existingViewElement = this.existingView.parentElement.firstElementChild;
        this.currentViewProvider = this.viewProviders[this.existingViewIndex];
      } else {
        this.currentViewProvider = viewProvider;
        this.view = this.currentViewProvider.view(this.domainObject, this.objectPath);
      }

      if (isExistingView) {
        this.viewContainer.appendChild(this.existingViewElement);
      } else {
        // in preview mode, we're always visible
        this.viewOptions.renderWhenVisible = (func) => {
          window.requestAnimationFrame(func);
          return true;
        };
        this.view.show(this.viewContainer, false, this.viewOptions);
      }

      this.initObjectStyles();

      nextTick(() => {
        this.getActionsCollection(this.view);
      });
    },
    addExistingViewBackToParent() {
      this.existingView.parentElement.appendChild(this.existingViewElement);
      delete this.existingViewElement;
    },
    initializeViewContainer() {
      this.viewContainer = this.$refs.objectView;
    },
    getActionsCollection(view) {
      if (this.actionCollection) {
        this.actionCollection.destroy();
      }

      this.actionCollection = this.openmct.actions.getActionsCollection(this.objectPath, view);
    },
    initObjectStyles() {
      if (!this.styleRuleManager) {
        this.styleRuleManager = new StyleRuleManager(
          this.domainObject.configuration && this.domainObject.configuration.objectStyles,
          this.openmct,
          this.updateStyle.bind(this)
        );
      } else {
        this.styleRuleManager.updateObjectStyleConfig(
          this.domainObject.configuration && this.domainObject.configuration.objectStyles
        );
      }

      if (this.stopListeningStyles) {
        this.stopListeningStyles();
      }

      this.stopListeningStyles = this.openmct.objects.observe(
        this.domainObject,
        'configuration.objectStyles',
        (newObjectStyle) => {
          //Updating styles in the inspector view will trigger this so that the changes are reflected immediately
          this.styleRuleManager.updateObjectStyleConfig(newObjectStyle);
        }
      );
    },
    updateStyle(styleObj) {
      if (!styleObj) {
        return;
      }

      let keys = Object.keys(styleObj);
      let firstChild = this.$refs.objectView.querySelector(':first-child');

      keys.forEach((key) => {
        if (firstChild) {
          if (typeof styleObj[key] === 'string' && styleObj[key].indexOf('__no_value') > -1) {
            if (firstChild.style[key]) {
              firstChild.style[key] = '';
            }
          } else {
            if (
              !styleObj.isStyleInvisible &&
              firstChild.classList.contains(STYLE_CONSTANTS.isStyleInvisible)
            ) {
              firstChild.classList.remove(STYLE_CONSTANTS.isStyleInvisible);
            } else if (
              styleObj.isStyleInvisible &&
              !firstChild.classList.contains(styleObj.isStyleInvisible)
            ) {
              firstChild.classList.add(styleObj.isStyleInvisible);
            }

            firstChild.style[key] = styleObj[key];
          }
        }
      });
    }
  }
};
</script>

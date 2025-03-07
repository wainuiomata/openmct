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
    class="l-layout__frame c-frame"
    :class="{
      'no-frame': !item.hasFrame,
      'u-inspectable': inspectable
    }"
    :style="style"
  >
    <slot name="content"></slot>
    <div
      class="c-frame__move-bar"
      :aria-label="`Move ${typeName} Frame`"
      @mousedown.left="startMove($event)"
    ></div>
  </div>
</template>

<script>
import _ from 'lodash';

import DRAWING_OBJECT_TYPES from '../DrawingObjectTypes.js';
import LayoutDrag from './../LayoutDrag.js';

export default {
  inject: ['openmct'],
  props: {
    item: {
      type: Object,
      required: true
    },
    gridSize: {
      type: Array,
      required: true,
      validator: (arr) => arr && arr.length === 2 && arr.every((el) => typeof el === 'number')
    },
    isEditing: {
      type: Boolean,
      required: true
    }
  },
  emits: ['move', 'end-move'],
  computed: {
    typeName() {
      const { type } = this.item;
      if (DRAWING_OBJECT_TYPES[type]) {
        return DRAWING_OBJECT_TYPES[type].name;
      }
      return 'Sub-object';
    },
    size() {
      let { width, height } = this.item;

      return {
        width: this.gridSize[0] * width,
        height: this.gridSize[1] * height
      };
    },
    style() {
      let { x, y, width, height } = this.item;

      return {
        left: this.gridSize[0] * x + 'px',
        top: this.gridSize[1] * y + 'px',
        width: this.gridSize[0] * width + 'px',
        height: this.gridSize[1] * height + 'px',
        minWidth: this.gridSize[0] * width + 'px',
        minHeight: this.gridSize[1] * height + 'px'
      };
    },
    inspectable() {
      return this.item.type === 'subobject-view' || this.item.type === 'telemetry-view';
    }
  },
  methods: {
    updatePosition(event) {
      let currentPosition = [event.pageX, event.pageY];
      this.initialPosition = this.initialPosition || currentPosition;
      this.delta = currentPosition.map(
        function (value, index) {
          return value - this.initialPosition[index];
        }.bind(this)
      );
    },
    startMove(event, posFactor = [1, 1], dimFactor = [0, 0]) {
      if (!this.isEditing) {
        return;
      }

      document.body.addEventListener('mousemove', this.continueMove);
      document.body.addEventListener('mouseup', this.endMove);
      this.dragPosition = {
        position: [this.item.x, this.item.y]
      };
      this.updatePosition(event);
      this.activeDrag = new LayoutDrag(this.dragPosition, posFactor, dimFactor, this.gridSize);
      event.preventDefault();
    },
    continueMove(event) {
      event.preventDefault();
      this.updatePosition(event);
      let newPosition = this.activeDrag.getAdjustedPosition(this.delta);

      if (!_.isEqual(newPosition, this.dragPosition)) {
        this.dragPosition = newPosition;
        this.$emit('move', this.toGridDelta(this.delta));
      }
    },
    endMove(event) {
      document.body.removeEventListener('mousemove', this.continueMove);
      document.body.removeEventListener('mouseup', this.endMove);
      this.continueMove(event);
      this.$emit('end-move');
      this.dragPosition = undefined;
      this.initialPosition = undefined;
      this.delta = undefined;
      event.preventDefault();
    },
    toGridDelta(pixelDelta) {
      return pixelDelta.map((v, i) => {
        return Math.round(v / this.gridSize[i]);
      });
    }
  }
};
</script>

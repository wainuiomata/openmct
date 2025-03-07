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
    aria-label="Clock Indicator"
    class="c-indicator t-indicator-clock icon-clock no-minify c-indicator--not-clickable"
    role="complementary"
  >
    <span class="label c-indicator__label">
      {{ timeTextValue }}
    </span>
  </div>
</template>

<script>
import moment from 'moment';
import raf from 'utils/raf';

export default {
  inject: ['openmct'],
  props: {
    indicatorFormat: {
      type: String,
      default: 'YYYY/MM/DD HH:mm:ss'
    }
  },
  data() {
    return {
      timestamp: this.openmct.time.getClock() ? this.openmct.time.now() : undefined
    };
  },
  computed: {
    timeTextValue() {
      return `${moment.utc(this.timestamp).format(this.indicatorFormat)} ${
        this.openmct.time.getTimeSystem().name
      }`;
    }
  },
  mounted() {
    this.tick = raf(this.tick);
    this.openmct.time.on('tick', this.tick);
    this.tick(this.timestamp);
  },
  beforeUnmount() {
    this.openmct.time.off('tick', this.tick);
  },
  methods: {
    tick(timestamp) {
      this.timestamp = timestamp;
    }
  }
};
</script>

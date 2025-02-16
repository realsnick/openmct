<!--
 Open MCT, Copyright (c) 2014-2023, United States Government
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
  <div class="c-toolbar">
    <div class="c-toolbar__element-controls">
      <component
        :is="item.control"
        v-for="(item, index) in primaryStructure"
        :key="index"
        :options="item"
        @change="updateObjectValue"
        @click="triggerMethod(item, $event)"
      />
    </div>
    <div class="c-toolbar__dimensions-and-controls">
      <component
        :is="item.control"
        v-for="(item, index) in secondaryStructure"
        :key="index"
        :options="item"
        @change="updateObjectValue"
        @click="triggerMethod(item, $event)"
      />
    </div>
  </div>
</template>

<script>
import _ from 'lodash';

import ToolbarButton from './components/ToolbarButton.vue';
import ToolbarCheckbox from './components/ToolbarCheckbox.vue';
import ToolbarColorPicker from './components/ToolbarColorPicker.vue';
import ToolbarInput from './components/ToolbarInput.vue';
import ToolbarMenu from './components/ToolbarMenu.vue';
import ToolbarSelectMenu from './components/ToolbarSelectMenu.vue';
import ToolbarSeparator from './components/ToolbarSeparator.vue';
import ToolbarToggleButton from './components/ToolbarToggleButton.vue';

export default {
  components: {
    ToolbarButton,
    ToolbarColorPicker,
    ToolbarCheckbox,
    ToolbarInput,
    ToolbarMenu,
    ToolbarSelectMenu,
    ToolbarSeparator,
    ToolbarToggleButton
  },
  inject: ['openmct'],
  data: function () {
    return {
      structure: []
    };
  },
  computed: {
    primaryStructure() {
      return this.structure.filter((item) => !item.secondary);
    },
    secondaryStructure() {
      return this.structure.filter((item) => item.secondary);
    }
  },
  mounted() {
    this.openmct.selection.on('change', this.handleSelection);
    this.handleSelection(this.openmct.selection.get());

    // Toolbars may change when edit mode is enabled/disabled, so listen
    // for edit mode changes and update toolbars if necessary.
    this.openmct.editor.on('isEditing', this.handleEditing);
  },
  unmounted() {
    this.openmct.selection.off('change', this.handleSelection);
    this.openmct.editor.off('isEditing', this.handleEditing);
    this.removeListeners();
  },
  methods: {
    handleSelection(selection) {
      this.removeListeners();
      this.domainObjectsById = {};

      if (selection.length === 0 || !selection[0][0]) {
        this.structure = [];

        return;
      }

      let structure = this.openmct.toolbars.get(selection) || [];
      this.structure = structure.map((toolbarItem) => {
        let domainObject = toolbarItem.domainObject;
        let formKeys = [];
        toolbarItem.control = 'toolbar-' + toolbarItem.control;

        if (toolbarItem.dialog) {
          toolbarItem.dialog.sections.forEach((section) => {
            section.rows.forEach((row) => {
              formKeys.push(row.key);
            });
          });
          toolbarItem.formKeys = formKeys;
        }

        if (domainObject) {
          toolbarItem.value = this.getValue(domainObject, toolbarItem);
          this.registerListener(domainObject);
        }

        return toolbarItem;
      });
    },
    registerListener(domainObject) {
      let id = this.openmct.objects.makeKeyString(domainObject.identifier);

      if (!this.domainObjectsById[id]) {
        this.domainObjectsById[id] = {
          domainObject: domainObject
        };
        this.observeObject(domainObject, id);
      }
    },
    observeObject(domainObject, id) {
      let unobserveObject = this.openmct.objects.observe(
        domainObject,
        '*',
        function (newObject) {
          this.domainObjectsById[id].newObject = newObject;
          this.updateToolbarAfterMutation();
        }.bind(this)
      );
      this.unObserveObjects.push(unobserveObject);
    },
    updateToolbarAfterMutation() {
      this.structure = this.structure.map((toolbarItem) => {
        let domainObject = toolbarItem.domainObject;

        if (domainObject) {
          let id = this.openmct.objects.makeKeyString(domainObject.identifier);
          let newObject = this.domainObjectsById[id].newObject;

          if (newObject) {
            toolbarItem.domainObject = newObject;
            toolbarItem.value = this.getValue(newObject, toolbarItem);
          }
        }

        return toolbarItem;
      });

      Object.values(this.domainObjectsById).forEach(function (tracker) {
        if (tracker.newObject) {
          tracker.domainObject = tracker.newObject;
          delete tracker.newObject;
        }
      });
    },
    getValue(domainObject, toolbarItem) {
      let value = undefined;
      let applicableSelectedItems = toolbarItem.applicableSelectedItems;

      if (!applicableSelectedItems && !toolbarItem.property) {
        return value;
      }

      if (toolbarItem.formKeys) {
        value = this.getFormValue(domainObject, toolbarItem);
      } else {
        let values = [];
        if (applicableSelectedItems) {
          applicableSelectedItems.forEach((selectionPath) => {
            values.push(this.getPropertyValue(domainObject, toolbarItem, selectionPath));
          });
        } else {
          values.push(this.getPropertyValue(domainObject, toolbarItem));
        }

        // If all values are the same, use it, otherwise mark the item as non-specific.
        if (values.every((val) => val === values[0])) {
          value = values[0];
          toolbarItem.nonSpecific = false;
        } else {
          toolbarItem.nonSpecific = true;
        }
      }

      return value;
    },
    getPropertyValue(domainObject, toolbarItem, selectionPath, formKey) {
      let property = this.getItemProperty(toolbarItem, selectionPath);

      if (formKey) {
        property = property + '.' + formKey;
      }

      return _.get(domainObject, property);
    },
    getFormValue(domainObject, toolbarItem) {
      let value = {};
      let values = {};

      toolbarItem.formKeys.forEach((key) => {
        values[key] = [];

        if (toolbarItem.applicableSelectedItems) {
          toolbarItem.applicableSelectedItems.forEach((selectionPath) => {
            values[key].push(this.getPropertyValue(domainObject, toolbarItem, selectionPath, key));
          });
        } else {
          values[key].push(this.getPropertyValue(domainObject, toolbarItem, undefined, key));
        }
      });

      for (let key in values) {
        if (values[key].every((val) => val === values[key][0])) {
          value[key] = values[key][0];
          toolbarItem.nonSpecific = false;
        } else {
          toolbarItem.nonSpecific = true;

          return {};
        }
      }

      return value;
    },
    getItemProperty(item, selectionPath) {
      return typeof item.property === 'function' ? item.property(selectionPath) : item.property;
    },
    removeListeners() {
      if (this.unObserveObjects) {
        this.unObserveObjects.forEach((unObserveObject) => {
          unObserveObject();
        });
      }

      this.unObserveObjects = [];
    },
    updateObjectValue(value, item) {
      let changedItemId = this.openmct.objects.makeKeyString(item.domainObject.identifier);
      this.structure = this.structure.map((toolbarItem) => {
        if (toolbarItem.domainObject) {
          let id = this.openmct.objects.makeKeyString(toolbarItem.domainObject.identifier);

          if (changedItemId === id && _.isEqual(toolbarItem, item)) {
            toolbarItem.value = value;
          }
        }

        return toolbarItem;
      });

      // If value is an object, iterate the toolbar structure and mutate all keys in form.
      // Otherwise, mutate the property.
      if (value === Object(value)) {
        this.structure.forEach((s) => {
          if (s.formKeys) {
            s.formKeys.forEach((key) => {
              if (item.applicableSelectedItems) {
                item.applicableSelectedItems.forEach((selectionPath) => {
                  this.mutateObject(item, value[key], selectionPath, key);
                });
              } else {
                this.mutateObject(item, value[key], undefined, key);
              }
            });
          }
        });
      } else {
        if (item.applicableSelectedItems) {
          item.applicableSelectedItems.forEach((selectionPath) => {
            this.mutateObject(item, value, selectionPath);
          });
        } else {
          this.mutateObject(item, value);
        }
      }
    },
    mutateObject(item, value, selectionPath, formKey) {
      let property = this.getItemProperty(item, selectionPath);

      if (formKey) {
        property = property + '.' + formKey;
      }

      this.openmct.objects.mutate(item.domainObject, property, value);
    },
    triggerMethod(item, event) {
      if (item.method) {
        item.method({ ...event });
      }
    },
    handleEditing(isEditing) {
      this.handleSelection(this.openmct.selection.get());
    }
  }
};
</script>

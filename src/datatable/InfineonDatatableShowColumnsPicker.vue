<template>
  <div>
    <TreeSelect
      id="showColumnSelect"
      :model-value="shownColumns"
      :multiple="true"
      :options="columnOptions"
      :limit="0"
      placeholder="Click here to show columns"
      :limit-text="(count) => `show ${count +
        columns.length - columnOptions.length} / ${columns.length} columns`"
      :clearable="false"
      @select="changeColumnVisibility({value: $event})"
      @deselect="changeColumnVisibility({value: $event})"
    />
  </div>
</template>

<script setup>
import { TreeSelect } from '@/plugins/treeView';
import {
  toRefs, computed,
} from 'vue';

const props = defineProps({
  columns: { type: Array, default: () => [] },
  hiddenColumnKeys: { type: Array, default: () => [] },
});

const {
  columns, hiddenColumnKeys,
} = toRefs(props);

const emit = defineEmits(['changeColumnVisibility']);

const shownColumns = computed(() => columns.value
  .filter((c) => !hiddenColumnKeys.value.includes(c.key) && c.hidable).map((c) => c.key));
const columnOptions = computed(() => columns.value.filter((c) => c.hidable)
  .map((c) => ({ id: c.key, label: c.title, isDisabled: !c.hidable })));

function changeColumnVisibility(columnKey) {
  emit('changeColumnVisibility', columnKey.value.id);
}

</script>

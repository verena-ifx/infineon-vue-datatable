<template>
  <div
    class="d-flex flex-column justify-content-center flex-grow-1 pt-3"
    style="overflow:auto"
  >
    <div
      class="flex-grow-1 table-responsive"
      style="overflow:auto"
    >
      <table
        class="table table-sm table-hover"
        style="border-collapse: separate;border-spacing: 0;"
      >
        <thead>
          <tr>
            <th
              v-if="canEdit"
              class="stickyHeader"
            >
              Actions
            </th>
            <th
              v-for="(column, index) in shownColumns"
              :key="index"
              class="stickyHeader text-nowrap"
              scope="col"
            >
              {{ column.title }}
              <DatatableSortIcon
                v-model:sort-column="sortColumn"
                :column="column"
              />
              <a
                v-if="column.hidable"
                href="#"
                @click="changeColumnVisibility(column.key)"
              >
                <font-awesome-icon
                  class="fa-sm ms-2"
                  :icon="['fas', 'times']"
                />
              </a>
            </th>
          </tr>
        </thead>

        <tbody>
          <DatatableRow

            v-for="(row,idx) in processedData"
            :key="row.id"
            :row-index="idx"

            :row="row"
            :columns="realColumns"
            :hidden-column-keys="hiddenColumnKeys"
            :row-is-in-edit-mode="(row.id) === (rowInEditMode?.id)"
            :can-edit="canEdit"
            :additional-actions="additionalActions"
            @startEditRow="startEditRow"
            @saveRow="saveRow"
            @cancelRow="cancelRow"
          >
            <template
              v-for="(_, name) in $slots"
              #[name]="slotData"
            >
              <slot
                :name="name"
                v-bind="slotData || {}"
              />
            </template>
          </DatatableRow>
        </tbody>
      </table>
    </div>
    <div class="mt-1 d-flex flex-row">
      <DatatablePager
        v-model:currentPage="currentPage"
        class="flex-grow-1"
        :page-size="pageSize"
        :count="count"
        @updatePageSize="updatePageSize"
      />
      <DatatableShowColumnsPicker
        style="max-width:15em"
        :columns="realColumns"
        :hidden-column-keys="hiddenColumnKeys"
        @changeColumnVisibility="changeColumnVisibility"
      />
      <div
        v-if="exportable"
        class="mt-1 ms-1"
      >
        <button
          class="btn btn-sm btn-primary"
          title="Download CSV File"
          @click="exportCSV"
        >
          <font-awesome-icon :icon="['fas', 'file-download']" />
          Download
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
// style="max-width: 13em"
import {
  toRefs, computed, ref, onMounted, watch,
} from 'vue';
import exportFromJSON from 'export-from-json';

// import json2Csv from 'json2csv/dist/json2csv.umd';
import DatatableRow from './InfineonDatatableRow.vue';
import DatatablePager from './InfineonDatatablePager.vue';
import DatatableSortIcon from './InfineonDatatableSortIcon.vue';
import DatatableShowColumnsPicker from './InfineonDatatableShowColumnsPicker.vue';

const props = defineProps({
  canEdit: Boolean,
  data: { type: Array, default: () => [] },
  columns: { type: Array, default: () => [] },
  exportable: Boolean,
  defaultSort: { type: Object, default: () => {} }, // {key: '', type: 'A/D'}
  additionalActions: { type: Object, default: () => {} },
  // [ { label: '', action: (row) => {}, icon: ['fas', 'list-ol'] } ]
});
const emit = defineEmits(['saveRow']);

const { data, columns } = toRefs(props);

const sortColumn = ref(props.defaultSort);
const currentPage = ref(0);
const pageSize = ref(10);
const rowInEditMode = ref(undefined);
const count = ref(0);

const hiddenColumnKeys = ref([]);

const realColumns = computed(() => columns.value
  .filter((c) => c.visible === undefined || c.visible === false));

const shownColumns = computed(() => realColumns.value
  .filter((c) => !c.hidable || !hiddenColumnKeys.value.includes(c.key)));

const exportType = exportFromJSON.types.csv;
const file = 'download';

// reset page & item count when data changes
watch(
  data,
  (newData) => {
    count.value = newData ? newData.length : 0;
    if (count.value / pageSize.value < currentPage.value) {
      currentPage.value = parseInt(count.value / pageSize.value, 10);
    }
  },
  { immediate: true },
);

const processedData = computed(() => {
  const sortedData = data.value.slice(0);
  if (sortColumn.value && sortColumn.value.key) {
    const { key, type } = sortColumn.value;
    const {
      sortType,
      valueResolver,
      filterResolverKey,
    } = realColumns.value.find((c) => c.key === key);

    if (filterResolverKey || !valueResolver) {
      const vKey = filterResolverKey || key;

      if (sortType === 'NUMBER' && type === 'D') {
        sortedData.sort((a, b) => b[vKey] - a[vKey]);
      } else if (sortType === 'NUMBER' && type === 'A') {
        sortedData.sort((a, b) => a[vKey] - b[vKey]);
      } else if (sortType === 'STRING' && type === 'D') {
        sortedData.sort((a, b) => b[vKey]?.localeCompare(a[vKey]));
      } else if (sortType === 'STRING' && type === 'A') {
        sortedData.sort((a, b) => a[vKey]?.localeCompare(b[vKey]));
      }
    } else if (valueResolver) {
      if (sortType === 'NUMBER' && type === 'D') {
        sortedData.sort((a, b) => valueResolver(b) - valueResolver(a));
      } else if (sortType === 'NUMBER' && type === 'A') {
        sortedData.sort((a, b) => valueResolver(a) - valueResolver(b));
      } else if (sortType === 'STRING' && type === 'D') {
        sortedData.sort((a, b) => valueResolver(b)?.localeCompare(valueResolver(a)));
      } else if (sortType === 'STRING' && type === 'A') {
        sortedData.sort((a, b) => valueResolver(a)?.localeCompare(valueResolver(b)));
      }
    }
  }

  const sliced = sortedData.slice(
    currentPage.value * pageSize.value,
    (currentPage.value + 1) * pageSize.value,
  );

  return sliced;
});

// we listen to column changes. so each table will have it's own storage key
const hiddenColumnsLocalStorageKey = computed(() => realColumns
  .value.map((c) => c.key).sort().join());
onMounted(() => {
  if (localStorage.getItem(hiddenColumnsLocalStorageKey.value)) {
    try {
      hiddenColumnKeys.value = JSON
        .parse(localStorage.getItem(hiddenColumnsLocalStorageKey.value));
    } catch (e) {
      localStorage.removeItem(hiddenColumnsLocalStorageKey.value);
    }
  } else {
    hiddenColumnKeys.value = realColumns.value.filter((c) => c.defaultHidden).map((c) => c.key);
  }
});

function changeColumnVisibility(columnKey) {
  if (hiddenColumnKeys.value.includes(columnKey)) {
    const index = hiddenColumnKeys.value.indexOf(columnKey);
    hiddenColumnKeys.value.splice(index, 1);
  } else {
    hiddenColumnKeys.value.push(columnKey);
  }
  localStorage.setItem(hiddenColumnsLocalStorageKey.value, JSON.stringify(hiddenColumnKeys.value));
}

function startEditRow(row) {
  rowInEditMode.value = row ? { ...row } : undefined;
}
async function saveRow(row) {
  emit('saveRow', row);
  rowInEditMode.value = undefined;
}
function cancelRow() {
  rowInEditMode.value = undefined;
}
function updatePageSize(size) {
  pageSize.value = size;
}

async function exportCSV() {
  const fields = shownColumns.value.map((col) => ({ [col.key]: col.title }));
  const transforms = (row) => Object.fromEntries(
    shownColumns.value.map((col) => [
      col.key,
      col.valueResolver ? col.valueResolver(row) : row[col.key],
    ]),
  );

  // const opts = { fields, transforms };

  // json2Csv.parseAsync(data.value, opts)
  //   .then((csv) => {
  //     const sep = 'sep=,\r\n';
  //     const BOM = new Uint8Array([0xEF, 0xBB, 0xBF]);
  //     const blob = new Blob([BOM, csv], { type: 'text/csv;charset=utf-8' });
  //     const link = document.createElement('a');
  //     link.href = URL.createObjectURL(blob);
  //     link.download = 'download.csv';
  //     link.click();
  //     URL.revokeObjectURL(link.href);
  //   }).catch((err) => console.log('Error downloading file!'));

  // const data = exportData;

  exportFromJSON({
    data: data.value, file, fields: fields.value, exportType,
  });
}
</script>

<style scoped>
.oddRow {
  --bs-table-accent-bg: var(--bs-table-striped-bg);
  color: var(--bs-table-striped-color);
}

.stickyHeader {
  position:sticky;
  top:0;
  border-bottom: 2px solid black;
  background-color: white
}
</style>

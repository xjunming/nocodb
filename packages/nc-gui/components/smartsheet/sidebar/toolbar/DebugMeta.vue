<script setup lang="ts">
const editorOpen = ref(false)

const tabKey = ref()

const { metas } = $(useMetas())

const { tables } = useTable()

const localTables = tables.value.filter((t) => metas[t.id as string])
</script>

<template>
  <a-tooltip placement="bottom">
    <template #title>
      <span> Debug Meta </span>
    </template>
    <mdi-bug-outline class="cursor-pointer" @click="editorOpen = true" />
  </a-tooltip>

  <a-modal v-model:visible="editorOpen" :footer="null" width="80%">
    <a-tabs v-model:activeKey="tabKey" type="card" closeable="false" class="shadow-sm">
      <a-tab-pane v-for="table in localTables" :key="table.id" :tab="table.title">
        <MonacoEditor v-model="metas[table.id]" class="h-max-[70vh]" :read-only="true" />
      </a-tab-pane>
    </a-tabs>
  </a-modal>
</template>

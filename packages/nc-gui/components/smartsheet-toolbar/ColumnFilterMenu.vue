<script setup lang="ts">
import { useNuxtApp } from 'nuxt/app'
import type ColumnFilter from './ColumnFilter.vue'
import { ActiveViewInj, IsLockedInj, IsPublicInj, computed, inject, ref, useGlobal, useViewFilters } from '#imports'

const isLocked = inject(IsLockedInj, ref(false))

const activeView = inject(ActiveViewInj)

const isPublic = inject(IsPublicInj, ref(false))

const { filterAutoSave } = useGlobal()

const filterComp = ref<typeof ColumnFilter>()

const { $e } = useNuxtApp()

const { nestedFilters } = useSmartsheetStoreOrThrow()
// todo: avoid duplicate api call by keeping a filter store
const { filters, loadFilters } = useViewFilters(
  activeView,
  undefined,
  computed(() => true),
  () => false,
  nestedFilters.value,
  true,
)

const filtersLength = ref(0)

watch(
  () => activeView?.value,
  async () => {
    await loadFilters()
    filtersLength.value = filters.value.length || 0
  },
  { immediate: true },
)

const applyChanges = async () => await filterComp.value?.applyChanges()

const filterAutoSaveLoc = computed({
  get() {
    return filterAutoSave.value
  },
  set(val) {
    $e('a:filter:auto-apply', { flag: val })
    filterAutoSave.value = val
  },
})
</script>

<template>
  <a-dropdown :trigger="['click']">
    <div :class="{ 'nc-badge nc-active-btn': filtersLength }">
      <a-button v-t="['c:filter']" class="nc-filter-menu-btn nc-toolbar-btn txt-sm" :disabled="isLocked">
        <div class="flex items-center gap-1">
          <MdiFilterOutline />
          <!-- Filter -->
          <span class="text-capitalize !text-sm font-weight-normal">{{ $t('activity.filter') }}</span>
          <MdiMenuDown class="text-grey" />
        </div>
      </a-button>
    </div>
    <template #overlay>
      <SmartsheetToolbarColumnFilter
        ref="filterComp"
        class="nc-table-toolbar-menu shadow-lg"
        :auto-save="filterAutoSave"
        @update:filters-length="filtersLength = $event"
      >
        <div v-if="!isPublic" class="flex items-end mt-2 min-h-[30px]" @click.stop>
          <a-checkbox id="col-filter-checkbox" v-model:checked="filterAutoSaveLoc" class="col-filter-checkbox" hide-details dense>
            <span class="text-grey text-xs">
              {{ $t('msg.info.filterAutoApply') }}
              <!-- Auto apply -->
            </span>
          </a-checkbox>

          <div class="flex-1" />
          <a-button
            v-show="!filterAutoSave"
            v-t="['a:filter:auto-apply']"
            size="small"
            class="text-xs ml-2"
            @click="applyChanges"
          >
            Apply changes
          </a-button>
        </div>
      </SmartsheetToolbarColumnFilter>
    </template>
  </a-dropdown>
</template>

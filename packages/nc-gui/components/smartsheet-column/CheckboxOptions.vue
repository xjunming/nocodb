<script setup lang="ts">
import { getMdiIcon } from '@/utils'

interface Props {
  value: Record<string, any>
}

const props = defineProps<Props>()
const emit = defineEmits(['update:value'])
const vModel = useVModel(props, 'value', emit)

// cater existing v1 cases
const iconList = [
  {
    checked: 'mdi-check-bold',
    unchecked: 'mdi-crop-square',
  },
  {
    checked: 'mdi-check-circle-outline',
    unchecked: 'mdi-checkbox-blank-circle-outline',
  },
  {
    checked: 'mdi-star',
    unchecked: 'mdi-star-outline',
  },
  {
    checked: 'mdi-heart',
    unchecked: 'mdi-heart-outline',
  },
  {
    checked: 'mdi-moon-full',
    unchecked: 'mdi-moon-new',
  },
  {
    checked: 'mdi-thumb-up',
    unchecked: 'mdi-thumb-up-outline',
  },
  {
    checked: 'mdi-flag',
    unchecked: 'mdi-flag-outline',
  },
]

const picked = computed({
  get: () => vModel.value.meta.color,
  set: (val) => {
    vModel.value.meta.color = val
  },
})

// set default value
vModel.value.meta = {
  icon: {
    checked: 'mdi-check-bold',
    unchecked: 'mdi-crop-square',
  },
  color: '#777',
  ...vModel.value.meta,
}

// antdv doesn't support object as value
// use iconIdx as value and update back in watch
const iconIdx = iconList.findIndex(
  (ele) => ele.checked === vModel.value.meta.icon.checked && ele.unchecked === vModel.value.meta.icon.unchecked,
)

vModel.value.meta.iconIdx = iconIdx === -1 ? 0 : iconIdx

watch(
  () => vModel.value.meta.iconIdx,
  (v) => {
    vModel.value.meta.icon = iconList[v]
  },
)
</script>

<template>
  <a-row>
    <a-col :span="24">
      <a-form-item label="Icon">
        <a-select v-model:value="vModel.meta.iconIdx" class="w-52">
          <a-select-option v-for="(icon, i) of iconList" :key="i" :value="i">
            <div class="flex items-center">
              <component
                :is="getMdiIcon(icon.checked)"
                class="mx-1"
                :style="{
                  color: vModel.meta.color,
                }"
              />
              <component
                :is="getMdiIcon(icon.unchecked)"
                :style="{
                  color: vModel.meta.color,
                }"
              />
            </div>
          </a-select-option>
        </a-select>
      </a-form-item>
    </a-col>
  </a-row>
  <a-row class="w-full justify-center">
    <GeneralColorPicker
      v-model="picked"
      :row-size="8"
      :colors="['#fcb401', '#faa307', '#f48c06', '#e85d04', '#dc2f02', '#d00000', '#9d0208', '#777']"
    />
  </a-row>
</template>

<style scoped lang="scss">
.color-selector:hover {
  @apply brightness-90;
}

.color-selector.selected {
  @apply py-[5px] px-[10px] brightness-90;
}
</style>

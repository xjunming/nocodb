<script setup lang="ts">
import MdiPlusIcon from '~icons/mdi/plus'
import MdiDeleteOutlineIcon from '~icons/mdi/delete-outline'

interface Props {
  modelValue: Record<string, any>[]
}

const props = defineProps<Props>()

const emits = defineEmits(['update:modelValue'])

const vModel = useVModel(props, 'modelValue', emits)

const addParamRow = () => vModel.value.push({})

const deleteParamRow = (idx: number) => vModel.value.splice(idx, 1)
</script>

<template>
  <div class="flex flex-row justify-center">
    <table>
      <thead>
        <tr>
          <th>
            <!-- Intended to be empty - For checkbox -->
          </th>
          <th>
            <div class="text-center font-normal mb-2">Param Name</div>
          </th>
          <th>
            <div class="text-center font-normal mb-2">Value</div>
          </th>
          <th>
            <!-- Intended to be empty - For delete button -->
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(paramRow, idx) in vModel" :key="idx">
          <td class="px-2">
            <a-form-item>
              <a-checkbox v-model:checked="paramRow.enabled" />
            </a-form-item>
          </td>
          <td class="px-2">
            <a-form-item>
              <a-input v-model:value="paramRow.name" size="large" placeholder="Key" />
            </a-form-item>
          </td>
          <td class="px-2">
            <a-form-item>
              <a-input v-model:value="paramRow.value" size="large" placeholder="Value" />
            </a-form-item>
          </td>
          <td class="relative">
            <div v-if="idx !== 0" class="absolute flex flex-col justify-start mt-2 -right-6 top-0">
              <MdiDeleteOutlineIcon class="cursor-pointer" @click="deleteParamRow(idx)" />
            </div>
          </td>
        </tr>
        <tr>
          <td :colspan="12" class="text-center">
            <a-button type="default" class="!bg-gray-100 rounded-md border-none mr-1" @click="addParamRow">
              <template #icon>
                <MdiPlusIcon class="flex mx-auto" />
              </template>
            </a-button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

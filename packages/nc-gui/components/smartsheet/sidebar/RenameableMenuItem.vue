<script lang="ts" setup>
import type { ViewTypes } from 'nocodb-sdk'
import { message } from 'ant-design-vue'
import { viewIcons } from '~/utils'
import { IsLockedInj, onKeyStroke, useDebounceFn, useNuxtApp, useUIPermission, useVModel } from '#imports'

interface Props {
  view: Record<string, any>
  onValidate: (view: Record<string, any>) => boolean | string
}

interface Emits {
  (event: 'update:view', data: Record<string, any>): void
  (event: 'changeView', view: Record<string, any>): void
  (event: 'rename', view: Record<string, any>): void
  (event: 'delete', view: Record<string, any>): void
  (event: 'openModal', data: { type: ViewTypes; title?: string; copyViewId?: string }): void
}

const props = defineProps<Props>()

const emits = defineEmits<Emits>()

const vModel = useVModel(props, 'view', emits)

const { $e } = useNuxtApp()

const { isUIAllowed } = useUIPermission()

const isLocked = inject(IsLockedInj)

/** Is editing the view name enabled */
let isEditing = $ref<boolean>(false)

/** Helper to check if editing was disabled before the view navigation timeout triggers */
let isStopped = $ref(false)

/** Original view title when editing the view name */
let originalTitle = $ref<string | undefined>()

/** Debounce click handler, so we can potentially enable editing view name {@see onDblClick} */
const onClick = useDebounceFn(() => {
  if (isEditing || isStopped) return

  emits('changeView', vModel.value)
}, 250)

/** Enable editing view name on dbl click */
function onDblClick() {
  if (!isEditing) {
    isEditing = true
    originalTitle = vModel.value.title
    $e('c:view:rename', { view: vModel.value?.type })
  }
}

/** Handle keydown on input field */
function onKeyDown(event: KeyboardEvent) {
  if (event.key === 'Escape') {
    onKeyEsc(event)
  } else if (event.key === 'Enter') {
    onKeyEnter(event)
  }
}

/** Rename view when enter is pressed */
function onKeyEnter(event: KeyboardEvent) {
  event.stopImmediatePropagation()
  event.preventDefault()

  onRename()
}

/** Disable renaming view when escape is pressed */
function onKeyEsc(event: KeyboardEvent) {
  event.stopImmediatePropagation()
  event.preventDefault()

  onCancel()
}

onKeyStroke('Enter', (event) => {
  if (isEditing) {
    onKeyEnter(event)
  }
})

function focusInput(el: HTMLInputElement) {
  if (el) el.focus()
}

/** Duplicate a view */
// todo: This is not really a duplication, maybe we need to implement a true duplication?
function onDuplicate() {
  emits('openModal', { type: vModel.value.type, title: vModel.value.title, copyViewId: vModel.value.id })

  $e('c:view:copy', { view: vModel.value.type })
}

/** Delete a view */
async function onDelete() {
  emits('delete', vModel.value)
}

/** Rename a view */
async function onRename() {
  if (!isEditing) return

  const isValid = props.onValidate(vModel.value)

  if (isValid !== true) {
    message.error(isValid)

    onCancel()
    return
  }

  if (vModel.value.title === '' || vModel.value.title === originalTitle) {
    onCancel()
    return
  }

  emits('rename', vModel.value)

  onStopEdit()
}

/** Cancel renaming view */
function onCancel() {
  if (!isEditing) return

  vModel.value.title = originalTitle
  onStopEdit()
}

/** Stop editing view name, timeout makes sure that view navigation (click trigger) does not pick up before stop is done */
function onStopEdit() {
  isStopped = true
  isEditing = false
  originalTitle = ''

  setTimeout(() => {
    isStopped = false
  }, 250)
}
</script>

<template>
  <a-menu-item
    class="select-none group !flex !items-center !my-0 hover:(bg-primary !bg-opacity-5)"
    @dblclick.stop="isUIAllowed('virtualViewsCreateOrEdit') && onDblClick()"
    @click.stop="onClick"
  >
    <div v-t="['a:view:open', { view: vModel.type }]" class="text-xs flex items-center w-full gap-2">
      <div class="flex w-auto">
        <MdiDrag
          class="nc-drag-icon hidden group-hover:block transition-opacity opacity-0 group-hover:opacity-100 text-gray-500 !cursor-move"
          @click.stop.prevent
        />

        <component
          :is="viewIcons[vModel.type].icon"
          class="nc-view-icon group-hover:hidden"
          :style="{ color: viewIcons[vModel?.type]?.color }"
        />
      </div>

      <a-input v-if="isEditing" :ref="focusInput" v-model:value="vModel.title" @blur="onCancel" @keydown="onKeyDown($event)" />

      <div v-else>
        <GeneralTruncateText>{{ vModel.alias || vModel.title }}</GeneralTruncateText>
      </div>

      <div class="flex-1" />

      <template v-if="!isEditing && !isLocked && isUIAllowed('virtualViewsCreateOrEdit')">
        <div class="flex items-center gap-1">
          <a-tooltip placement="left">
            <template #title>
              {{ $t('activity.copyView') }}
            </template>

            <MdiContentCopy class="hidden group-hover:block text-gray-500" @click.stop="onDuplicate" />
          </a-tooltip>

          <template v-if="!vModel.is_default">
            <a-tooltip placement="left">
              <template #title>
                {{ $t('activity.deleteView') }}
              </template>

              <MdiTrashCan class="hidden group-hover:block text-red-500 nc-view-delete-icon" @click.stop="onDelete" />
            </a-tooltip>
          </template>
        </div>
      </template>
    </div>
  </a-menu-item>
</template>

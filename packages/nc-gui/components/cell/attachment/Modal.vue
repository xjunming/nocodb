<script lang="ts" setup>
import { onKeyDown } from '@vueuse/core'
import { useAttachmentCell } from './utils'
import { useSortable } from './sort'
import { isImage, openLink, ref, useDropZone, useUIPermission, watch } from '#imports'

const { isUIAllowed } = useUIPermission()

const {
  open,
  isLoading,
  isPublicGrid,
  isReadonly,
  visibleItems,
  modalVisible,
  column,
  FileIcon,
  removeFile,
  onDrop,
  downloadFile,
  updateModelValue,
  selectedImage,
} = useAttachmentCell()!

// todo: replace placeholder var
const isLocked = ref(false)

const dropZoneRef = ref<HTMLDivElement>()

const sortableRef = ref<HTMLDivElement>()

const { dragging } = useSortable(sortableRef, visibleItems, updateModelValue, isReadonly)

const { isOverDropZone } = useDropZone(dropZoneRef, onDrop)

const { isSharedForm } = useSmartsheetStoreOrThrow()

onKeyDown('Escape', () => {
  modalVisible.value = false
  isOverDropZone.value = false
})

function onClick(item: Record<string, any>) {
  selectedImage.value = item
  modalVisible.value = false

  const stopHandle = watch(selectedImage, (nextImage, _, onCleanup) => {
    if (!nextImage) {
      setTimeout(() => {
        modalVisible.value = true
      }, 50)
      stopHandle?.()
    }

    onCleanup(() => stopHandle?.())
  })
}
</script>

<template>
  <a-modal v-model:visible="modalVisible" class="nc-attachment-modal" width="80%" :footer="null">
    <template #title>
      <div class="flex gap-4">
        <div
          v-if="isSharedForm || (!isReadonly && isUIAllowed('tableAttachment') && !isPublicGrid && !isLocked)"
          class="nc-attach-file group"
          @click="open"
        >
          <MaterialSymbolsAttachFile class="transform group-hover:(text-accent scale-120)" />
          Attach File
        </div>

        <div class="flex items-center gap-2">
          <div v-if="isReadonly" class="text-gray-400">[Readonly]</div>
          Viewing Attachments of
          <div class="font-semibold underline">{{ column.title }}</div>
        </div>
      </div>
    </template>

    <div ref="dropZoneRef">
      <template v-if="isSharedForm || (!isReadonly && !dragging)">
        <general-overlay
          v-model="isOverDropZone"
          inline
          class="text-white ring ring-accent bg-gray-700/75 flex items-center justify-center gap-2 backdrop-blur-xl"
        >
          <MaterialSymbolsFileCopyOutline class="text-accent" height="35" width="35" />
          <div class="text-white text-3xl">Drop here</div>
        </general-overlay>
      </template>

      <div ref="sortableRef" :class="{ dragging }" class="grid grid-cols-2 md:grid-cols-3 xl:grid-cols-4 gap-6 relative p-6">
        <div v-for="(item, i) of visibleItems" :key="`${item.title}-${i}`" class="flex flex-col gap-1">
          <a-card class="nc-attachment-item group">
            <a-tooltip v-if="!isReadonly">
              <template #title> Remove File </template>
              <MdiCloseCircle
                v-if="isSharedForm || (isUIAllowed('tableAttachment') && !isPublicGrid && !isLocked)"
                class="nc-attachment-remove"
                @click.stop="removeFile(i)"
              />
            </a-tooltip>

            <a-tooltip placement="bottom">
              <template #title> Download file </template>

              <div class="nc-attachment-download group-hover:(opacity-100)">
                <MdiDownload @click.stop="downloadFile(item)" />
              </div>
            </a-tooltip>

            <div
              :class="[dragging ? 'cursor-move' : 'cursor-pointer']"
              class="nc-attachment h-full w-full flex items-center justify-center"
            >
              <div
                v-if="isImage(item.title, item.mimetype)"
                :style="{ backgroundImage: `url('${item.url || item.data}')` }"
                class="w-full h-full bg-contain bg-center bg-no-repeat"
                @click.stop="onClick(item)"
              />

              <component
                :is="FileIcon(item.icon)"
                v-else-if="item.icon"
                height="150"
                width="150"
                @click.stop="openLink(item.url || item.data)"
              />

              <IcOutlineInsertDriveFile v-else height="150" width="150" @click.stop="openLink(item.url || item.data)" />
            </div>
          </a-card>

          <div class="truncate" :title="item.title">
            {{ item.title }}
          </div>
        </div>

        <div v-if="isLoading" class="flex flex-col gap-1">
          <a-card class="nc-attachment-item group">
            <div class="nc-attachment h-full w-full flex items-center justify-center">
              <a-skeleton-image class />
            </div>
          </a-card>
        </div>
      </div>
    </div>
  </a-modal>
</template>

<style lang="scss">
.nc-attachment-modal {
  .nc-attach-file {
    @apply select-none cursor-pointer color-transition flex items-center gap-1 border-1 p-2 rounded
    @apply hover:(bg-primary bg-opacity-10 text-primary ring);
    @apply active:(ring-accent bg-primary bg-opacity-20);
  }

  .nc-attachment-item {
    @apply !h-2/3 !min-h-[200px] flex items-center justify-center relative;

    @supports (-moz-appearance: none) {
      @apply hover:border-0;
    }

    &::after {
      @apply pointer-events-none rounded absolute top-0 left-0 right-0 bottom-0 transition-all duration-150 ease-in-out;
      content: '';
    }

    @supports (-moz-appearance: none) {
      &:hover::after {
        @apply ring shadow transform scale-103;
      }

      &:active::after {
        @apply ring ring-accent shadow transform scale-103;
      }
    }
  }

  .nc-attachment-download {
    @apply bg-white absolute bottom-2 right-2;
    @apply transition-opacity duration-150 ease-in opacity-0 hover:ring;
    @apply cursor-pointer rounded shadow flex items-center p-1 border-1;
    @apply active:(ring border-0 ring-accent);
  }

  .nc-attachment-remove {
    @apply absolute top-2 right-2 bg-white;
    @apply hover:(ring ring-red-500);
    @apply cursor-pointer rounded-full border-2;
    @apply active:(ring border-0 ring-red-500);
  }

  .ant-card-body {
    @apply !p-2 w-full h-full;
  }

  .ant-modal-body {
    @apply !p-0;
  }

  .ghost,
  .ghost > * {
    @apply !pointer-events-none;
  }

  .dragging {
    .nc-attachment-item {
      @apply !pointer-events-none;
    }

    .ant-tooltip {
      @apply !hidden;
    }
  }
}
</style>

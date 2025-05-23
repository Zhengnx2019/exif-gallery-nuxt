<script setup lang="ts">
import Viewer from 'viewerjs'
import 'viewerjs/dist/viewer.css'

const props = defineProps<{
  id: number
  file: File
  compressFile?: compressFiles
  aiLoading?: boolean
  uploadLoading?: boolean
}>()

const emit = defineEmits<{
  upload: []
  generate: []
  close: []
}>()

const { config: uploadConfig } = useUploadConfig()
const { config: aiConfig } = useAIConfig()
const { loggedIn } = useUserSession()

const photo = defineModel<IPhotoForm>({ required: true })
const activeId = defineModel<number>('activeId')
const isOpen = computed({
  get: () => activeId.value === props.id,
  set: value => value ? activeId.value = props.id : activeId.value = undefined,
})

const compressLoading = computed(() =>
  props.compressFile?.jpeg === 'loading'
  || props.compressFile?.webp === 'loading'
  || props.compressFile?.avif === 'loading'
  || props.compressFile?.thumbnail === 'loading',
)

let viewer: Viewer | undefined
const viewerRef = ref<HTMLElement>()

onMounted(() => {
  if (!viewerRef.value)
    return
  viewer = new Viewer(viewerRef.value, {
    transition: false,
    toolbar: {
      prev: 1,
      zoomIn: 1,
      oneToOne: 1,
      reset: 1,
      zoomOut: 1,
      next: 1,
    },
  })
})

watch(() => [props.file, props.compressFile], async () => {
  await nextTick()
  if (viewer)
    viewer.update()
})

onUnmounted(() => {
  if (viewer)
    viewer.destroy()
})
</script>

<template>
  <Card class="relative p4">
    <Collapsible v-model:open="isOpen" class="space-y-4">
      <div class="flex justify-between gap-2 lt-md:flex-col">
        <div class="flex flex-col gap-2">
          <div class="flex flex-wrap items-center gap-2">
            <span class="mr-2">{{ file.name }}</span>
            <Tag
              v-for="tag in photo.tags?.split(',') || []"
              :key="tag"
              :label="tag"
            />
          </div>
          <div class="flex flex-wrap items-center gap-4">
            <span class="text-lg font-medium">{{ photo.title || $t('upload_photo.untitled') }}</span>
            <span class="text-sm text-muted-foreground">{{ photo.caption }}</span>
          </div>
          <div class="flex flex-1 flex-wrap items-start gap-6">
            <div ref="viewerRef" class="flex flex-wrap gap-4">
              <UploadPhotoImage type="original" :file="file" />
              <template v-if="uploadConfig.enableCompression">
                <UploadPhotoImage v-if="uploadConfig.formats.jpeg" type="JPEG" :file="compressFile?.jpeg" />
                <UploadPhotoImage v-if="uploadConfig.formats.webp" type="WebP" :file="compressFile?.webp" />
                <UploadPhotoImage v-if="uploadConfig.formats.avif" type="AVIF" :file="compressFile?.avif" />
              </template>
              <UploadPhotoImage v-if="uploadConfig.formats.thumbnail" type="thumbnail" :file="compressFile?.thumbnail" />
            </div>

            <div class="min-w-[200px] flex flex-1 flex-col gap-2 text-sm text-muted-foreground font-mono">
              <span>{{ formatCameraText(photo) }}</span>
              <span>{{ formatExposure(photo).join(' • ') }}</span>
              <span>{{ photo.focalLength ? toFixed(photo.focalLength, 1) : '--' }}mm <span
                v-if="photo.focalLengthIn35mmFormat"
                :title="$t('camera_lens.focal_length_35mm')"
                class="op-50"
              >{{ photo.focalLengthIn35mmFormat }}mm</span> • {{ photo.exposureCompensation ? `${photo.exposureCompensation > 0 ? '+' : ''}${photo.exposureCompensation.toFixed(1)}ev` : '0ev' }}</span>
              <span>{{ formatDate(photo.takenAt) }}</span>
            </div>
          </div>
        </div>
        <div class="flex items-end justify-center gap-2 md:flex-col">
          <Button
            variant="outline"
            class="w-fit"
            :disabled="aiLoading || compressLoading || !loggedIn"
            :loading="uploadLoading"
            @click="loggedIn && emit('upload')"
          >
            <span>{{ $t('upload_photo.upload_button') }}</span>
          </Button>
          <Button
            v-if="aiConfig.enabled"
            variant="outline"
            class="w-fit"
            :loading="aiLoading"
            @click="emit('generate')"
          >
            <span>{{ $t('upload_photo.ai_button') }}</span>
          </Button>
        </div>
      </div>
      <CollapsibleContent>
        <Card class="p4">
          <UploadPhotoForm v-model="photo" />
        </Card>
      </CollapsibleContent>
    </Collapsible>
    <Button
      variant="outline"
      class="absolute bottom-0 left-50% h-6 w-12 translate-x--50% translate-y-50% p-0"
      @click="isOpen = !isOpen"
    >
      <div :class="isOpen ? 'i-lucide-chevron-up' : 'i-lucide-chevron-down'" />
    </Button>
    <button
      class="absolute right-4 top-4 rounded-sm bg-accent text-muted-foreground opacity-70 ring-offset-background transition-opacity disabled:pointer-events-none hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-ring"
      @click="emit('close')"
    >
      <div class="i-lucide-x size-4" />
      <span class="sr-only">Close</span>
    </button>
  </Card>
</template>

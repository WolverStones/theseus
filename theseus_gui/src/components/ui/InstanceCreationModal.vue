<template>
  <Modal ref="modal" header="Create instance" :noblur="!themeStore.advancedRendering">
    <div class="modal-header">
      <Chips v-model="creationType" :items="['custom', 'from file']" />
    </div>
    <div v-if="creationType === 'custom'" class="modal-body">
      <div class="image-upload">
        <Avatar :src="display_icon" size="md" :rounded="true" />
        <div class="image-input">
          <Button @click="upload_icon()">
            <UploadIcon />
            Select icon
          </Button>
          <Button @click="reset_icon">
            <XIcon />
            Remove icon
          </Button>
        </div>
      </div>
      <div class="input-row">
        <p class="input-label">Name</p>
        <input v-model="profile_name" autocomplete="off" class="text-input" type="text" />
      </div>
      <div class="input-row">
        <p class="input-label">Loader</p>
        <Chips v-model="loader" :items="loaders" />
      </div>
      <div class="input-row">
        <p class="input-label">Game version</p>
        <div class="versions">
          <multiselect
            v-model="game_version"
            class="selector"
            :options="game_versions"
            :multiple="false"
            :searchable="true"
            placeholder="Select game version"
            open-direction="top"
            :show-labels="false"
          />
          <Checkbox
            v-if="showAdvanced"
            v-model="showSnapshots"
            class="filter-checkbox"
            label="Include snapshots"
          />
        </div>
      </div>
      <div v-if="showAdvanced && loader !== 'vanilla'" class="input-row">
        <p class="input-label">Loader version</p>
        <Chips v-model="loader_version" :items="['stable', 'latest', 'other']" />
      </div>
      <div v-if="showAdvanced && loader_version === 'other' && loader !== 'vanilla'">
        <div v-if="game_version" class="input-row">
          <p class="input-label">Select version</p>
          <multiselect
            v-model="specified_loader_version"
            class="selector"
            :options="selectable_versions"
            :searchable="true"
            placeholder="Select loader version"
            open-direction="top"
            :show-labels="false"
          />
        </div>
        <div v-else class="input-row">
          <p class="warning">Select a game version before you select a loader version</p>
        </div>
      </div>
      <div class="input-group push-right">
        <Button @click="toggle_advanced">
          <CodeIcon />
          {{ showAdvanced ? 'Hide advanced' : 'Show advanced' }}
        </Button>
        <Button @click="$refs.modal.hide()">
          <XIcon />
          Cancel
        </Button>
        <Button color="primary" :disabled="!check_valid || creating" @click="create_instance()">
          <PlusIcon v-if="!creating" />
          {{ creating ? 'Creating...' : 'Create' }}
        </Button>
      </div>
    </div>
    <div v-else class="modal-body">
      <Button @click="openFile"> <FolderOpenIcon /> Import from file </Button>
      <div class="info"><InfoIcon /> Or drag and drop your .mrpack file</div>
    </div>
  </Modal>
</template>

<script setup>
import {
  Avatar,
  Button,
  Chips,
  Modal,
  PlusIcon,
  UploadIcon,
  XIcon,
  CodeIcon,
  Checkbox,
  FolderOpenIcon,
  InfoIcon,
} from 'omorphia'
import { computed, ref, shallowRef } from 'vue'
import { get_loaders } from '@/helpers/tags'
import { create } from '@/helpers/profile'
import { open } from '@tauri-apps/api/dialog'
import { tauri } from '@tauri-apps/api'
import {
  get_game_versions,
  get_fabric_versions,
  get_forge_versions,
  get_quilt_versions,
} from '@/helpers/metadata'
import { handleError } from '@/store/notifications.js'
import Multiselect from 'vue-multiselect'
import mixpanel from 'mixpanel-browser'
import { useTheming } from '@/store/state.js'
import { listen } from '@tauri-apps/api/event'
import { install_from_file } from '@/helpers/pack.js'

const themeStore = useTheming()

const profile_name = ref('')
const game_version = ref('')
const loader = ref('vanilla')
const loader_version = ref('stable')
const specified_loader_version = ref('')
const icon = ref(null)
const display_icon = ref(null)
const showAdvanced = ref(false)
const creating = ref(false)
const showSnapshots = ref(false)
const creationType = ref('from file')

defineExpose({
  show: () => {
    game_version.value = ''
    specified_loader_version.value = ''
    profile_name.value = ''
    creating.value = false
    showAdvanced.value = false
    showSnapshots.value = false
    loader.value = 'vanilla'
    loader_version.value = 'stable'
    icon.value = null
    display_icon.value = null
    modal.value.show()

    mixpanel.track('InstanceCreateStart', { source: 'CreationModal' })
  },
})

const [fabric_versions, forge_versions, quilt_versions, all_game_versions, loaders] =
  await Promise.all([
    get_fabric_versions().then(shallowRef).catch(handleError),
    get_forge_versions().then(shallowRef).catch(handleError),
    get_quilt_versions().then(shallowRef).catch(handleError),
    get_game_versions().then(shallowRef).catch(handleError),
    get_loaders()
      .then((value) =>
        value
          .filter((item) => item.supported_project_types.includes('modpack'))
          .map((item) => item.name.toLowerCase())
      )
      .then(ref)
      .catch(handleError),
  ])
loaders.value.unshift('vanilla')

const game_versions = computed(() => {
  return all_game_versions.value.versions
    .filter((item) => {
      let defaultVal = item.type === 'release' || showSnapshots.value
      if (loader.value === 'fabric') {
        defaultVal &= fabric_versions.value.gameVersions.some((x) => item.id === x.id)
      } else if (loader.value === 'forge') {
        defaultVal &= forge_versions.value.gameVersions.some((x) => item.id === x.id)
      } else if (loader.value === 'quilt') {
        defaultVal &= quilt_versions.value.gameVersions.some((x) => item.id === x.id)
      }

      return defaultVal
    })
    .map((item) => item.id)
})

const modal = ref(null)

const check_valid = computed(() => {
  return (
    profile_name.value.trim() &&
    game_version.value &&
    game_versions.value.includes(game_version.value)
  )
})

const create_instance = async () => {
  creating.value = true
  const loader_version_value =
    loader_version.value === 'other' ? specified_loader_version.value : loader_version.value
  const loaderVersion = loader.value === 'vanilla' ? null : loader_version_value ?? 'stable'

  modal.value.hide()
  creating.value = false

  await create(
    profile_name.value,
    game_version.value,
    loader.value,
    loader.value === 'vanilla' ? null : loader_version_value ?? 'stable',
    icon.value
  ).catch(handleError)

  mixpanel.track('InstanceCreate', {
    profile_name: profile_name.value,
    game_version: game_version.value,
    loader: loader.value,
    loader_version: loaderVersion,
    has_icon: !!icon.value,
    source: 'CreationModal',
  })
}

const upload_icon = async () => {
  icon.value = await open({
    multiple: false,
    filters: [
      {
        name: 'Image',
        extensions: ['png', 'jpeg', 'svg', 'webp', 'gif', 'jpg'],
      },
    ],
  })

  if (!icon.value) return
  display_icon.value = tauri.convertFileSrc(icon.value)
}

const reset_icon = () => {
  icon.value = null
  display_icon.value = null
}

const selectable_versions = computed(() => {
  if (game_version.value) {
    if (loader.value === 'fabric') {
      return fabric_versions.value.gameVersions[0].loaders.map((item) => item.id)
    } else if (loader.value === 'forge') {
      return forge_versions.value.gameVersions
        .find((item) => item.id === game_version.value)
        .loaders.map((item) => item.id)
    } else if (loader.value === 'quilt') {
      return quilt_versions.value.gameVersions[0].loaders.map((item) => item.id)
    }
  }
  return []
})

const toggle_advanced = () => {
  showAdvanced.value = !showAdvanced.value
}

const openFile = async () => {
  const newProject = await open({ multiple: false })
  if (!newProject) return

  modal.value.hide()
  await install_from_file(newProject).catch(handleError)

  mixpanel.track('InstanceCreate', {
    source: 'CreationModalFileOpen',
  })
}

listen('tauri://file-drop', async (event) => {
  modal.value.hide()
  if (event.payload && event.payload.length > 0 && event.payload[0].endsWith('.mrpack')) {
    await install_from_file(event.payload[0]).catch(handleError)
    mixpanel.track('InstanceCreate', {
      source: 'CreationModalFileDrop',
    })
  }
})
</script>

<style lang="scss" scoped>
.modal-body {
  display: flex;
  flex-direction: column;
  padding: var(--gap-lg);
  gap: var(--gap-md);
}

.input-label {
  font-size: 1rem;
  font-weight: bolder;
  color: var(--color-contrast);
  margin-bottom: 0.5rem;
}

.text-input {
  width: 20rem;
}

.image-upload {
  display: flex;
  gap: 1rem;
}

.image-input {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  justify-content: center;
}

.warning {
  font-style: italic;
}

.versions {
  display: flex;
  flex-direction: row;
  gap: 1rem;
}

:deep(button.checkbox) {
  border: none;
}

.selector {
  max-width: 20rem;
}

.labeled-divider {
  text-align: center;
}

.labeled-divider:after {
  background-color: var(--color-raised-bg);
  content: 'Or';
  color: var(--color-base);
  padding: var(--gap-sm);
  position: relative;
  top: -0.5rem;
}

.info {
  display: flex;
  flex-direction: row;
  gap: 0.5rem;
  align-items: center;
}

.modal-header {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  padding: var(--gap-lg);
  padding-bottom: 0;
}
</style>

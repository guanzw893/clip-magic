<template>
  <div class="flex justify-center pt-[10px] text-[12px]">
    <a-tag
      class="cursor-pointer rounded-full"
      v-for="tag in clipboard.tags"
      :color="tag.color"
      :key="tag.id"
      @dragenter="handleDragEnter(tag)"
      @dragleave="handleDragLeave"
      @dragover="$event.preventDefault()"
      @click="selectTag(tag)"
      @contextmenu="onTagMenuClick($event, tag)"
    >
      {{ tag.title }}
    </a-tag>

    <a-tag
      v-if="!isShowAddTagInput"
      class="w-[60px] cursor-pointer border-dashed bg-white"
      @click="handleShowAddTagInput"
    >
      New Tag
    </a-tag>
    <a-input
      v-else
      ref="addTagInputRef"
      v-model:value="clipboard.currTag.title"
      type="text"
      size="small"
      class="h-full w-[60px]"
      @blur="handleAddTagInputBlur"
      @keyup.enter="handleAddTagInputBlur"
    />
  </div>

  <div
    ref="pasteListRef"
    class="flex overflow-y-hidden overflow-x-scroll p-[10px]"
  >
    <a-tooltip
      class="clip-item ml-[20px] h-[200px] min-w-[200px] max-w-[200px] select-none overflow-hidden rounded-md border border-primary-500 p-[5px] transition-shadow first:ml-0 hover:shadow-md hover:shadow-primary-500"
      v-if="clipboard.clipList.length"
      v-for="item in clipboard.clipList"
      :key="item.id"
      placement="leftTop"
      trigger="click"
      title="双击复制"
    >
      <div
        draggable="true"
        @dragstart="handleDragStart(item)"
        @dragend="handleDragEnd(item)"
        @dblclick="handleSelect(item)"
        @contextmenu="onItemMenuClick($event, item)"
      >
        <a-tag
          class="h-[50px] w-full py-[5px]"
          :color="clipboard.currTag.color"
        >
          <input
            class="border-none bg-transparent outline-none"
            v-model.lazy="item.title"
          />
          <div>
            {{ moment(item.date).format('YYYY/MM/DD hh:mm:ss') }}
          </div>
        </a-tag>
        {{ item.content }}
      </div>
    </a-tooltip>

    <div
      v-else
      class="flex h-[200px] w-full items-center justify-center text-[12px] text-slate-400"
    >
      暂无内容
    </div>
  </div>

  <ContextMenu ref="itemMenuRef" :model="itemMenu" />
  <ContextMenu ref="tagMenuRef" :model="tagMenu" />
</template>

<script setup lang="ts">
import moment from 'moment'
import ContextMenu from 'primevue/contextmenu'
import { MenuItem } from 'primevue/menuitem'

import { useClipboard, type ClipItem, type Tag, OptionType } from '~/store'

definePageMeta({
  middleware: 'route-guard'
})

const clipboard = useClipboard()
const pasteListRef = ref<HTMLDivElement>()
const addTagInputRef = ref<HTMLInputElement>()
const isShowAddTagInput = ref(false)
const currentDrag = ref<{ tag?: Tag; item?: ClipItem }>({})

onMounted(() => {
  // 监听修改创建
  window.addEventListener('storage', content => {
    const data = JSON.parse(content.newValue!)
    clipboard.list = data.list
    clipboard.currentItemMenu = data.currentItemMenu
  })

  let timer: any = null
  const interval = 300
  document.addEventListener('visibilitychange', async () => {
    if (!document.hidden) {
      const list = await window.electron.getClipTextSaveList()
      list.forEach(text => {
        clipboard.add({
          content: text
        })
      })
      timer = setInterval(async () => {
        const text = await window.electron.getClipText()
        if (!text) return
        clipboard.add({
          content: text
        })
      }, interval)
    } else {
      timer && clearInterval(timer)
    }
  })
})

const handleSelect = async (item: ClipItem) => {
  await window.electron.paste(item.content)
  pasteListRef.value?.scrollTo({
    left: 0
  })
}

const handleShowAddTagInput = async () => {
  isShowAddTagInput.value = true
  await nextTick()
  addTagInputRef.value?.focus()
}

const handleAddTagInputBlur = () => {
  isShowAddTagInput.value = false
  if (!clipboard.currTag.title) return
  clipboard.addTag({ title: clipboard.currTag.title })
  clipboard.currTag.title = ''
}

const selectTag = (tag: Tag) => {
  clipboard.selectTag(tag)
}

const handleDragStart = (item: ClipItem) => {
  currentDrag.value.item = item
}

const handleDragEnter = (tag: Tag) => {
  currentDrag.value.tag = tag
}

const handleDragLeave = () => {
  currentDrag.value.tag = undefined
}

const handleDragEnd = (item: ClipItem) => {
  if (!currentDrag.value?.tag) return
  const tag = currentDrag.value.tag
  item.tagId = tag.id
  clipboard.selectTag(tag)
}

const itemMenuRef = ref<InstanceType<typeof ContextMenu> | null>(null)
const tagMenuRef = ref<InstanceType<typeof ContextMenu> | null>(null)

const itemMenu: MenuItem[] = [
  {
    label: '修改',
    command() {
      clipboard.optionType = OptionType.Edit
      clipboard.currentTagMenu = undefined
      window.electron.showEdit()
    }
  },
  {
    label: '删除',
    command() {
      if (!clipboard.currentItemMenu) return
      clipboard.remove(clipboard.currentItemMenu.id)
    }
  }
]

const firstTagMenu: MenuItem[] = [
  {
    label: '新增记录',
    order: 1,
    command() {
      clipboard.optionType = OptionType.Add
      clipboard.currentItemMenu = undefined
      window.electron.showEdit()
    }
  },
  {
    label: '删除全部记录',
    order: 3,
    command() {
      if (!clipboard.currentTagMenu) return
      clipboard.removeByTagId(clipboard.currentTagMenu.id)
    }
  }
]

const otherTagMenu: MenuItem[] = [
  {
    label: '删除标签',
    order: 2,
    command() {
      const id = clipboard.currentTagMenu?.id
      if (!clipboard.currentTagMenu || !id) return
      clipboard.removeTag(id)
      clipboard.removeByTagId(id)
      clipboard.selectTag(clipboard.firstTag)
    }
  },
  ...firstTagMenu
]

const tagMenu = computed(() =>
  (clipboard.currentTagMenu?.id === 0 ? firstTagMenu : otherTagMenu).sort(
    (a, b) => a.order - b.order
  )
)

const onItemMenuClick = (e: Event, item: ClipItem) => {
  clipboard.currentItemMenu = item
  itemMenuRef.value?.show(e)
}

const onTagMenuClick = (e: Event, tag: Tag) => {
  clipboard.currentTagMenu = tag
  tagMenuRef.value?.show(e)
}

onUnmounted(() => {
  clearInterval(timer)
})
</script>

<style scoped>
.clip-item {
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  word-break: break-all;
  -webkit-line-clamp: 7;
  -webkit-box-orient: vertical;
}
</style>

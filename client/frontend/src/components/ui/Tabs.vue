<script>
import { ref, onMounted, onUnmounted, provide, nextTick, watch } from 'vue'
import Icon from './Icon.vue'

export default {
  components: {
    Icon
  },
  props: {
    anchors: { type: Boolean, default: () => false },
    active: { type: String, default: () => '' }
  },
  emits: ['tabChanged', 'update:active'],
  setup(props, { slots, emit }) {
    const tabButtons = ref(null)
    const needsScroller = ref(false)

    const tabs = ref([])
    const activeKey = ref('')
    provide('activeKey', activeKey)

    // keep activeKey in sync with parent-provided `active` prop (v-model:active)
    watch(() => props.active, (v) => {
      if (v && v !== activeKey.value) activeKey.value = v
    }, { immediate: true })

    function setActive(key) {
      activeKey.value = key

      // Notify parent so it can decide how to represent the tab in the URL (router-driven)
      emit('update:active', key)

      // deferring emit to next tick to ensure the tab content has changed
      nextTick(() => emit('tabChanged', key))
    }

    function onResize() {
      needsScroller.value = tabButtons.value.scrollWidth > tabButtons.value.offsetWidth
    }

    function scroll(dir) {
      const dist = (tabButtons.value.offsetWidth / 2)
      tabButtons.value.scrollTo({
        behavior: 'smooth',
        left: tabButtons.value.scrollLeft + (dir === 'right' ? dist : dist * -1)
      })
    }

    onMounted(() => {
      window.addEventListener('resize', onResize)
      nextTick(() => onResize())

      tabs.value = slots
        .default()
        .filter(e => e && e.props && e.props.title)
        .map(e => {
          return {
            key: e.props.id || e.props.title.toLowerCase().replace(/ /g, '-'),
            title: e.props.title,
            icon: e.props.icon,
            hotkey: e.props.hotkey
          }
        })
      // if parent provided an active prop it will be applied via the watcher above
      // otherwise fall back to legacy hash behavior if anchors is enabled
      if (!props.active && props.anchors && tabs.value.length > 0 && location.hash) {
        const tab = tabs.value.find(e => e.key === location.hash.substring(1))
        if (tab) setActive(tab.key)
      }

      if (tabs.value.length > 0 && !activeKey.value) {
        setActive(tabs.value[0].key)
      }
    })

    onUnmounted(() => {
      window.removeEventListener('resize', onResize)
    })

    return { tabButtons, needsScroller, tabs, activeKey, setActive, scroll }
  }
}
</script>

<template>
  <div class="tabs">
    <div v-if="needsScroller" class="scroll-left" @click="scroll('left')" />
    <div v-if="needsScroller" class="scroll-right" @click="scroll('right')" />
    <div ref="tabButtons" class="tab-buttons">
      <div
        v-for="tab in tabs"
        :key="tab.key"
        v-hotkey="tab.hotkey"
        :class="['tab-button', tab.key === activeKey ? 'active' : 'inactive', tab.icon ? 'has-icon' : '']"
        @click="setActive(tab.key)"
      >
        <icon v-if="tab.icon" :name="tab.icon" />
        <span class="title" v-text="tab.title" />
      </div>
    </div>
    <slot />
  </div>
</template>

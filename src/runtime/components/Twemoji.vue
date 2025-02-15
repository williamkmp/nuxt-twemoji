<!-- eslint-disable vue/multi-word-component-names -->
<script setup lang="ts">
import { ref, computed, defineComponent, h, watchEffect } from 'vue'
import { useState } from '#imports'
import type { EmojiDefinition } from './../assets/emojis'

const props = defineProps({
  emoji: {
    type: [String, Object as () => EmojiDefinition],
    required: true
  },
  size: {
    type: String,
    default: '1em'
  },
  png: {
    type: Boolean,
    default: false
  }
})

const toCodePoint = (unicodeSurrogates: string) => {
  const points: string[] = []
  let char = 0
  let previous = 0
  let i = 0
  while (i < unicodeSurrogates.length) {
    char = unicodeSurrogates.charCodeAt(i++)
    if (previous) {
      points.push((0x10000 + (previous - 0xd800 << 10) + (char - 0xdc00)).toString(16))
      previous = 0
    }
    else if (char > 0xd800 && char <= 0xdbff) {
      previous = char
    }
    else {
      points.push(char.toString(16))
    }
  }
  return points.join('-')
}

const isString = typeof props.emoji === 'string' 
const isDefinition = typeof props.emoji === 'object' && props.emoji.code !== undefined && props.emoji.emoji !== undefined && props.emoji.name !== undefined

const isValid = computed(() => isString || isDefinition)

if (!isValid.value) {
  console.warn(`Invalid emoji property:`, props.emoji)
}

const twemoji = computed(() => {
  if (isDefinition) return props.emoji.code.toLowerCase()
  else if (isString) return props.emoji.replace(/u\+/ig, '').toLowerCase()
  return ''
})

const alt = computed(() => {
  if (isDefinition) return props.emoji.name
  else if (isString) return props.emoji
  return JSON.stringify(props.emoji)
})

const isHex = computed(() => (/^[0-9A-Fa-f]{1,6}(-[0-9A-Fa-f]{1,6})*?$/i).test(twemoji.value))

const codePoint = ref<{ [key: string]: string }>({})
const isFetching = ref(false)

const parsed = computed(() => {
  if (isHex.value) return twemoji.value
  return toCodePoint(twemoji.value)
})

codePoint.value[parsed.value] = parsed.value

const cdn = 'https://cdn.jsdelivr.net/gh/jdecked/twemoji@latest/assets'
const emojiLinkPNG = computed(() => `${cdn}/72x72/${codePoint.value[parsed.value]}.png`)
const emojiLinkSVG = computed(() => `${cdn}/svg/${codePoint.value[parsed.value]}.svg`)

const fetchSVG = () => $fetch(emojiLinkSVG.value).then(async (res: any) => await res.text()).catch(() => undefined)
const svgTwemojis = useState('twemojis', () => ({}) as { [key: string]: { body: string } })

const component = computed (() => {
  if (!svgTwemojis.value[parsed.value]) return
  return defineComponent({
    render() {
      return h('svg', {
        class: 'twemoji',
        xmlns: "http://www.w3.org/2000/svg",
        viewBox: '0 0 36 36',
        width: props.size,
        height: props.size,
        innerHTML: svgTwemojis.value[parsed.value]?.body
      })
    }
  })
})

const loadSVG = async () => {
  if (svgTwemojis.value[parsed.value] || !isValid.value) return
  isFetching.value = true
  let svgFetch = await fetchSVG()
  isFetching.value = false
  const split = parsed.value.split('-')
  while (!svgFetch && split.length > 1) {
    // While svgFetch fetch fails, retry with previous codepoint segments
    split.pop()
    codePoint.value[parsed.value] = split.join('-')
    svgFetch = await fetchSVG()
  }
  if (!svgFetch) return
  svgTwemojis.value[parsed.value] = {
    body: svgFetch.replace(/<\/*svg[^>]*>/g, '')
  }
}
watchEffect(async () => {
  codePoint.value[parsed.value] = parsed.value
  !props.png && await loadSVG()
});

!props.png && await loadSVG()
</script>

<template>
  <span v-if="isFetching" />
  <img v-else-if="png" class="twemoji" :src="emojiLinkPNG" :alt="alt" :style="{ width: size, height: size }">
  <Component :is="component" v-else-if="component" />
  <span v-else>{{ twemoji }}</span>
</template>

<style>
.twemoji {
  display: inline-block;
  vertical-align: middle;
}
</style>

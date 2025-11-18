<script setup lang="ts">
import {Item} from './Models/Item';

const props = defineProps<{
  item: Item,
  navbarExpanded: boolean,
  iconSize: string,
}>();
</script>

<template>
  <div class="w-full h-8 flex items-center relative" :class="[props.navbarExpanded ? 'justify-start px-1.5' : 'justify-center']">
    <span :class="[item.icon, 'text-' + iconSize]"/>
    <div v-if="props.navbarExpanded" class="ml-2 flex items-center justify-between w-full max-w-[165px]">
      <div class="truncate">{{ item.title }}</div>
      <div class="flex items-center gap-1">
        <span v-if="item.badge" class="navbar-badge">{{ item.badge }}</span>
        <span v-if="props.navbarExpanded && item.children" class="mdi mdi-chevron-down"/>
      </div>
    </div>
    <div v-else>
      <div class="absolute right-0 top-0 -mt-1 -mr-1">
        <span v-if="item.badge" class="navbar-badge">{{ item.badge }}</span>
      </div>
    </div>
  </div>
</template>

<style scoped>
.navbar-badge {
  @apply text-xs text-center py-0.5 px-1 min-w-6 ml-2 bg-rose-500 text-white rounded-full;
}
</style>
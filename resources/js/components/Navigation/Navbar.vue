<script setup lang="ts">

import {computed, nextTick, onMounted, onUnmounted, ref} from "vue";
import router from "../../router.js";
import {useRoute} from "vue-router";
import NavbarItem from "./NavbarItem.vue";
import NavbarSubmenu from "./NavbarSubmenu.vue";
import {Item} from "./Models/Item";
import {NavbarConfig} from "./Models/NavbarConfig";


const props = defineProps<{
  menuItems: { floating: Item[], fixed?: Item[], iconMenu?: Item[], iconUser?: Item[] },
  navbarConfig: NavbarConfig,
}>();

const route = useRoute();

const mobileBreakpoint = props.navbarConfig?.mobileBreakpoint ?? 640;

// Navbar state
const isMobileInit = window.innerWidth < mobileBreakpoint;
const navbarExpanded = ref<boolean>(
    isMobileInit
        ? false                        // always collapsed on mobile at load
        : localStorage.getItem('navbar_is_expanded') === 'true'
);
const submenuExpanded = ref<Record<string, boolean>>({});
const tooltip = ref<{ visible: boolean, text: string, styles: any, arrowPosition: 'left' | 'right' | 'center' }>({visible: false, text: '', styles: {}, arrowPosition: 'center'});
const popup = ref<{ visible: boolean, items: Item[], styles: any }>({visible: false, items: [], styles: {}});
const mainPopup = ref<{ visible: boolean; styles: any }>({visible: false, styles: {}});

// Parse navbar config
const site = props.navbarConfig?.site ?? null;
const user = props.navbarConfig?.user ?? null;
const itemActiveBGColor = props.navbarConfig?.colors?.itemActiveBG ?? '#756094';
const itemActiveTextColor = props.navbarConfig?.colors?.itemActiveText ?? '#FFF';
const itemTextColor = props.navbarConfig?.colors?.itemText ?? '#9CA4B0';
const itemHoverBGColor = props.navbarConfig?.colors?.itemHoverBG ?? 'rgba(255,255,255,0.3)';
const itemHoverTextColor = props.navbarConfig?.colors?.itemHoverText ?? '#c1c6cd';
const primaryColor = props.navbarConfig?.colors?.navbarBackgroundFrom ?? '#452750';
const secondaryColor = props.navbarConfig?.colors?.navbarBackgroundTo ?? '#2F1937';
const iconSize = props.navbarConfig?.iconSize ?? 'xl'; // Tailwind JIT safelist: 'text-xs', 'text-sm', 'text-base', 'text-lg', 'text-xl', 'text-2xl', 'text-3xl', 'text-4xl', 'text-5xl', 'text-6xl', 'text-7xl', 'text-8xl', 'text-9xl'

// Helper function to check if an item (or its children) is active
const isActive = (item: Item): boolean => {
  if (item.to && item.to === route.path) {
    return true;
  } else if (item.children) {
    return item.children.some(child => child.to === route.path);
  }
  return false;
}

// Compute active state for each item
const addComputedValues = (items: Item[]): Item[] => {
  const result = items.map((item) => ({
    ...item,
    isActive: isActive(item),
    children: item.children ? addComputedValues(item.children) : undefined,
  }));
  // look for active children and add to submenuExpanded
  if (navbarExpanded.value) {
    result.forEach((item, index) => {
      if (item.children && item.children.some(child => child.isActive)) {
        submenuExpanded.value[index] = true;
      }
    });
  }
  // look for 'auto' badge and when has children, set the badge to the sum of children badges values
  result.forEach((item) => {
    if (item.badge === 'auto' && item.children) {
      item.badge = item.children.reduce((acc, child) => acc + (child.badge ?? 0), 0);
    }
  });
  return result;
}

const floatingMenuItems = computed(() => addComputedValues(props.menuItems.floating));
const fixedMenuItems = computed(() => addComputedValues(props.menuItems.fixed ?? []));
const iconMenuItems = computed(() => addComputedValues(props.menuItems.iconMenu ?? []));
const iconUserItems = computed(() => addComputedValues(props.menuItems.iconUser ?? []));

// Toggle navbar size
function toggleNavbarExpansion(newState: boolean | null = null, byResize: boolean = false) {
  navbarExpanded.value = newState ?? !navbarExpanded.value;
  if (navbarExpanded.value === false) {
    submenuExpanded.value = {};
  } else {
    hideTooltip();
    // expand submenu for active items
    floatingMenuItems.value.forEach((item, index) => {
      if (item.children && item.isActive) {
        submenuExpanded.value['floating_' + index] = true;
      }
    });
    fixedMenuItems.value.forEach((item, index) => {
      if (item.children && item.isActive) {
        submenuExpanded.value['fixed_' + index] = true;
      }
    });
  }
  if (!byResize) {
    localStorage.setItem('navbar_is_expanded', navbarExpanded.value.toString());
  }
}

// Toggle submenu visibility for expanded navbar
const toggleSubmenuExpand = (groupAndIndex: string) => {
  submenuExpanded.value[groupAndIndex] = !submenuExpanded.value[groupAndIndex];
}

// Handle item click
const handleClick = async (item: Item, groupAndIndex: string, event: MouseEvent) => {
  if (item.children && !item.to && navbarExpanded.value) {
    // Expand submenu when navbar is expanded
    toggleSubmenuExpand(groupAndIndex)
  } else if (item.children && !navbarExpanded.value) {
    // Show popup submenu when navbar is collapsed
    showPopupMenu(item.title, item.children, event);
    event.stopPropagation();
  } else if (item.to) {
    // Navigate if `to` param exists
    router.push(item.to)
  }

  await nextTick();
  checkScrollbar();
}

// Handle submenu item click
const handleSubmenuClick = (child: Item) => {
  if (child.to) router.push(child.to);
  popup.value.visible = false; // Close popup after click
};

const showTooltip = (groupAndIndex: string, event: MouseEvent) => {
  const [group, index] = groupAndIndex.split('_');
  if (group === 'iconMenu' || group === 'iconUser') {
    return
  }
  const item = group && index ? props.menuItems[group][index] ?? null : null;
  const rect = event.target.getBoundingClientRect();

  const isMobile = window.innerWidth < mobileBreakpoint;
  const tooltipWidth = item?.title.length * 6 ?? 40;
  const viewportWidth = window.innerWidth;
  let arrowPosition = 'center';

  let left = isMobile
      ? rect.left + rect.width / 2
      : rect.right + 10;

  if (left + tooltipWidth > viewportWidth) {
    left = viewportWidth - tooltipWidth;
    arrowPosition = 'right';
  }

  tooltip.value = {
    visible: true,
    text: group === 'expandToggle' ? 'Expand' : item.title,
    styles: {
      top: isMobile ? `${rect.bottom}px` : `${rect.top + rect.height / 2}px`,
      transform: isMobile ? 'translateX(-50%)' : 'translateY(-50%)',
      left: `${left}px`,
    },
    arrowPosition,
  };
};

// Hide Tooltip
const hideTooltip = () => {
  tooltip.value.visible = false;
};

// Show popup submenu
const showPopupMenu = (title: string, items: Item[], event: MouseEvent) => {
  const isMobile = window.innerWidth < mobileBreakpoint;
  let rect;
  if (isMobile) {
    rect = document.getElementById('navbar').getBoundingClientRect();
  } else {
    rect = event.target.getBoundingClientRect();
  }

  popup.value = {
    visible: true,
    title: title,
    items,
    styles: {
      top: isMobile ? `${rect.bottom}px` : `${rect.top}px`,
      left: isMobile ? 0 : `${rect.right + 10}px`,
      width: isMobile ? '100%' : 'auto',
    }
  };
};

// Show main popup
const showMainPopup = (event: MouseEvent) => {
  const rect = document.getElementById('logo-section').getBoundingClientRect();
  mainPopup.value = {
    visible: true,
    styles: {
      top: `${rect.bottom + 10}px`,
      left: `${rect.left + 10}px`,
    }
  };
};

// Close popup when clicked outside
const handleClickOutside = (event: MouseEvent) => {
  // Check if clicked inside the popup
  if (!event.target.closest('.navbar-popup') && popup.value.visible) {
    popup.value.visible = false;
  }

  if (!event.target.closest('.main-popup') && mainPopup.value.visible) {
    mainPopup.value.visible = false;
  }
};

// Handle window resize
let timeout: NodeJS.Timeout;
const handleResize = () => {
  clearTimeout(timeout);
  timeout = setTimeout(() => {
    const isMobile = window.innerWidth < mobileBreakpoint;
    if (isMobile) {
      toggleNavbarExpansion(false, true);
      submenuExpanded.value = {};
    } else {
      // Restore expansion based on saved preference when leaving mobile mode
      const saved = localStorage.getItem('navbar_is_expanded') === 'true';
      toggleNavbarExpansion(saved, true);
    }
  }, 200); // Delay in milliseconds
};

function checkScrollbar() {
  console.log("Toggling scrollbar shadows");
  const nav = document.getElementById("floating-nav");
  const shadow = document.getElementsByClassName("nav-shadow");
  console.log(nav.scrollHeight, nav.clientHeight);
  if (nav.scrollHeight > nav.clientHeight || (window.innerWidth < mobileBreakpoint && nav.scrollWidth > nav.clientWidth)) {
    Array.from(shadow).forEach(element => element.classList.add("opacity-100"));
  } else {
    Array.from(shadow).forEach(element => element.classList.remove("opacity-100"));
  }
}

// Attach and detach event listeners
onMounted(() => {
  window.addEventListener('resize', handleResize)
  document.addEventListener('click', handleClickOutside);
  checkScrollbar();
  window.addEventListener("resize", checkScrollbar);
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
  document.removeEventListener('click', handleClickOutside);
});


// Run on load and when window resizes

</script>

<template>
  <div id="navbar" class="sm:h-screen flex flex-row sm:flex-col gap-1 sm:gap-4 items-center py-2 sm:py-4"
       :class="[ navbarExpanded ? 'sm:w-64 sm:max-w-64' : 'sm:w-14' ]"
       :style="[ `background: ${primaryColor}`, `background: linear-gradient(135deg, ${primaryColor} 0%, ${secondaryColor} 100%)`]">

    <!-- Logo section -->
    <div @click.stop="showMainPopup" id="logo-section" class="min-w-14 sm:w-full flex items-center" :class="[navbarExpanded ? 'justify-start pl-4' : 'justify-center']">
      <img v-if="site?.logo" :src="site.logo" alt="logo" class="w-full max-w-8 aspect-square"/>
      <svg v-else viewBox="0 0 500 500" xmlns="http://www.w3.org/2000/svg" class="w-full max-w-8 aspect-square">
        <rect y="20" width="250" height="250" style="stroke: rgb(0, 0, 0); fill: rgb(117, 96, 148);" x="20"></rect>
        <rect x="20" y="300" width="180" height="180" style="stroke: rgb(0, 0, 0); fill: rgb(143, 117, 180);"></rect>
        <rect x="330" y="330" width="150" height="150" style="stroke: rgb(0, 0, 0); fill: rgb(179, 147, 227);"></rect>
        <rect x="300" y="20" width="180" height="180" style="stroke: rgb(0, 0, 0); fill: rgb(143, 117, 180);"></rect>
      </svg>
      <div v-if="navbarExpanded" class="flex flex-col h-full ml-2 overflow-hidden leading-3">
        <span class="font-bold text-lg truncate text-white">{{ site.name }}</span>
        <span>{{ user?.name }}</span>
      </div>
    </div>

    <!-- Navigation Items -->
    <nav id="floating-nav" class="flex flex-row sm:flex-col gap-2 h-full w-full overflow-y-hidden sm:overflow-y-auto overflow-x-auto sm:overflow-x-hidden scrollbar-hidden px-2">
      <div v-for="(item, index) in floatingMenuItems" :key="'floating_' + index" class="relative w-full">
        <div @click="handleClick(item, 'floating_' + index, $event)"
             @mouseenter="navbarExpanded ? () => {} : showTooltip('floating_' + index, $event)"
             @mouseleave="hideTooltip"
             class="navbar-button"
             :style="item.isActive ? `background-color: ${itemActiveBGColor}; color: ${itemActiveTextColor};` : `--hover-bg: ${itemHoverBGColor}; color: ${itemTextColor}; --hover-text: ${itemHoverTextColor};`">

          <!-- Icon and Title -->
          <NavbarItem :navbar-expanded="navbarExpanded" :item="item" :icon-size="iconSize"/>

        </div>

        <!-- Submenu -->
        <NavbarSubmenu v-if="item.children && submenuExpanded['floating_' + index]"
                       :children="item.children" :active-color="itemActiveBGColor" :icon-size="iconSize"
                       @submenu-clicked="handleSubmenuClick"/>

      </div>
    </nav>

    <!-- Bottom Navigation Items -->
    <div class="relative sm:w-full">
      <div class="nav-shadow absolute -top-2 sm:-top-8 -left-4 sm:left-0 w-4 sm:w-full h-[calc(100%+1rem)] sm:h-4 bg-gradient-to-l sm:bg-gradient-to-t from-black/30 to-transparent opacity-0 pointer-events-none transition-opacity duration-300"></div>
      <nav id="fixed-nav" class="flex flex-row sm:flex-col gap-2 sm:w-full justify-center px-2">
        <div v-for="(item, index) in fixedMenuItems" :key="'fixed_' + index" class="relative w-full">
          <div @click="handleClick(item, 'fixed_' + index, $event)"
               @mouseenter="navbarExpanded ? () => {} : showTooltip('fixed_' + index, $event)"
               @mouseleave="hideTooltip"
               class="navbar-button"
               :style="item.isActive ? `background-color: ${itemActiveBGColor}; color: ${itemActiveTextColor};` : `--hover-bg: ${itemHoverBGColor}; color: ${itemTextColor}; --hover-text: ${itemHoverTextColor};`">

            <!-- Icon and Title -->
            <NavbarItem :navbar-expanded="navbarExpanded" :item="item" :icon-size="iconSize"/>

          </div>

          <!-- Submenu -->
          <NavbarSubmenu v-if="item.children && submenuExpanded['fixed_' + index]"
                         :children="item.children" :active-color="itemActiveBGColor" :icon-size="iconSize"
                         @submenu-clicked="handleSubmenuClick"/>

        </div>
      </nav>
    </div>

    <!-- Expand/Collapse Button -->
    <nav id="collapse-nav" class="hidden sm:flex flex-row sm:flex-col space-y-2 w-full xs:pb-2 px-2">
      <div class="relative w-full">
        <div @click="toggleNavbarExpansion(null, false)"
             @mouseenter="navbarExpanded ? () => {} : showTooltip('expandToggle', $event)"
             @mouseleave="navbarExpanded ? () => {} : hideTooltip"
             class="navbar-button"
             :style="`--hover-bg: ${itemHoverBGColor}; color: ${itemTextColor}; --hover-text: ${itemHoverTextColor};`">
          <div class="w-full h-8 flex items-center" :class="[navbarExpanded ? 'justify-start px-1.5' : 'justify-center']">
            <span class="mdi" :class="[navbarExpanded ? 'mdi-chevron-left' : 'mdi-chevron-right', 'text-' + iconSize]"/>
            <div v-if="navbarExpanded" class="ml-2">
              Collapse
            </div>
          </div>
        </div>
      </div>
    </nav>

  </div>

  <!-- Tooltip -->
  <Teleport to="body">
    <div v-if="tooltip.visible && !navbarExpanded" class="navbar-tooltip" :style="tooltip.styles">
      {{ tooltip.text }}
      <div class="navbar-tooltip-arrow"
           :class="{
        'left-1/2 sm:left-0 -translate-x-1/2 sm:translate-x-0': tooltip.arrowPosition === 'center',
        'left-1/4 sm:left-0 -translate-x-1/2 sm:translate-x-0': tooltip.arrowPosition === 'left',
        'left-3/4 sm:left-0 -translate-x-1/2 sm:translate-x-0': tooltip.arrowPosition === 'right'
      }"/>
    </div>
  </Teleport>

  <!-- Popup Submenu -->
  <Teleport to="body">
    <div v-if="popup.visible"
         :style="[popup.styles, `background-color: ${primaryColor}`]"
         class="navbar-popup">
      <div class="p-2 font-bold">{{ popup.title }}</div>
      <div v-for="(child, index) in popup.items" :key="'popup_' + index"
           @click="handleSubmenuClick(child)"
           class="popup-item"
           :class="child.isActive ? 'text-white' : 'hover:bg-white/30 text-gray-400'"
           :style="child.isActive ? `background-color: ${itemActiveBGColor}` : ''">
        <div class="truncate">
          <div class="flex items-center gap-2">
            <span v-if="child.icon" :class="[child.icon, 'text-' + iconSize]"/>
            <span v-else class="mdi mdi-circle-medium" :class="['text-' + iconSize]"/>
            <span class="truncate">{{ child.title }}</span>
          </div>
        </div>
        <span v-if="child.badge" class="navbar-badge">{{ child.badge }}</span>
      </div>
    </div>
  </Teleport>

  <!-- Icon Popup -->
  <Teleport to="body">
    <div v-if="mainPopup.visible"
         :style="[mainPopup.styles, `background: ${primaryColor}`, `background: linear-gradient(135deg, ${primaryColor} 0%, ${secondaryColor} 100%); border: 1px solid ${secondaryColor}; color: ${itemTextColor}`]"
         class="main-popup">
      <div class="flex">
        <img v-if="site?.logo" :src="site.logo" alt="logo" class="max-h-10 aspect-square"/>
        <svg v-else viewBox="0 0 500 500" xmlns="http://www.w3.org/2000/svg" class="w-full max-w-8 aspect-square">
          <rect y="20" width="250" height="250" style="stroke: rgb(0, 0, 0); fill: rgb(117, 96, 148);" x="20"></rect>
          <rect x="20" y="300" width="180" height="180" style="stroke: rgb(0, 0, 0); fill: rgb(143, 117, 180);"></rect>
          <rect x="330" y="330" width="150" height="150" style="stroke: rgb(0, 0, 0); fill: rgb(179, 147, 227);"></rect>
          <rect x="300" y="20" width="180" height="180" style="stroke: rgb(0, 0, 0); fill: rgb(143, 117, 180);"></rect>
        </svg>
        <div class="flex flex-col h-full mx-2 overflow-hidden leading-3">
          <span class="font-bold text-lg truncate" :style="`color: ${itemActiveTextColor}`">{{ site.name }}</span>
          <span>Some description</span>
        </div>
      </div>

      <nav class="flex flex-row sm:flex-col gap-2 h-full w-full">
        <div v-for="(item, index) in iconMenuItems" :key="'iconMenu_' + index" class="relative w-full">
          <div @click="handleClick(item, 'iconMenu_' + index, $event)"
               @mouseenter="navbarExpanded ? () => {} : showTooltip('iconMenu_' + index, $event)"
               @mouseleave="hideTooltip"
               class="navbar-button"
               :style="item.isActive ? `background-color: ${itemActiveBGColor}; color: ${itemActiveTextColor};` : `--hover-bg: ${itemHoverBGColor}; color: ${itemTextColor}; --hover-text: ${itemHoverTextColor};`">

            <!-- Icon and Title -->
            <NavbarItem :navbar-expanded="true" :item="item" :icon-size="iconSize"/>

          </div>

          <!-- Submenu -->
          <NavbarSubmenu v-if="item.children && submenuExpanded['iconMenu_' + index]"
                         :children="item.children" :active-color="itemActiveBGColor" :icon-size="iconSize"
                         @submenu-clicked="handleSubmenuClick"/>

        </div>
      </nav>

      <div class="flex">
        <img v-if="user.avatar" :src="user.avatar" alt="avatar" class="max-h-10 aspect-square"/>
        <div class="flex flex-col h-full mx-2 overflow-hidden leading-3">
          <span class="font-bold text-lg truncate" :style="`color: ${itemActiveTextColor}`">{{ user.name }}</span>
          <span class="pb-1">{{ user.email }}</span>
        </div>
      </div>

      <nav class="flex flex-row sm:flex-col gap-2 h-full w-full">
        <div v-for="(item, index) in iconUserItems" :key="'iconUser_' + index" class="relative w-full">
          <div @click="handleClick(item, 'iconUser_' + index, $event)"
               @mouseenter="navbarExpanded ? () => {} : showTooltip('iconUser_' + index, $event)"
               @mouseleave="hideTooltip"
               class="navbar-button"
               :style="item.isActive ? `background-color: ${itemActiveBGColor}; color: ${itemActiveTextColor};` : `--hover-bg: ${itemHoverBGColor}; color: ${itemTextColor}; --hover-text: ${itemHoverTextColor};`">

            <!-- Icon and Title -->
            <NavbarItem :navbar-expanded="true" :item="item" :icon-size="iconSize"/>

          </div>

          <!-- Submenu -->
          <NavbarSubmenu v-if="item.children && submenuExpanded['iconUser_' + index]"
                         :children="item.children" :active-color="itemActiveBGColor" :icon-size="iconSize"
                         @submenu-clicked="handleSubmenuClick"/>

        </div>
      </nav>
    </div>
  </Teleport>
</template>

<style scoped>

/* Custom scrollbar styles */
.scrollbar-thin {
  scrollbar-width: thin;
}

.scrollbar-hidden {
  scrollbar-width: none;
}

.scrollbar-gutter-stable {
  scrollbar-gutter: stable;
}

.navbar-button {
  @apply rounded-lg w-full min-w-8 cursor-pointer transition-colors duration-300;
}

.navbar-button:hover {
  color: var(--hover-text) !important;
  background-color: var(--hover-bg);
}

.navbar-tooltip {
  @apply absolute z-20 bg-white text-gray-800 border border-gray-200 text-sm px-2 py-1 rounded shadow whitespace-nowrap pointer-events-none transition-opacity duration-1000;
}

.navbar-tooltip-arrow {
  @apply absolute top-0 sm:top-1/2 translate-y-0 sm:-translate-y-1/2 -mt-1 sm:mt-0 ml-0 sm:-ml-1 w-2 h-2 bg-white rotate-45;
}

.navbar-popup {
  @apply fixed z-10 text-white rounded-md shadow-lg p-2;
}

.navbar-badge {
  @apply text-xs text-center py-0.5 px-1 min-w-6 my-1 bg-rose-500 text-white rounded-full;
}

.popup-item {
  @apply px-2 py-1 flex items-center justify-between gap-2 rounded cursor-pointer;
}

.main-popup {
  @apply fixed z-10 flex flex-col gap-3 rounded-md shadow p-2;
}
</style>
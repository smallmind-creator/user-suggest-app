<script setup lang="ts">
import { ref, onUnmounted, nextTick, watch } from 'vue'
import type { User } from '../types/types.ts'

const selectedUser = defineModel<User | null>()

const searchQuery = ref('')
const suggestions = ref<User[]>([])
const isOpen = ref(false)
const isLoading = ref(false)
const activeIndex = ref(-1)
const isFocused = ref(false)

// Ссылки на элементы DOM
const inputRef = ref<HTMLInputElement | null>(null)
// listRef указывает на обертку (.dropdown-wrapper)
const listRef = ref<HTMLDivElement | null>(null)
const measurerRef = ref<HTMLSpanElement | null>(null)

// Позиция курсора в пикселях
const cursorPosition = ref(0)

let debounceTimer: ReturnType<typeof setTimeout> | null = null
let abortCtrl: AbortController | null = null

// Логика вычисления позиции курсора
const updateCursorPosition = () => {
  if (measurerRef.value && inputRef.value) {
    const textWidth = measurerRef.value.offsetWidth
    const scrollLeft = inputRef.value.scrollLeft
    cursorPosition.value = textWidth - scrollLeft
  }
}

watch(searchQuery, () => {
  nextTick(updateCursorPosition)
})

// API и поиск
const fetchUsers = async (query: string) => {
  if (abortCtrl) abortCtrl.abort()
  abortCtrl = new AbortController()

  if (!query.trim()) {
    suggestions.value = []
    isOpen.value = false
    return
  }

  isLoading.value = true

  try {
    const res = await fetch(`https://apimocker.com/users/search?q=${query}`, {
      signal: abortCtrl.signal,
    })

    if (!res.ok) throw new Error('API Error')

    const data = await res.json()
    suggestions.value = data.results || data || []

    if (suggestions.value.length) {
      isOpen.value = true
      activeIndex.value = -1
    } else {
      isOpen.value = false
    }
  } catch (e: any) {
    if (e.name !== 'AbortError') {
      console.error(e)
      suggestions.value = []
    }
  } finally {
    isLoading.value = false
  }
}

const onInput = (e: Event) => {
  const target = e.target as HTMLInputElement
  searchQuery.value = target.value
  updateCursorPosition()

  if (debounceTimer) clearTimeout(debounceTimer)
  debounceTimer = setTimeout(() => fetchUsers(target.value), 300)
}

const handleScroll = () => {
  updateCursorPosition()
}

const handleFocus = () => {
  isFocused.value = true
  if (searchQuery.value && suggestions.value.length) {
    isOpen.value = true
  }
  nextTick(updateCursorPosition)
}

const handleBlur = () => {
  setTimeout(() => {
    isFocused.value = false
    isOpen.value = false
  }, 200)
}

const selectItem = (user: User) => {
  selectedUser.value = user
  searchQuery.value = ''
  suggestions.value = []
  isOpen.value = false
}

const clearSelection = () => {
  selectedUser.value = null
  nextTick(() => {
    inputRef.value?.focus()
    updateCursorPosition()
  })
}

const onKeydown = (e: KeyboardEvent) => {
  setTimeout(updateCursorPosition, 0)
  /*
  if (e.key === 'Backspace' && !searchQuery.value && selectedUser.value) {
    clearSelection()
    return
  }
  */

  if (!isOpen.value) return

  if (e.key === 'ArrowDown') {
    e.preventDefault()
    if (activeIndex.value < suggestions.value.length - 1) {
      activeIndex.value++
      scrollIntoView()
    }
  } else if (e.key === 'ArrowUp') {
    e.preventDefault()
    if (activeIndex.value > 0) {
      activeIndex.value--
      scrollIntoView()
    }
  } else if (e.key === 'Enter' && activeIndex.value >= 0) {
    e.preventDefault()
    selectItem(suggestions.value[activeIndex.value] as User)
  } else if (e.key === 'Escape') {
    isOpen.value = false
    inputRef.value?.blur()
  }
}

const scrollIntoView = () => {
  if (!listRef.value) return

  // Так как listRef на обертке, ищем внутри список UL
  const listEl = listRef.value.querySelector('.dropdown-list')
  if (!listEl) return

  const el = listEl.children[activeIndex.value] as HTMLElement
  if (el) el.scrollIntoView({ block: 'nearest' })
}

onUnmounted(() => {
  if (debounceTimer) clearTimeout(debounceTimer)
})
</script>

<template>
  <div class="control-wrapper" @click="inputRef?.focus()">
    <!-- Логотип слева -->
    <svg
      class="left-icon"
      width="34"
      height="22"
      viewBox="0 0 34 22"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
    >
      <path
        d="M25.6434 21.9996H25.6548H25.6808C27.1895 21.9903 29.8288 21.088 31.4434 19.2202C32.7566 17.7005 33.3675 15.4022 33.3675 13.4601V8.52466C33.3675 3.84465 30.5473 0 24.6235 0H5.13477V5.00945H22.4192C24.9167 5.00945 27.9455 5.50273 27.9455 9.4642V15.017C27.9455 18.3375 25.8177 19.1515 23.659 19.3281H20.2881L25.6428 21.9996H25.6434Z"
        fill="#BCED0A"
      />
      <path
        fill-rule="evenodd"
        clip-rule="evenodd"
        d="M0.366981 5.12894H5.13323V16.862H0.366981C0.167198 16.862 0.00257785 16.6987 0 16.4983V5.49266C0.00257785 5.29243 0.167198 5.12894 0.366981 5.12894Z"
        fill="#BCED0A"
      />
      <path d="M15.4012 16.8667H5.13477V22H15.4012V16.8667Z" fill="#BCED0A" />
      <path
        fill-rule="evenodd"
        clip-rule="evenodd"
        d="M15.397 16.8667L25.6634 22H23.7634H15.397V16.8667Z"
        fill="#81A306"
      />
    </svg>

    <!-- Выбранный юзер -->
    <div v-if="selectedUser" class="selected-tag">
      <span class="tag-text">{{ selectedUser.name }}</span>
      <button class="tag-close-btn" @click.stop="clearSelection">
        <svg viewBox="0 0 17 17" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path
            d="M0 8.5C3.17948e-07 3.80558 3.80558 5.76746e-07 8.5 3.71547e-07C13.1944 1.66347e-07 17 3.80558 17 8.5C17 13.1944 13.1944 17 8.5 17C3.80558 17 -2.052e-07 13.1944 0 8.5ZM5.70898 5.70898C5.4774 5.94057 5.4774 6.31529 5.70898 6.54688L7.66211 8.5L5.70898 10.4531C5.47739 10.6847 5.4774 11.0594 5.70898 11.291C5.94058 11.5226 6.31529 11.5226 6.54688 11.291L8.5 9.33789L10.4531 11.291C10.6847 11.5226 11.0594 11.5226 11.291 11.291C11.5226 11.0594 11.5226 10.6847 11.291 10.4531L9.33789 8.5L11.291 6.54688C11.5226 6.31529 11.5226 5.94058 11.291 5.70898C11.0594 5.4774 10.6847 5.4774 10.4531 5.70898L8.5 7.66211L6.54688 5.70898C6.31529 5.47739 5.94058 5.4774 5.70898 5.70898Z"
            fill="#858585"
          />
        </svg>
      </button>
    </div>

    <!-- ИНПУТ ОБЛАСТЬ -->
    <div class="input-area">
      <!-- Настоящий инпут -->
      <input
        ref="inputRef"
        type="text"
        class="main-input"
        :value="searchQuery"
        @input="onInput"
        @scroll="handleScroll"
        @keydown="onKeydown"
        @focus="handleFocus"
        @blur="handleBlur"
        :disabled="!!selectedUser"
      />

      <!-- Скрытый измеритель -->
      <span ref="measurerRef" class="text-measurer">{{ searchQuery }}</span>

      <!-- Кастомный курсор -->
      <div
        v-if="isFocused && !selectedUser"
        class="custom-cursor"
        :style="{ transform: `translateX(${cursorPosition}px)` }"
      ></div>

      <!-- Плейсхолдер -->
      <div v-if="!searchQuery && !selectedUser" class="placeholder-text">Введите запрос</div>
    </div>

    <!-- Иконка лупы -->
    <div v-if="isFocused && !isLoading" class="right-icon-search">
      <svg
        width="18"
        height="18"
        viewBox="0 0 24 24"
        fill="none"
        stroke="#FFFFFF"
        stroke-opacity="0.5"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <circle cx="11" cy="11" r="8"></circle>
        <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
      </svg>
    </div>

    <div v-if="isLoading" class="loader"></div>

    <!-- Выпадающий список -->
    <div v-if="isOpen && !selectedUser" class="dropdown-wrapper" ref="listRef">
      <ul class="dropdown-list">
        <li
          v-for="(u, idx) in suggestions"
          :key="u.id || idx"
          class="dropdown-item"
          :class="{ active: idx === activeIndex }"
          @mousedown.prevent="selectItem(u)"
          @mouseenter="activeIndex = idx"
        >
          <span class="item-text">{{ u.name }}</span>
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
/* 
  ГЛАВНАЯ ЛОГИКА МАСШТАБИРОВАНИЯ
  База: 1920px. Формула: (px / 1920) * 100 = vw
*/

.control-wrapper {
  position: relative;
  width: 100%;

  /* Высота 80px -> 4.167vw */
  height: 4.167vw;

  display: flex;
  flex-direction: row;
  align-items: center;

  /* Padding 30px -> 1.56vw */
  padding: 0 1.56vw;

  /* Gap 20px -> 1.04vw */
  gap: 1.04vw;

  background: #242424;

  /* Radius 10px -> 0.52vw */
  border-radius: 0.52vw;

  font-family: 'Montserrat', sans-serif;
  box-sizing: border-box;
  z-index: 10;
}

/* Левая иконка */
.left-icon {
  display: flex;
  justify-content: center;
  align-items: center;

  /* 40px -> 2.08vw */
  width: 2.08vw;
  height: 2.08vw;
  flex: none;
}

.left-icon svg {
  width: 1.25vw; /* 24px */
  height: 1.25vw;
}

/* Область ввода */
.input-area {
  flex-grow: 1;
  position: relative;
  height: 100%;
  display: flex;
  align-items: center;

  /* Padding Right 50px -> 2.6vw */
  padding-right: 2.6vw;
  overflow: hidden;
}

/* Настоящий инпут */
.main-input {
  background: transparent;

  caret-color: transparent;

  border: none;
  outline: none;
  font-family: 'Montserrat', sans-serif;
  font-weight: 500;

  /* Font 15px -> 0.78vw */
  font-size: 0.78vw;

  line-height: 160%;
  color: #ffffff;
  width: 100%;
  height: 100%;
  position: relative;
  z-index: 2;
  padding: 0;
  margin: 0;
}

/* Кастомный курсор */
.custom-cursor {
  position: absolute;
  left: 0;
  top: 50%;
  /* 21px / 2 -> 0.55vw */
  margin-top: -0.55vw;

  /* Width 2px -> 0.1vw, Height 21px -> 1.09vw */
  width: 0.1vw;
  height: 1.1vw;

  background: #ffffff;
  border-radius: 10px;
  pointer-events: none;
  z-index: 1;
  will-change: transform;
  transition: transform 0.05s linear;
  animation: blink 1.2s infinite steps(1, start);
}

/* Плейсхолдер */
.placeholder-text {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  font-family: 'Montserrat', sans-serif;
  font-weight: 500;

  /* Font 15px */
  font-size: 0.78vw;

  color: #858585;
  pointer-events: none;
  z-index: 0;
  padding-left: 0.1vw;
}

/* Правая иконка */
.right-icon-search {
  display: flex;
  justify-content: center;
  align-items: center;

  /* 24px -> 1.25vw */
  width: 1.25vw;
  height: 1.25vw;
  flex: none;
  color: #858585;
}
.right-icon-search svg {
  width: 100%;
  height: 100%;
}

/* Измеритель текста */
.text-measurer {
  position: absolute;
  visibility: hidden;
  white-space: pre;
  font-family: 'Montserrat', sans-serif;
  font-weight: 500;
  font-size: 0.78vw;
}

/* Тег */
.selected-tag {
  display: inline-flex;
  align-items: center;
  background: #000000;

  /* Padding: 0 10px 0 20px -> 0 0.52vw 0 1.04vw */
  padding: 0 0.76vw 0 1.04vw;

  height: 50%;
  border-radius: 0.52vw;
  margin-right: 0.78vw;
  flex-shrink: 0;
}

.tag-text {
  color: #ffffff;
  font-size: 0.78vw;
  font-weight: 500;
  margin-right: 0.78vw;
}

.tag-close-btn {
  background: transparent;
  border-radius: 0;

  border: none;
  cursor: pointer;
  padding: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  width: 1vw;
  height: 1vw;
}

/* Оболочка (Wrapper) */
.dropdown-wrapper {
  position: absolute;
  /* Top Gap: 10px -> 0.52vw */
  top: calc(100% + 0.52vw);
  left: 0;
  width: 100%;

  /* Padding контейнера: 20px -> 1.04vw */
  padding: 1.04vw 0;

  background: #383836;
  /* Radius: 10px -> 0.52vw */
  border-radius: 0.52vw;
  /* Тень */
  box-shadow: 0px 0.2vw 0.2vw rgba(0, 0, 0, 0.25);

  z-index: 100;
}

/* Список (List) - отвечает за скролл */
.dropdown-list {
  list-style: none;
  margin: 0;
  padding: 0;
  width: 100%;

  /* 
     Max Height = 4 items * 60px/item
     60px -> 3.125vw
     4 * 3.125vw = 12.5vw
  */
  max-height: 12.5vw;

  overflow-y: auto;
  overflow-x: hidden;
}

/* Элемент списка */
.dropdown-item {
  /* Height: 60px -> 3.125vw */
  height: 3.125vw;

  /* Padding: 30px -> 1.56vw */
  padding: 0 1.56vw;

  display: flex;
  align-items: center;
  cursor: pointer;
  transition: background-color 0.1s;
}

.dropdown-item.active,
.dropdown-item:hover {
  background-color: #242424;
}

.item-text {
  color: #ffffff;
  font-family: 'Montserrat', sans-serif;
  font-size: 0.78vw; /* 15px */
  font-weight: 500;
}

/* Скроллбар */
.dropdown-list::-webkit-scrollbar {
  width: 0.3vw;
}
.dropdown-list::-webkit-scrollbar-track {
  background: transparent;
}
.dropdown-list::-webkit-scrollbar-thumb {
  background-color: #555;
  border-radius: 0.15vw;
}

/* Лоадер */
.loader {
  width: 0.83vw; /* 16px */
  height: 0.83vw;
  border: 0.1vw solid #555;
  border-bottom-color: transparent;
  border-radius: 50%;
  animation: rotation 1s linear infinite;
}

@keyframes rotation {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

@keyframes blink {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}
</style>

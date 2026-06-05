<template>
  <div class="slides-bar-container">
    <!-- 悬浮打开幻灯片面板按钮 -->
    <Transition name="fade">
      <div class="open-slides-btn" v-show="!slidesBarShow" @click="switchSlidesBar">
        <Tooltip :content="$t('slides.title')" placement="left">
          <Icon type="md-images" size="20" />
        </Tooltip>
      </div>
    </Transition>

    <!-- 幻灯片管理侧边栏 -->
    <Transition name="fade-slide-right">
      <div class="slides-bar" v-show="slidesBarShow">
        <!-- 面板头部 -->
        <div class="panel-header">
          <span class="panel-title">{{ $t('slides.title') }}</span>
          <Icon type="md-close" size="18" class="close-icon" @click="switchSlidesBar" />
        </div>

        <!-- 顶部操作栏 -->
        <div class="action-header">
          <Tooltip :content="$t('slides.add')" placement="bottom">
            <Button type="text" icon="md-add" size="large" @click="addNewSlide" />
          </Tooltip>
          <Tooltip :content="$t('slides.present')" placement="bottom">
            <Button type="text" icon="md-play" size="large" @click="startPresentation" />
          </Tooltip>
        </div>

        <!-- 幻灯片列表 -->
        <div class="panel-body">
          <div
            v-for="(slide, index) in slides"
            :key="slide.id"
            class="slide-card-wrapper"
            :class="{ active: slide.id === activeSlideId }"
            draggable="true"
            @dragstart="onDragStart($event, index)"
            @dragover.prevent
            @drop="onDrop($event, index)"
            @click="switchSlide(slide.id)"
          >
            <!-- 编号与重命名 (图2结构：外置在顶部) -->
            <div class="slide-header">
              <span class="slide-num">{{ index + 1 }}</span>
              <div class="slide-title-box" @click.stop>
                <Input
                  v-if="editingId === slide.id"
                  v-model="editTitleStr"
                  size="small"
                  @on-blur="saveRename(slide.id)"
                  @on-enter="saveRename(slide.id)"
                  ref="renameInput"
                  style="width: 130px"
                />
                <span v-else class="slide-title-text" @dblclick="startRename(slide)">
                  {{ slide.title }}
                </span>
              </div>
            </div>

            <!-- 缩略图盒子 (图2结构：高亮边框和圆角应用于此，下拉悬浮其中) -->
            <div class="slide-thumbnail-box">
              <img v-if="slide.thumbnail" :src="slide.thumbnail" alt="Slide Thumbnail" />
              <div v-else class="empty-thumbnail">
                <Icon type="ios-image" size="32" color="#cbd5e1" />
              </div>

              <!-- 右上角悬浮的下拉控制菜单 -->
              <Dropdown
                trigger="click"
                @on-click="handleAction($event, slide)"
                @click.stop
                class="slide-menu"
              >
                <div class="more-btn-wrapper">
                  <Icon type="ios-more" size="16" class="more-icon" />
                </div>
                <template #list>
                  <DropdownMenu>
                    <DropdownItem name="rename">{{ $t('slides.rename') }}</DropdownItem>
                    <DropdownItem name="duplicate">{{ $t('slides.duplicate') }}</DropdownItem>
                    <DropdownItem name="delete" :disabled="slides.length <= 1" class="delete-item">
                      {{ $t('slides.delete') }}
                    </DropdownItem>
                  </DropdownMenu>
                </template>
              </Dropdown>
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <!-- 全屏演示播放模式 (Presenter Mode) -->
    <div
      v-if="isPresenting"
      class="presenter-overlay"
      @keydown="handleKeydown"
      tabindex="0"
      ref="presenterRef"
    >
      <!-- 顶部状态栏 -->
      <div class="presenter-top-bar">
        <span class="presenter-title">
          {{ slides[presentIndex]?.title || $t('slides.untitled') }}
        </span>
        <Button type="text" icon="md-close" class="exit-btn" @click="exitPresentation">
          {{ $t('slides.exit_present') }}
        </Button>
      </div>

      <!-- 中央幻灯片展示区 -->
      <div class="presenter-view-area">
        <Transition name="fade" mode="out-in">
          <div :key="presentIndex" class="presenter-img-box">
            <img
              v-if="slides[presentIndex]?.thumbnail"
              :src="slides[presentIndex].thumbnail"
              alt="Present Slide"
            />
            <div v-else class="presenter-empty">
              <Icon type="ios-image" size="120" color="#475569" />
            </div>
          </div>
        </Transition>
      </div>

      <!-- 底部控制栏 -->
      <div class="presenter-controls">
        <Button
          type="text"
          icon="ios-arrow-back"
          size="large"
          :disabled="presentIndex === 0"
          @click="presentIndex--"
        >
          {{ $t('slides.prev_page') }}
        </Button>
        <span class="presenter-page-info">Slide {{ presentIndex + 1 }} / {{ slides.length }}</span>
        <Button
          type="text"
          icon="ios-arrow-forward"
          size="large"
          :disabled="presentIndex === slides.length - 1"
          @click="presentIndex++"
        >
          {{ $t('slides.next_page') }}
        </Button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, inject, nextTick, onMounted, onUnmounted, type Ref } from 'vue';
import { Message, Modal } from 'view-ui-plus';

interface Slide {
  id: string;
  title: string;
  thumbnail: string;
  canvasData: any;
}

// 注入顶级状态与动作
const slides = inject('slides', ref<Slide[]>([])) as Ref<Slide[]>;
const activeSlideId = inject('activeSlideId', ref('')) as Ref<string>;
const switchSlide = inject('switchSlide') as (id: string) => Promise<void>;
const addNewSlide = inject('addNewSlide') as () => Promise<void>;
const duplicateSlide = inject('duplicateSlide') as (slide: Slide) => Promise<void>;
const deleteSlide = inject('deleteSlide') as (id: string) => Promise<void>;
const renameSlide = inject('renameSlide') as (id: string, title: string) => void;
const updateSlidesOrder = inject('updateSlidesOrder') as (newSlides: Slide[]) => void;
const updateCurrentSlideData = inject('updateCurrentSlideData') as () => Promise<void>;

// 全局侧栏状态，与属性面板联动
const slidesBarShow = inject('slidesBarShow', ref(true)) as Ref<boolean>;
const switchSlidesBar = () => {
  slidesBarShow.value = !slidesBarShow.value;
};

// 重命名编辑状态
const editingId = ref('');
const editTitleStr = ref('');
const renameInput = ref<any>(null);

const startRename = (slide: Slide) => {
  editingId.value = slide.id;
  editTitleStr.value = slide.title;
  nextTick(() => {
    if (renameInput.value && renameInput.value[0]) {
      renameInput.value[0].focus();
    }
  });
};

const saveRename = (slideId: string) => {
  if (editTitleStr.value.trim() !== '') {
    renameSlide(slideId, editTitleStr.value.trim());
  }
  editingId.value = '';
};

// 卡片菜单选项点击事件
const handleAction = (name: string, slide: Slide) => {
  if (name === 'rename') {
    startRename(slide);
  } else if (name === 'duplicate') {
    duplicateSlide(slide);
  } else if (name === 'delete') {
    if (slides.value.length <= 1) {
      Message.warning('必须保留至少一个幻灯片');
      return;
    }
    Modal.confirm({
      title: '提示',
      content: '<p>确定删除该幻灯片吗？</p>',
      onOk: () => {
        deleteSlide(slide.id);
      },
    });
  }
};

// 拖拽排序逻辑
let dragIndex = -1;
const onDragStart = (e: DragEvent, index: number) => {
  dragIndex = index;
  if (e.dataTransfer) {
    e.dataTransfer.effectAllowed = 'move';
  }
};

const onDrop = (e: DragEvent, index: number) => {
  if (dragIndex === -1 || dragIndex === index) return;
  const list = [...slides.value];
  const dragged = list[dragIndex];
  list.splice(dragIndex, 1);
  list.splice(index, 0, dragged);
  updateSlidesOrder(list);
  dragIndex = -1;
};

// 演示播放逻辑
const isPresenting = ref(false);
const presentIndex = ref(0);
const presenterRef = ref<HTMLElement | null>(null);

const startPresentation = async () => {
  // 演示前先保存当前幻灯片的最新状态并更新缩略图
  if (updateCurrentSlideData) {
    await updateCurrentSlideData();
  }

  // 默认从当前活跃幻灯片开始播放
  const activeIdx = slides.value.findIndex((s) => s.id === activeSlideId.value);
  presentIndex.value = activeIdx !== -1 ? activeIdx : 0;
  isPresenting.value = true;

  nextTick(() => {
    if (presenterRef.value) {
      presenterRef.value.focus();
    }
  });
};

const exitPresentation = () => {
  isPresenting.value = false;
};

// 演示模式键盘监听
const handleKeydown = (e: KeyboardEvent) => {
  if (!isPresenting.value) return;

  if (e.key === 'ArrowRight' || e.key === ' ' || e.key === 'Enter') {
    if (presentIndex.value < slides.value.length - 1) {
      presentIndex.value++;
    }
    e.preventDefault();
  } else if (e.key === 'ArrowLeft' || e.key === 'Backspace') {
    if (presentIndex.value > 0) {
      presentIndex.value--;
    }
    e.preventDefault();
  } else if (e.key === 'Escape') {
    exitPresentation();
    e.preventDefault();
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
});
</script>

<style scoped lang="less">
.slides-bar-container {
  position: relative;
  z-index: 1000;
}

// 打开侧边栏悬浮按钮 (放置在最右侧边缘)
.open-slides-btn {
  position: absolute;
  right: 15px;
  top: 15px;
  width: 40px;
  height: 40px;
  background: #ffffff;
  border-radius: 50%;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.08);
  border: 1px solid rgba(0, 0, 0, 0.06);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: #64748b;
  transition: all 0.2s ease;
  z-index: 1001;

  &:hover {
    background: rgba(0, 0, 0, 0.03);
    color: #1e293b;
    transform: scale(1.05);
  }
}

// 侧边栏整体容器 (定位在最右侧)
.slides-bar {
  position: absolute;
  right: 12px;
  top: 12px;
  bottom: 12px;
  width: 240px;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
  border: 1px solid rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  z-index: 1000;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px;
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
}

.panel-title {
  font-size: 14px;
  font-weight: 600;
  color: #1e293b;
}

.close-icon {
  cursor: pointer;
  color: #64748b;
  padding: 4px;
  border-radius: 50%;
  transition: all 0.2s ease;

  &:hover {
    background: rgba(0, 0, 0, 0.05);
    color: #1e293b;
  }
}

// 顶部操作栏
.action-header {
  display: flex;
  justify-content: space-around;
  padding: 8px 16px;
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  background: #f8fafc;

  :deep(.ivu-btn-text) {
    color: #475569;
    padding: 4px 12px;
    border-radius: 6px;

    &:hover {
      background: rgba(0, 0, 0, 0.05);
      color: #0f172a;
    }
  }
}

// 侧栏主体列表
.panel-body {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 20px;

  &::-webkit-scrollbar {
    width: 6px;
  }
  &::-webkit-scrollbar-thumb {
    background: rgba(0, 0, 0, 0.15);
    border-radius: 3px;
  }
}

// 幻灯片卡片包裹器 (对齐图2：去除原有背景边框，变成无感点击层)
.slide-card-wrapper {
  background: transparent;
  cursor: pointer;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
  gap: 6px;
  user-select: none;
}

.slide-header {
  display: flex;
  align-items: center;
  gap: 6px;
}

.slide-num {
  font-size: 12px;
  font-weight: 600;
  color: #64748b;
}

.slide-title-box {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.slide-title-text {
  font-size: 12px;
  font-weight: 500;
  color: #64748b;
  cursor: text;
}

// 缩略图样式 (圆角、边框、投影均在此应用，以对齐图2)
.slide-thumbnail-box {
  position: relative;
  aspect-ratio: 1.6;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.02);
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);

  img {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .empty-thumbnail {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  // 鼠标悬浮在卡片缩略图上时，展示 ... 下拉菜单图标
  &:hover {
    .slide-menu {
      opacity: 1;
    }
  }
}

// 悬浮在缩略图右上角的 Dropdown 菜单
.slide-menu {
  position: absolute;
  right: 8px;
  top: 8px;
  z-index: 10;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.more-btn-wrapper {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.2s ease;
  cursor: pointer;

  &:hover {
    background: #f1f5f9;
    border-color: #cbd5e1;
  }
}

.more-icon {
  color: #64748b;
}

.delete-item {
  color: #ef4444 !important;
  &:hover {
    background: #fef2f2 !important;
  }
}

// 激活选中状态样式 (高亮缩略图边框与阴影，对齐图2)
.active {
  .slide-title-text,
  .slide-num {
    color: #1e293b;
    font-weight: 600;
  }

  .slide-thumbnail-box {
    border-color: #1565c0;
    box-shadow: 0 0 0 2px rgba(21, 101, 192, 0.15), 0 4px 12px rgba(21, 101, 192, 0.1);
  }
}

// 演示播放全屏遮罩
.presenter-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #090d16;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  outline: none;
}

.presenter-top-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  background: rgba(15, 23, 42, 0.8);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);

  .presenter-title {
    color: #f8fafc;
    font-size: 16px;
    font-weight: 600;
  }

  .exit-btn {
    color: #94a3b8 !important;
    font-size: 14px;
    &:hover {
      color: #f8fafc !important;
    }
  }
}

.presenter-view-area {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

.presenter-img-box {
  max-width: 90vw;
  max-height: 70vh;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.5);
  background: #ffffff;
  border-radius: 8px;
  overflow: hidden;

  img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
  }

  .presenter-empty {
    width: 600px;
    height: 450px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #1e293b;
  }
}

.presenter-controls {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 24px;
  padding: 24px;
  background: rgba(15, 23, 42, 0.8);
  border-top: 1px solid rgba(255, 255, 255, 0.05);

  :deep(.ivu-btn-text) {
    color: #cbd5e1;
    &:hover:not([disabled]) {
      color: #f8fafc;
      background: rgba(255, 255, 255, 0.05);
    }
    &[disabled] {
      color: #475569;
    }
  }

  .presenter-page-info {
    color: #f8fafc;
    font-size: 14px;
    font-weight: 500;
  }
}

// 动画
.fade-slide-right-enter-active,
.fade-slide-right-leave-active {
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.fade-slide-right-enter-from,
.fade-slide-right-leave-to {
  opacity: 0;
  transform: translateX(15px);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>

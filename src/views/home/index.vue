<!--
 * @Author: 秦少卫
 * @Date: 2024-05-17 15:30:21
 * @LastEditors: 秦少卫
 * @LastEditTime: 2024-11-18 10:30:30
 * @Description: file content
-->
<template>
  <div class="home">
    <Layout>
      <!-- 头部区域 -->
      <Top v-if="state.show" :ruler="state.ruler" @update:ruler="rulerSwitch"></Top>
      <Content
        style="display: flex; height: calc(100vh - 45px); position: relative; overflow: hidden"
      >
        <!-- 左侧区域 -->
        <Left v-if="state.show"></Left>
        <!-- 画布区域 -->
        <div id="workspace">
          <div class="canvas-box">
            <div class="inside-shadow"></div>
            <canvas id="canvas" :class="state.ruler ? 'design-stage-grid' : ''"></canvas>
            <dragMode v-if="state.show"></dragMode>
            <zoom></zoom>
          </div>
        </div>
        <Slides v-if="state.show"></Slides>
        <Right v-if="state.show"></Right>
      </Content>
    </Layout>
  </div>
</template>

<script name="Home" setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, nextTick, provide, computed } from 'vue';
import Top from './components/top/index.vue';
import Left from './components/left/index.vue';
import Right from './components/right/index.vue';
import Slides from './components/right/Slides.vue';
import { v4 as uuid } from 'uuid';
import { useDebounceFn } from '@vueuse/core';

import zoom from '@/components/zoom.vue';
import dragMode from '@/components/dragMode.vue';
// 功能组件
import { fabric } from 'fabric';

import Editor, {
  IEditor,
  DringPlugin,
  AlignGuidLinePlugin,
  ControlsPlugin,
  // ControlsRotatePlugin,
  CenterAlignPlugin,
  LayerPlugin,
  CopyPlugin,
  MoveHotKeyPlugin,
  DeleteHotKeyPlugin,
  GroupPlugin,
  DrawLinePlugin,
  GroupTextEditorPlugin,
  GroupAlignPlugin,
  WorkspacePlugin,
  HistoryPlugin,
  FlipPlugin,
  RulerPlugin,
  MaterialPlugin,
  WaterMarkPlugin,
  FontPlugin,
  PolygonModifyPlugin,
  DrawPolygonPlugin,
  FreeDrawPlugin,
  PathTextPlugin,
  PsdPlugin,
  SimpleClipImagePlugin,
  BarCodePlugin,
  QrCodePlugin,
  ImageStroke,
  ResizePlugin,
  LockPlugin,
  AddBaseTypePlugin,
  MaskPlugin,
} from '@kuaitu/core';

const APIHOST = import.meta.env.APP_APIHOST;

// 创建编辑器
const canvasEditor = new Editor() as IEditor;

const state = reactive({
  show: false,
  select: null,
  ruler: true,
});

onMounted(async () => {
  // 初始化fabric
  const canvas = new fabric.Canvas('canvas', {
    fireRightClick: true, // 启用右键，button的数字为3
    stopContextMenu: true, // 禁止默认右键菜单
    controlsAboveOverlay: true, // 超出clipPath后仍然展示控制条
    // imageSmoothingEnabled: false, // 解决文字导出后不清晰问题
    preserveObjectStacking: true, // 当选择画布中的对象时，让对象不在顶层。
  });

  // 初始化编辑器
  canvasEditor.init(canvas);
  canvasEditor
    .use(DringPlugin)
    .use(PolygonModifyPlugin)
    .use(AlignGuidLinePlugin)
    .use(ControlsPlugin)
    // .use(ControlsRotatePlugin)
    .use(CenterAlignPlugin)
    .use(LayerPlugin)
    .use(CopyPlugin)
    .use(MoveHotKeyPlugin)
    .use(DeleteHotKeyPlugin)
    .use(GroupPlugin)
    .use(DrawLinePlugin)
    .use(GroupTextEditorPlugin)
    .use(GroupAlignPlugin)
    .use(WorkspacePlugin)
    .use(HistoryPlugin)
    .use(FlipPlugin)
    .use(RulerPlugin)
    .use(DrawPolygonPlugin)
    .use(FreeDrawPlugin)
    .use(PathTextPlugin)
    .use(SimpleClipImagePlugin)
    .use(BarCodePlugin)
    .use(QrCodePlugin)
    .use(FontPlugin, {
      repoSrc: APIHOST,
    })
    .use(MaterialPlugin, {
      repoSrc: APIHOST,
    })
    .use(WaterMarkPlugin)
    .use(PsdPlugin)
    .use(ImageStroke)
    .use(ResizePlugin)
    .use(LockPlugin)
    .use(AddBaseTypePlugin)
    .use(MaskPlugin);

  state.show = true;
  // 默认打开标尺
  if (state.ruler) {
    canvasEditor.rulerEnable();
  }

  // 初始化多画布 Slides
  await initSlides();

  // 监听画布的变动，更新缩略图和 canvasData
  canvasEditor.canvas.on('object:added', updateCurrentSlideData);
  canvasEditor.canvas.on('object:modified', updateCurrentSlideData);
  canvasEditor.canvas.on('object:removed', updateCurrentSlideData);
  // 监听标尺或自定义尺寸改变
  canvasEditor.on('sizeChange', updateCurrentSlideData);
  canvasEditor.on('loadJson', updateCurrentSlideData);
});

onUnmounted(() => canvasEditor.destory());
const rulerSwitch = (val) => {
  if (val) {
    canvasEditor.rulerEnable();
  } else {
    canvasEditor.rulerDisable();
  }
  // 使标尺开关组件失焦，避免响应键盘的空格事件
  document.activeElement.blur();
};

// --- Slides 多页状态管理开始 ---
interface Slide {
  id: string;
  title: string;
  thumbnail: string;
  canvasData: any;
}

const slides = ref<Slide[]>([]);
const activeSlideId = ref<string>('');
const slidesBarShow = ref(true);

// 初始化 slides 数据，将当前已经就绪的画布状态存为第 1 页
const initSlides = async () => {
  const initialId = uuid();
  const initialJson = canvasEditor.getJson();
  slides.value = [
    {
      id: initialId,
      title: '1 Untitled',
      thumbnail: '',
      canvasData: initialJson,
    },
  ];
  activeSlideId.value = initialId;

  // 渲染第一帧缩略图
  nextTick(async () => {
    try {
      slides.value[0].thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error('Failed to generate initial slide thumbnail', e);
    }
  });
};

// 切换 slide
const switchSlide = async (slideId: string) => {
  if (slideId === activeSlideId.value) return;

  // 1. 保存当前活跃幻灯片最新状态
  const currentSlide = slides.value.find((s) => s.id === activeSlideId.value);
  if (currentSlide) {
    currentSlide.canvasData = canvasEditor.getJson();
    try {
      currentSlide.thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error(e);
    }
  }

  // 2. 切换当前 ID
  activeSlideId.value = slideId;

  // 3. 加载目标 slide 数据到画布上
  const targetSlide = slides.value.find((s) => s.id === slideId);
  if (targetSlide) {
    canvasEditor.clear();
    await canvasEditor.loadJSON(targetSlide.canvasData);
  }
};

// 新增空白 slide
const addNewSlide = async () => {
  // 1. 保存当前 slide 状态
  const currentSlide = slides.value.find((s) => s.id === activeSlideId.value);
  if (currentSlide) {
    currentSlide.canvasData = canvasEditor.getJson();
    try {
      currentSlide.thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error(e);
    }
  }

  // 2. 获取当前 workspace 宽高和背景填充，用于新建页的继承
  const workspace = canvasEditor.canvas.getObjects().find((o: any) => o.id === 'workspace');
  const width = workspace?.width || 900;
  const height = workspace?.height || 1200;
  const fill = workspace?.fill || 'rgba(255,255,255,1)';

  // 3. 构建空白 JSON 画布数据
  canvasEditor.clear();
  canvasEditor.setSize(width, height);
  canvasEditor.setWorkspaseBg(fill);
  const emptyCanvasData = canvasEditor.getJson();

  // 4. 创建新页并切换
  const newId = uuid();
  const newSlide: Slide = {
    id: newId,
    title: `${slides.value.length + 1} Untitled`,
    thumbnail: '',
    canvasData: emptyCanvasData,
  };
  slides.value.push(newSlide);
  activeSlideId.value = newId;

  // 生成缩略图
  nextTick(async () => {
    try {
      newSlide.thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error(e);
    }
  });
};

// 复制 slide
const duplicateSlide = async (slide: Slide) => {
  if (slide.id === activeSlideId.value) {
    slide.canvasData = canvasEditor.getJson();
    try {
      slide.thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error(e);
    }
  }

  const newSlide: Slide = {
    id: uuid(),
    title: `${slide.title} Copy`,
    thumbnail: slide.thumbnail,
    canvasData: JSON.parse(JSON.stringify(slide.canvasData)),
  };

  const idx = slides.value.findIndex((s) => s.id === slide.id);
  if (idx !== -1) {
    slides.value.splice(idx + 1, 0, newSlide);
  } else {
    slides.value.push(newSlide);
  }

  await switchSlide(newSlide.id);
};

// 删除 slide
const deleteSlide = async (slideId: string) => {
  if (slides.value.length <= 1) return;

  const idx = slides.value.findIndex((s) => s.id === slideId);
  if (idx === -1) return;

  const isActive = slideId === activeSlideId.value;
  slides.value.splice(idx, 1);

  if (isActive) {
    const nextIdx = idx === 0 ? 0 : idx - 1;
    const nextSlide = slides.value[nextIdx];
    activeSlideId.value = nextSlide.id;
    canvasEditor.clear();
    await canvasEditor.loadJSON(nextSlide.canvasData);
  }
};

// 重命名 slide
const renameSlide = (slideId: string, title: string) => {
  const slide = slides.value.find((s) => s.id === slideId);
  if (slide) {
    slide.title = title;
  }
};

// 拖拽排序后更新 slides 列表
const updateSlidesOrder = (newSlides: Slide[]) => {
  slides.value = newSlides;
};

// 防抖更新当前画布最新的缩略图和 canvasData 到状态中
const updateCurrentSlideData = useDebounceFn(async () => {
  const currentSlide = slides.value.find((s) => s.id === activeSlideId.value);
  if (currentSlide && canvasEditor) {
    try {
      currentSlide.canvasData = canvasEditor.getJson();
      currentSlide.thumbnail = await canvasEditor.preview();
    } catch (e) {
      console.error('Failed to update slide thumbnail', e);
    }
  }
}, 1000);
// --- Slides 多页状态管理结束 ---

provide('fabric', fabric);
provide('canvasEditor', canvasEditor);
provide('slides', slides);
provide('activeSlideId', activeSlideId);
provide('switchSlide', switchSlide);
provide('addNewSlide', addNewSlide);
provide('duplicateSlide', duplicateSlide);
provide('deleteSlide', deleteSlide);
provide('renameSlide', renameSlide);
provide('updateSlidesOrder', updateSlidesOrder);
provide('updateCurrentSlideData', updateCurrentSlideData);
provide('slidesBarShow', slidesBarShow);
// provide('mixinState', mixinState);
</script>

<style lang="less" scoped>
:deep(.ivu-layout-header) {
  --height: 45px;
  padding: 0 0px;
  border-bottom: 1px solid #eef2f8;
  background: #fff;
  height: var(--height);
  line-height: var(--height);
  display: flex;
  justify-content: space-between;
}

.home,
.ivu-layout {
  height: 100vh;
}

.icon {
  display: block;
}

.canvas-box {
  position: relative;
}

// 画布内阴影
.inside-shadow {
  position: absolute;
  width: 100%;
  height: 100%;
  box-shadow: inset 0 0 9px 2px #0000001f;
  z-index: 2;
  pointer-events: none;
}

#canvas {
  width: 300px;
  height: 300px;
  margin: 0 auto;
}

#workspace {
  flex: 1;
  width: 100%;
  position: relative;
  background: #f1f1f1;
  overflow: hidden;
}

// 标尺
.switch {
  margin-right: 10px;
}

// 网格背景
.design-stage-grid {
  --offsetX: 0px;
  --offsetY: 0px;
  --size: 16px;
  --color: #dedcdc;
  background-image: linear-gradient(
      45deg,
      var(--color) 25%,
      transparent 0,
      transparent 75%,
      var(--color) 0
    ),
    linear-gradient(45deg, var(--color) 25%, transparent 0, transparent 75%, var(--color) 0);
  background-position: var(--offsetX) var(--offsetY),
    calc(var(--size) + var(--offsetX)) calc(var(--size) + var(--offsetY));
  background-size: calc(var(--size) * 2) calc(var(--size) * 2);
}
</style>

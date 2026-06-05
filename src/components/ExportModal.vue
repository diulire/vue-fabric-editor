<template>
  <Modal
    v-model="visible"
    :title="$t('slides.export_title')"
    width="900"
    :footer-hide="true"
    class="export-modal"
  >
    <div class="export-modal-content">
      <!-- 左栏：预览区域 -->
      <div class="preview-panel">
        <div class="preview-container">
          <img
            v-if="previewSlides[previewIndex]?.thumbnail"
            :src="previewSlides[previewIndex].thumbnail"
            alt="Preview"
          />
          <div v-else class="empty-preview">
            <Icon type="ios-image" size="48" color="#cbd5e1" />
          </div>
        </div>
        <!-- 页码切换 -->
        <div class="pagination" v-if="previewSlides.length > 0">
          <Button
            type="text"
            icon="ios-arrow-back"
            :disabled="previewIndex === 0"
            @click="previewIndex--"
          >
            {{ $t('slides.prev_page') }}
          </Button>
          <span class="page-info">Slide {{ previewIndex + 1 }} / {{ previewSlides.length }}</span>
          <Button
            type="text"
            icon="ios-arrow-forward"
            :disabled="previewIndex === previewSlides.length - 1"
            @click="previewIndex++"
          >
            {{ $t('slides.next_page') }}
          </Button>
        </div>
      </div>

      <!-- 右栏：配置选项区域 -->
      <div class="config-panel">
        <!-- 范围选择 -->
        <div class="form-item">
          <label class="form-label">{{ $t('slides.select_slides') }}</label>
          <Select v-model="exportRange" @on-change="handleRangeChange">
            <Option value="all">All Slides (1-{{ slides.length }})</Option>
            <Option value="current">{{ $t('slides.current_slide') }}</Option>
            <Option value="custom">{{ $t('slides.custom_range') }}</Option>
          </Select>
          <Input
            v-if="exportRange === 'custom'"
            v-model="customRangeStr"
            :placeholder="$t('slides.custom_range_placeholder')"
            style="margin-top: 8px"
            @on-blur="parseCustomRange"
          />
        </div>

        <!-- 文件类型 -->
        <div class="form-item">
          <label class="form-label">{{ $t('slides.file_type') }}</label>
          <Select v-model="fileType">
            <Option value="PNG">PNG</Option>
            <Option value="JPG">JPG</Option>
          </Select>
          <div v-if="fileType === 'PNG'" class="checkbox-item">
            <Checkbox v-model="transparentBg">
              {{ $t('slides.transparent_bg') }}
            </Checkbox>
          </div>
        </div>

        <!-- 尺寸规格 -->
        <div class="form-item">
          <label class="form-label">{{ $t('slides.dimensions') }}</label>
          <div class="dimension-inputs">
            <InputNumber
              v-model="customWidth"
              :disabled="isDimensionDisabled"
              style="width: 100px"
              :min="1"
            />
            <span class="dim-separator">x</span>
            <InputNumber
              v-model="customHeight"
              :disabled="isDimensionDisabled"
              style="width: 100px"
              :min="1"
            />
            <span class="dim-unit">px</span>
          </div>
          <div v-if="isDimensionDisabled" class="dim-tip">
            {{ $t('slides.dimension_disabled_tip') }}
          </div>
        </div>

        <!-- 分辨率 DPI -->
        <div class="form-item">
          <label class="form-label">{{ $t('slides.resolution') }}</label>
          <Select v-model="dpi">
            <Option value="72">{{ $t('slides.dpi_72') }}</Option>
            <Option value="150">{{ $t('slides.dpi_150') }}</Option>
            <Option value="300">{{ $t('slides.dpi_300') }}</Option>
          </Select>
        </div>

        <!-- 下载按钮 -->
        <div class="footer-btn">
          <Button type="primary" long size="large" @click="handleDownload">
            {{ $t('slides.download') }}
          </Button>
        </div>
      </div>
    </div>
  </Modal>
</template>

<script setup lang="ts">
import { ref, computed, watch, inject, type Ref } from 'vue';
import { Message, Spin } from 'view-ui-plus';

interface Slide {
  id: string;
  title: string;
  thumbnail: string;
  canvasData: any;
}

const props = defineProps<{
  modelValue: boolean;
}>();

const emit = defineEmits(['update:modelValue']);

const canvasEditor: any = inject('canvasEditor');
const slides = inject('slides', ref<Slide[]>([])) as Ref<Slide[]>;
const activeSlideId = inject('activeSlideId', ref('')) as Ref<string>;

const visible = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val),
});

// 状态
const exportRange = ref('all'); // all, current, custom
const customRangeStr = ref('');
const fileType = ref('PNG');
const transparentBg = ref(false);
const dpi = ref('72'); // 72, 150, 300

const customWidth = ref<number | null>(null);
const customHeight = ref<number | null>(null);

const previewIndex = ref(0);

// 解析得到的导出幻灯片列表
const targetSlides = computed(() => {
  if (exportRange.value === 'all') {
    return slides.value;
  } else if (exportRange.value === 'current') {
    const cur = slides.value.find((s) => s.id === activeSlideId.value);
    return cur ? [cur] : [];
  } else {
    // 解析自定义范围，例如 "1,3-5" -> [0, 2, 3, 4]
    const list: Slide[] = [];
    if (!customRangeStr.value.trim()) return [];
    const ranges = customRangeStr.value.split(',');
    for (const r of ranges) {
      if (r.includes('-')) {
        const parts = r.split('-');
        const start = parseInt(parts[0], 10);
        const end = parseInt(parts[1], 10);
        if (!isNaN(start) && !isNaN(end)) {
          for (let i = start; i <= end; i++) {
            const slide = slides.value[i - 1];
            if (slide && !list.includes(slide)) {
              list.push(slide);
            }
          }
        }
      } else {
        const idx = parseInt(r, 10);
        if (!isNaN(idx)) {
          const slide = slides.value[idx - 1];
          if (slide && !list.includes(slide)) {
            list.push(slide);
          }
        }
      }
    }
    return list;
  }
});

// 左侧可预览的幻灯片（若解析不出自定义，则退回到所有）
const previewSlides = computed(() => {
  return targetSlides.value.length > 0 ? targetSlides.value : slides.value;
});

// 只有当导出的幻灯片页数大于 1 时，置灰尺寸输入框
const isDimensionDisabled = computed(() => {
  return targetSlides.value.length > 1;
});

// 监听弹窗显示，初始化默认的尺寸
watch(visible, (val) => {
  if (val) {
    previewIndex.value = 0;
    // 默认获取当前 workspace 的尺寸
    const workspace = canvasEditor.canvas.getObjects().find((o: any) => o.id === 'workspace');
    if (workspace) {
      customWidth.value = workspace.width || 900;
      customHeight.value = workspace.height || 1200;
    }
  }
});

const handleRangeChange = () => {
  previewIndex.value = 0;
};

const parseCustomRange = () => {
  if (
    exportRange.value === 'custom' &&
    targetSlides.value.length === 0 &&
    customRangeStr.value.trim() !== ''
  ) {
    Message.warning('请输入正确的页码范围');
  }
};

// 核心下载逻辑
const handleDownload = async () => {
  const slidesToExport = targetSlides.value;
  if (slidesToExport.length === 0) {
    Message.warning('没有可导出的幻灯片');
    return;
  }

  // 计算 multiplier (基于 72 DPI 作为 1)
  let multiplier = 1;
  if (dpi.value === '150') multiplier = 2;
  if (dpi.value === '300') multiplier = 4;

  // 备份当前的活跃 ID 和 JSON 数据，避免导出过程覆盖用户画布
  const originalActiveId = activeSlideId.value;
  const originalData = canvasEditor.getJson();

  Spin.show({
    render: (h: any) => {
      return h('div', { class: 'export-spin-content' }, [
        h('Icon', {
          class: 'export-spin-icon',
          props: {
            type: 'ios-loading',
            size: 24,
          },
        }),
        h('div', '正在导出多页，请稍候...'),
      ]);
    },
  });

  try {
    for (let i = 0; i < slidesToExport.length; i++) {
      const slide = slidesToExport[i];
      // 1. 临时载入画布数据
      canvasEditor.clear();
      await canvasEditor.loadJSON(slide.canvasData);

      // 2. 导出设置
      const workspace = canvasEditor.canvas.getObjects().find((o: any) => o.id === 'workspace');
      const { left, top, width, height } = workspace || {
        left: 0,
        top: 0,
        width: 900,
        height: 1200,
      };

      // 如果是 PNG 且勾选了透明背景
      const originalFill = workspace?.fill;
      if (fileType.value === 'PNG' && transparentBg.value && workspace) {
        workspace.set('fill', 'rgba(0,0,0,0)');
        canvasEditor.canvas.renderAll();
      }

      // 如果是单页导出且用户修改了尺寸
      let customMultiplier = multiplier;
      if (slidesToExport.length === 1 && customWidth.value && customHeight.value) {
        customMultiplier = (customWidth.value / width) * multiplier;
      }

      const option = {
        format: fileType.value.toLowerCase(), // png, jpeg
        quality: 1,
        width,
        height,
        left,
        top,
        multiplier: customMultiplier,
      };

      let dataUrl = '';
      if (workspace) {
        // 利用克隆作为裁切区域，裁掉画布外多出的元素
        const cloned = await new Promise<any>((resolve) => workspace.clone((c: any) => resolve(c)));
        canvasEditor.canvas.clipPath = cloned;
        canvasEditor.canvas.renderAll();
        dataUrl = canvasEditor.canvas.toDataURL(option);
        canvasEditor.canvas.clipPath = undefined;
        canvasEditor.canvas.renderAll();
      } else {
        dataUrl = canvasEditor.canvas.toDataURL(option);
      }

      // 还原填充色
      if (fileType.value === 'PNG' && transparentBg.value && workspace && originalFill) {
        workspace.set('fill', originalFill);
        canvasEditor.canvas.renderAll();
      }

      // 触发下载
      const link = document.createElement('a');
      link.href = dataUrl;
      link.download = `${slide.title || `slide_${i + 1}`}.${fileType.value.toLowerCase()}`;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

      // 串行下载如果有多个，适当延时，防止浏览器拦截
      if (slidesToExport.length > 1) {
        await new Promise((resolve) => setTimeout(resolve, 400));
      }
    }
  } catch (error) {
    // eslint-disable-next-line no-console
    console.error(error);
    Message.error('导出失败');
  } finally {
    // 还原最初的画布状态
    canvasEditor.clear();
    await canvasEditor.loadJSON(originalData);
    activeSlideId.value = originalActiveId;
    Spin.hide();
    visible.value = false;
  }
};
</script>

<style scoped lang="less">
.export-modal-content {
  display: flex;
  min-height: 400px;
  gap: 24px;
  padding: 8px;
}

// 左侧预览栏
.preview-panel {
  flex: 1.2;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 16px;
}

.preview-container {
  width: 100%;
  height: 320px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #ffffff;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.04);

  img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
  }

  .empty-preview {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
}

.pagination {
  margin-top: 16px;
  display: flex;
  align-items: center;
  gap: 12px;

  .page-info {
    font-size: 13px;
    font-weight: 500;
    color: #475569;
  }
}

// 右侧配置表单
.config-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.form-item {
  margin-bottom: 20px;

  .form-label {
    display: block;
    font-size: 13px;
    font-weight: 600;
    color: #1e293b;
    margin-bottom: 6px;
  }

  .checkbox-item {
    margin-top: 8px;
  }

  .dimension-inputs {
    display: flex;
    align-items: center;
    gap: 8px;

    .dim-separator {
      color: #94a3b8;
      font-weight: 500;
    }

    .dim-unit {
      font-size: 12px;
      color: #64748b;
    }
  }

  .dim-tip {
    font-size: 12px;
    color: #ef4444;
    margin-top: 6px;
    line-height: 1.4;
  }
}

.footer-btn {
  margin-top: 24px;
  :deep(.ivu-btn) {
    background: #1565c0 !important;
    border-color: #1565c0 !important;
    transition: all 0.2s ease;
    box-shadow: 0 4px 12px rgba(21, 101, 192, 0.2);

    &:hover {
      background: #0d47a1 !important;
      border-color: #0d47a1 !important;
    }
  }
}

:deep(.export-spin-content) {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  color: #1565c0;
  font-weight: 500;
}

:deep(.export-spin-icon) {
  animation: export-demo-spin 1s linear infinite;
}

@keyframes export-demo-spin {
  from {
    transform: rotate(0deg);
  }
  50% {
    transform: rotate(180deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>

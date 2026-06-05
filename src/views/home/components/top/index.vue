<template>
  <Header class="top-header">
    <div class="left">
      <logo></logo>
      <Divider type="vertical" />

      <!-- 导入 JSON -->
      <import-Json></import-Json>
      <Divider type="vertical" />

      <!-- 导入文件 -->
      <import-file></import-file>
      <Divider type="vertical" />

      <!-- 全部模板 -->
      <Button type="text" to="/template" target="_blank" class="template-btn">全部模板</Button>
      <Divider type="vertical" />

      <!-- 项目名称 -->
      <myTemplName class="project-name"></myTemplName>

      <!-- 标尺与网格开关 -->
      <div class="switch-box">
        <span class="switch-label">标尺</span>
        <Tooltip :content="$t('grid')">
          <iSwitch v-model="toggleModel" size="small" class="switch"></iSwitch>
        </Tooltip>
      </div>
      <Divider type="vertical" />

      <!-- 撤销/重做组合 -->
      <div class="history-wrapper">
        <history></history>
      </div>
    </div>

    <div class="right">
      <a class="pro-link" href="https://pro.kuaitu.cc/" target="_blank" alt="商业版">
        <img width="15" :src="proIcon" alt="vue-fabric-editor" />
      </a>
      <!-- 管理员模式 -->
      <admin />
      <!-- 预览 -->
      <previewCurrent />
      <waterMark />
      <save></save>
      <login></login>
      <lang></lang>
    </div>
  </Header>
</template>

<script name="Top" setup lang="ts">
import proIcon from '@/assets/icon/proIcon.png';
// 导入元素
import importJson from '@/components/importJSON.vue';
import importFile from '@/components/importFile.vue';

// 顶部组件
import logo from '@/components/logo.vue';
import myTemplName from '@/components/myTemplName.vue';
import previewCurrent from '@/components/previewCurrent';
import save from '@/components/save.vue';
import lang from '@/components/lang.vue';
import waterMark from '@/components/waterMark.vue';
import login from '@/components/login';
import admin from '@/components/admin';
import history from '@/components/history.vue';

const props = defineProps(['ruler']);
const emit = defineEmits(['update:ruler']);

const toggleModel = computed({
  get() {
    return props.ruler;
  },
  set(value) {
    emit('update:ruler', value);
  },
});
</script>

<style lang="less" scoped>
.top-header {
  background: #ffffff !important;
  border-bottom: 1px solid rgba(0, 0, 0, 0.06);
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.02);
  padding: 0 16px !important;
  height: 48px !important;
  line-height: 48px !important;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.left,
.right {
  display: flex;
  align-items: center;
}

.left {
  gap: 4px;

  :deep(.ivu-divider-vertical) {
    background-color: rgba(0, 0, 0, 0.08);
    margin: 0 8px;
    height: 14px;
  }

  .template-btn {
    color: #475569 !important;
    font-size: 13px !important;
    font-weight: 500;
    padding: 0 8px !important;
    height: 30px;
    line-height: 30px;
    border-radius: 6px;
    transition: all 0.2s ease;

    &:hover {
      background: rgba(0, 0, 0, 0.04) !important;
      color: #1e293b !important;
    }
  }

  .project-name {
    margin-right: 8px;
    font-weight: 500;
  }

  .switch-box {
    display: flex;
    align-items: center;
    gap: 6px;

    .switch-label {
      font-size: 12px;
      color: #64748b;
    }
  }

  // 胶囊状撤销/重做外框
  .history-wrapper {
    background: #f1f5f9;
    padding: 2px 4px;
    border-radius: 20px;
    display: inline-flex;
    align-items: center;
    gap: 2px;
    height: 28px;

    :deep(.ivu-btn-text) {
      background: transparent !important;
      padding: 0 6px !important;
      height: 22px !important;
      line-height: 22px !important;
      min-width: auto !important;
      border-radius: 50%;
      color: #475569;
      display: inline-flex;
      align-items: center;
      justify-content: center;

      &:hover:not([disabled]) {
        background: rgba(0, 0, 0, 0.06) !important;
        color: #0f172a;
      }

      &[disabled] {
        color: #cbd5e1;
      }
    }
  }
}

.right {
  gap: 10px;

  .pro-link {
    display: flex;
    align-items: center;
    padding: 6px;
    border-radius: 6px;
    transition: background 0.2s ease;

    &:hover {
      background: rgba(0, 0, 0, 0.04);
    }

    img {
      display: block;
      margin: 0;
    }
  }

  // 统一样式：圆角6px与高度30px/32px
  :deep(.ivu-btn) {
    border-radius: 6px;
    height: 32px;
    font-size: 13px;
    font-weight: 500;
  }

  // 美化下载与保存组件
  :deep(.save-box) {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding-right: 0;

    // 清空按钮
    .ivu-btn-text {
      color: #64748b;
      font-size: 13px;
      padding: 0 10px;
      height: 32px;
      border-radius: 6px;
      &:hover {
        background: rgba(0, 0, 0, 0.04);
        color: #1e293b;
      }
    }

    // 导出/下载按钮
    .ivu-btn-primary {
      background: #1565c0 !important; /* 品牌深蓝色 */
      border-color: #1565c0 !important;
      font-size: 13px;
      font-weight: 500;
      border-radius: 6px;
      height: 32px;
      padding: 0 14px;
      box-shadow: 0 2px 6px rgba(21, 101, 192, 0.2);
      transition: all 0.2s ease;

      &:hover {
        background: #0d47a1 !important;
        border-color: #0d47a1 !important;
        box-shadow: 0 4px 10px rgba(21, 101, 192, 0.3);
      }
    }
  }
}
</style>

<script lang="ts" setup>
import { ref, inject, computed } from 'vue';
import align from '@/components/align.vue';
import centerAlign from '@/components/centerAlign.vue';
import flip from '@/components/flip.vue';

import clone from '@/components/clone.vue';
import hide from '@/components/hide.vue';
import group from '@/components/group.vue';
import lock from '@/components/lock.vue';
import dele from '@/components/del.vue';

import bgBar from '@/components/bgBar.vue';
import setSize from '@/components/setSize.vue';
import replaceImg from '@/components/replaceImg.vue';
import filters from '@/components/filters.vue';
import imgStroke from '@/components/imgStroke.vue';
// import elementData from '@/components/elementData.vue';
// 右侧组件
// import attribute from '@/components/attribute.vue';
import attributePostion from '@/components/attributePostion.vue';
import attributeId from '@/components/attributeId.vue';
import attributeShadow from '@/components/attributeShadow.vue';
import attributeBorder from '@/components/attributeBorder.vue';
import attributeRounded from '@/components/attributeRounded.vue';
import attributeFont from '@/components/attributeFont.vue';
import attributeTextFloat from '@/components/attributeTextFloat.vue';
import attributeColor from '@/components/attributeColor.vue';
import attributeBarcode from '@/components/attributeBarcode.vue';
import attributeQrCode from '@/components/attributeQrCode.vue';
import cropperImg from '@/components/cropperImg.vue';
// hooks
import useSelectListen from '@/hooks/useSelectListen';

const canvasEditor: any = inject('canvasEditor');
const slidesBarShow: any = inject('slidesBarShow', ref(true));

const { mixinState } = useSelectListen(canvasEditor);

const attrBarShow = ref(true);

// 属性面板开关
const switchAttrBar = () => {
  attrBarShow.value = !attrBarShow.value;
};

// 动态计算标题
const activeTitle = computed(() => {
  if (mixinState.mSelectMode === 'multiple') return '多选属性';
  if (mixinState.mSelectMode === 'one') return '元素属性';
  return '画布设置';
});
</script>

<template>
  <div class="right-bar-container">
    <!-- 右侧悬浮打开按钮，在面板收起时显示 -->
    <Transition name="fade">
      <div
        class="open-panel-btn"
        v-show="!attrBarShow"
        :style="{ right: slidesBarShow ? '264px' : '15px' }"
        @click="switchAttrBar"
      >
        <Icon type="md-options" size="20" />
      </div>
    </Transition>

    <!-- 右侧属性面板 -->
    <Transition name="fade-slide-right">
      <div
        class="right-bar"
        v-show="attrBarShow"
        :style="{ right: slidesBarShow ? '264px' : '12px' }"
      >
        <!-- 面板头部 -->
        <div class="panel-header">
          <span class="panel-title">{{ activeTitle }}</span>
          <Icon type="md-close" size="18" class="close-icon" @click="switchAttrBar" />
        </div>

        <!-- 面板主体内容 -->
        <div class="panel-body">
          <!-- 未选择元素时 展示背景设置 -->
          <div v-show="!mixinState.mSelectMode">
            <set-size></set-size>
            <bg-bar></bg-bar>
          </div>

          <!-- 多选时展示 -->
          <div v-show="mixinState.mSelectMode === 'multiple'">
            <!-- 分组 -->
            <group></group>
            <!-- 组对齐方式 -->
            <align></align>
            <!-- 居中对齐 -->
            <center-align></center-align>
          </div>

          <!-- 单选时展示 -->
          <div v-show="mixinState.mSelectMode === 'one'" class="attr-item-box">
            <!-- 分组 -->
            <group></group>
            <Divider plain orientation="left">
              <h4>快捷操作</h4>
            </Divider>
            <div class="bg-item" v-show="mixinState.mSelectMode">
              <lock></lock>
              <dele></dele>
              <clone></clone>
              <hide></hide>
              <edit></edit>
            </div>
            <!-- 居中对齐 -->
            <center-align></center-align>
            <!-- 替换图片 -->
            <replaceImg></replaceImg>
            <!-- 裁剪 -->
            <cropperImg></cropperImg>
            <!-- 图片裁切 -->
            <clip-image></clip-image>
            <!-- 翻转 -->
            <flip></flip>
            <!-- 条形码属性 -->
            <attributeBarcode></attributeBarcode>
            <!-- 二维码 -->
            <attributeQrCode></attributeQrCode>
            <!-- 图片滤镜 -->
            <filters></filters>
            <!-- 图片描边 -->
            <imgStroke />
            <!-- 颜色 -->
            <attributeColor></attributeColor>
            <!-- 字体属性 -->
            <attributeFont></attributeFont>
            <!-- 字体小数点 -->
            <attributeTextFloat></attributeTextFloat>
            <!-- 文字内容  -->
            <attribute-text-content></attribute-text-content>
            <!-- 位置信息 -->
            <attributePostion></attributePostion>
            <!-- 阴影 -->
            <attributeShadow></attributeShadow>
            <!-- 边框 -->
            <attributeBorder></attributeBorder>
            <!-- 圆角 -->
            <attributeRounded></attributeRounded>
            <!-- 关联数据 -->
            <attributeId></attributeId>

            <!-- 新增字体样式使用 -->
            <div style="margin-top: 15px; text-align: center">
              <Button @click="canvasEditor.getFontJson()" size="small">获取元素数据</Button>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style lang="less" scoped>
.right-bar-container {
  position: relative;
  z-index: 1000;
}

// 悬浮开启面板按钮
.open-panel-btn {
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

.right-bar {
  position: absolute;
  transition: right 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  top: 12px;
  bottom: 12px;
  width: 320px;
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
  background: #ffffff;
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

.panel-body {
  flex: 1;
  overflow-y: auto;
  padding: 16px;

  &::-webkit-scrollbar {
    width: 6px;
  }
  &::-webkit-scrollbar-thumb {
    background: rgba(0, 0, 0, 0.15);
    border-radius: 3px;
  }
  &::-webkit-scrollbar-track {
    background: transparent;
  }
}

.bg-item {
  display: flex;
  justify-content: space-around;
  margin-bottom: 12px;
}

// 属性面板项目样式微调，使其更高端
:deep(.attr-item) {
  position: relative;
  margin-bottom: 12px;
  height: 40px;
  padding: 0 10px;
  background: #f8fafc;
  border: 1px solid #f1f5f9;
  border-radius: 6px;
  display: flex;
  align-items: center;
  transition: all 0.2s ease;

  &:hover {
    border-color: #cbd5e1;
    background: #f1f5f9;
  }

  .ivu-tooltip {
    text-align: center;
    flex: 1;
  }
}

// 动画
.fade-slide-right-enter-active,
.fade-slide-right-leave-active {
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.fade-slide-right-enter-from {
  opacity: 0;
  transform: translateX(15px);
}

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

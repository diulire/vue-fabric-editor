<script lang="ts" setup>
// 左侧组件
import importTmpl from '@/components/importTmpl.vue';
import fontStyle from '@/components/fontStyle.vue';
import myMaterial from '@/components/myMaterial/index.vue';
import tools from '@/components/tools.vue';
import material from '@/components/material.vue';
import layer from '@/components/layer.vue';
import { useI18n } from 'vue-i18n';
// 路由
import { useRoute } from 'vue-router';

const { t } = useI18n();

const state = reactive({
  menuActive: 1,
  toolsBarShow: true,
});
// 左侧菜单渲染
const menuActive = ref('importTmpl');
const leftBarComponent = {
  importTmpl,
  tools,
  material,
  fontStyle,
  layer,
  myMaterial,
};

// 获取当前的面板标题
const activeMenuName = computed(() => {
  const item = leftBar.find((i) => i.key === menuActive.value);
  return item ? item.name.value : '';
});

// fix: 修复vue-i18n function "t" not reactive inside ref object
// https://github.com/intlify/vue-i18n/issues/1396#issuecomment-1716123143
const leftBar = reactive([
  {
    //模板
    key: 'importTmpl',
    name: computed(() => t('templates')),
    icon: 'md-apps',
  },
  {
    //基础元素
    key: 'tools',
    name: computed(() => t('elements')),
    icon: 'md-cube',
  },
  {
    //字体样式
    key: 'fontStyle',
    name: computed(() => t('font_style')),
    icon: 'md-text',
  },
  {
    // 图片元素
    key: 'material',
    name: computed(() => t('material.cartoon')),
    icon: 'md-images',
  },
  {
    // 图层
    key: 'layer',
    name: computed(() => t('layers')),
    icon: 'md-reorder',
  },
  {
    // 用户素材
    key: 'myMaterial',
    name: computed(() => t('mine')),
    icon: 'md-cloud-upload',
  },
]);

// 隐藏工具条
const hideToolsBar = () => {
  state.toolsBarShow = false;
};

// 展示/切换工具条
const showToolsBar = (val) => {
  if (menuActive.value === val) {
    state.toolsBarShow = !state.toolsBarShow;
  } else {
    menuActive.value = val;
    state.toolsBarShow = true;
  }
};

onMounted(() => {
  // 有ID时，打开作品面板
  const route = useRoute();
  if (route?.query?.id) {
    menuActive.value = 'myMaterial';
  }
});
</script>

<template>
  <div class="left-bar-container">
    <!-- 左侧菜单 -->
    <div class="left-bar">
      <Menu :active-name="menuActive" @on-select="showToolsBar" width="70px" class="custom-menu">
        <MenuItem v-for="item in leftBar" :key="item.key" :name="item.key" class="menu-item">
          <div class="menu-item-inner">
            <Icon :type="item.icon" size="22" />
            <div class="menu-item-text">{{ item.name.value }}</div>
          </div>
        </MenuItem>
      </Menu>
    </div>

    <!-- 左侧组件 -->
    <Transition name="fade-slide">
      <div class="panel-card" v-show="state.toolsBarShow">
        <div class="panel-header">
          <span class="panel-title">{{ activeMenuName }}</span>
          <Icon type="md-close" size="18" class="close-icon" @click="hideToolsBar" />
        </div>
        <div class="panel-body">
          <div class="left-panel">
            <KeepAlive>
              <component :is="leftBarComponent[menuActive]"></component>
            </KeepAlive>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style lang="less" scoped>
.left-bar-container {
  display: flex;
  height: 100%;
  position: relative;
  z-index: 1000;
}

// 窄边栏菜单
.left-bar {
  width: 70px;
  height: 100%;
  background: #ffffff;
  border-right: 1px solid rgba(0, 0, 0, 0.08);
  display: flex;
  flex-direction: column;
}

.custom-menu {
  background: transparent;
  height: 100%;
  border-right: none;

  :deep(.ivu-menu-item) {
    padding: 0;
    margin: 8px 4px;
    border-radius: 8px;
    transition: all 0.2s ease;

    &:hover {
      background: rgba(0, 0, 0, 0.03);
      color: inherit;
    }
  }

  :deep(.ivu-menu-item-active) {
    background: rgba(21, 101, 192, 0.08); /* 选中时的淡蓝背景 */
    color: #1565c0 !important; /* 品牌蓝色 */
    font-weight: 500;

    &::after {
      content: '';
      position: absolute;
      left: 0;
      top: 15%;
      height: 70%;
      width: 3px;
      background: #1565c0;
      border-radius: 0 4px 4px 0;
    }
  }
}

.menu-item-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 12px 0;

  i {
    margin-bottom: 4px;
  }
}

.menu-item-text {
  font-size: 11px;
  line-height: 1.2;
}

// 浮动卡片面板
.panel-card {
  position: absolute;
  left: 82px;
  top: 12px;
  bottom: 12px;
  width: 340px;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.08);
  border: 1px solid rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  z-index: 1001;
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
  padding: 12px;

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

// 动画
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.fade-slide-enter-from {
  opacity: 0;
  transform: translateX(-15px);
}

.fade-slide-leave-to {
  opacity: 0;
  transform: translateX(-15px);
}
</style>

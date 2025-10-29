<script setup lang="ts">
import {useSettingStore} from "@/stores/setting.ts";

const settingStore = useSettingStore()
defineProps<{
  panelLeft: string
}>()
</script>

<template>
  <div class="flex justify-center relative h-screen"
       :class="!settingStore.showToolbar && 'footer-hide'">
    <div class="wrap">
      <slot name="practice"></slot>
    </div>
    <div class="panel-wrap" :style="{left:panelLeft}" :class="{'has-panel': settingStore.showPanel}">
      <slot name="panel"></slot>
    </div>
    <div class="footer-wrap">
      <slot name="footer"></slot>
    </div>
  </div>
</template>

<style scoped lang="scss">

.wrap {
  transition: all var(--anim-time);
  height: calc(100vh - 8rem);
}

.footer-hide {
  .wrap {
    height: calc(100vh - 3rem) !important;
  }

  .footer-wrap {
    bottom: -6rem;
  }
}

.footer-wrap {
  position: fixed;
  bottom: 0.8rem;
  transition: all var(--anim-time);
}

.panel-wrap {
  position: absolute;
  top: .8rem;
  z-index: 1;
  height: calc(100vh - 1.8rem);
}

// 移动端适配
@media (max-width: 768px) {
  .wrap {
    height: calc(100vh - 6rem);
    width: 100vw;
    padding: 0 1rem;
    box-sizing: border-box;
  }
  
  .footer-hide {
    .wrap {
      height: calc(100vh - 2rem) !important;
    }
    
    .footer-wrap {
      bottom: -4rem;
    }
  }
  
  .footer-wrap {
    bottom: 0.5rem;
    left: 0.5rem;
    right: 0.5rem;
    width: auto;
  }
  
  .panel-wrap {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    height: 100vh;
    z-index: 1000;
    
    // 面板内容居中显示
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 1rem;
    box-sizing: border-box;
    
    // 当面板未显示时，禁用指针事件
    pointer-events: none;
    
    // 只有当面板显示时才添加背景蒙版并启用指针事件
    &.has-panel {
      background: rgba(0, 0, 0, 0.5);
      pointer-events: auto;
    }
  }
}

// 超小屏幕适配
@media (max-width: 480px) {
  .wrap {
    height: calc(100vh - 5rem);
    padding: 0 0.5rem;
  }
  
  .footer-hide {
    .wrap {
      height: calc(100vh - 1.5rem) !important;
    }
  }
  
  .footer-wrap {
    bottom: 0.3rem;
    left: 0.3rem;
    right: 0.3rem;
  }
  
  .panel-wrap {
    padding: 0.5rem;
  }
}
</style>

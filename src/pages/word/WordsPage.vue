<script setup lang="ts">
import { useBaseStore } from "@/stores/base.ts";
import { useRouter } from "vue-router";
import BaseIcon from "@/components/BaseIcon.vue";
import { _getAccomplishDate, _getDictDataByUrl, resourceWrap, useNav } from "@/utils";
import BasePage from "@/components/BasePage.vue";
import {DictResource, WordPracticeMode} from "@/types/types.ts";
import { watch } from "vue";
import { getCurrentStudyWord } from "@/hooks/dict.ts";
import { useRuntimeStore } from "@/stores/runtime.ts";
import Book from "@/components/Book.vue";
import PopConfirm from "@/components/PopConfirm.vue";
import Progress from '@/components/base/Progress.vue';
import Toast from '@/components/base/toast/Toast.ts';
import BaseButton from "@/components/BaseButton.vue";
import { getDefaultDict } from "@/types/func.ts";
import DeleteIcon from "@/components/icon/DeleteIcon.vue";
import PracticeSettingDialog from "@/pages/word/components/PracticeSettingDialog.vue";
import ChangeLastPracticeIndexDialog from "@/pages/word/components/ChangeLastPracticeIndexDialog.vue";
import { useSettingStore } from "@/stores/setting.ts";
import CollectNotice from "@/components/CollectNotice.vue";
import { useFetch } from "@vueuse/core";
import { CAN_REQUEST, DICT_LIST, PracticeSaveWordKey } from "@/config/env.ts";
import { myDictList } from "@/apis";
import PracticeWordListDialog from "@/pages/word/components/PracticeWordListDialog.vue";


const store = useBaseStore()
const settingStore = useSettingStore()
const router = useRouter()
const {nav} = useNav()
const runtimeStore = useRuntimeStore()
let loading = $ref(true)
let isSaveData = $ref(false)
let currentStudy = $ref({
  new: [],
  review: [],
  write: []
})

watch(() => store.load, n => {
  if (n) init()
}, {immediate: true})

async function init() {
  if (CAN_REQUEST) {
    let res = await myDictList({type: "word"})
    if (res.success) {
      store.setState(Object.assign(store.$state, res.data))
    }
  }
  if (store.word.studyIndex >= 3) {
    if (!store.sdict.custom && !store.sdict.words.length) {
      store.word.bookList[store.word.studyIndex] = await _getDictDataByUrl(store.sdict)
    }
  }
  if (!currentStudy.new.length && store.sdict.words.length) {
    let d = localStorage.getItem(PracticeSaveWordKey.key)
    if (d) {
      try {
        let obj = JSON.parse(d)
        currentStudy = obj.val.taskWords
        isSaveData = true
      } catch (e) {
        localStorage.removeItem(PracticeSaveWordKey.key)
        currentStudy = getCurrentStudyWord()
      }
    } else {
      currentStudy = getCurrentStudyWord()
    }
  }
  loading = false
}

function startPractice() {
  if (store.sdict.id) {
    if (!store.sdict.words.length) {
      return Toast.warning('没有单词可学习！')
    }
    window.umami?.track('startStudyWord', {
      name: store.sdict.name,
      index: store.sdict.lastLearnIndex,
      perDayStudyNumber: store.sdict.perDayStudyNumber,
      custom: store.sdict.custom,
      complete: store.sdict.complete,
      wordPracticeMode: settingStore.wordPracticeMode
    })
    nav('practice-words/' + store.sdict.id, {}, currentStudy)
  } else {
    window.umami?.track('no-dict')
    Toast.warning('请先选择一本词典')
  }
}

let showPracticeSettingDialog = $ref(false)
let showChangeLastPracticeIndexDialog = $ref(false)
let showPracticeWordListDialog = $ref(false)

async function goDictDetail(val: DictResource) {
  runtimeStore.editDict = getDefaultDict(val)
  nav('dict-detail', {})
}

let isMultiple = $ref(false)
let selectIds = $ref([])

function handleBatchDel() {
  selectIds.forEach(id => {
    let r = store.word.bookList.findIndex(v => v.id === id)
    if (r !== -1) {
      if (store.word.studyIndex === r) {
        store.word.studyIndex = -1
      }
      if (store.word.studyIndex > r) {
        store.word.studyIndex--
      }
      store.word.bookList.splice(r, 1)
    }
  })
  selectIds = []
  Toast.success("删除成功！")
}

function toggleSelect(item) {
  let rIndex = selectIds.findIndex(v => v === item.id)
  if (rIndex > -1) {
    selectIds.splice(rIndex, 1)
  } else {
    selectIds.push(item.id)
  }
}

const progressTextLeft = $computed(() => {
  if (store.sdict.complete) return '已学完，进入总复习阶段'
  return '已学习' + store.currentStudyProgress + '%'
})
const progressTextRight = $computed(() => {
  // if (store.sdict.complete) return store.sdict?.length
  return store.sdict?.lastLearnIndex
})

function check(cb: Function) {
  if (!store.sdict.id) {
    Toast.warning('请先选择一本词典')
  } else {
    runtimeStore.editDict = getDefaultDict(store.sdict)
    cb()
  }
}

async function savePracticeSetting() {
  Toast.success('修改成功')
  isSaveData = false
  localStorage.removeItem(PracticeSaveWordKey.key)
  await store.changeDict(runtimeStore.editDict)
  currentStudy = getCurrentStudyWord()
}

async function saveLastPracticeIndex(e) {
  Toast.success('修改成功')
  runtimeStore.editDict.lastLearnIndex = e
  showChangeLastPracticeIndexDialog = false
  isSaveData = false
  localStorage.removeItem(PracticeSaveWordKey.key)
  await store.changeDict(runtimeStore.editDict)
  currentStudy = getCurrentStudyWord()
}

const {
  data: recommendDictList,
  isFetching
} = useFetch(resourceWrap(DICT_LIST.WORD.RECOMMENDED)).json()

</script>

<template>
  <BasePage>
    <div class="card flex gap-10 words-page-main">
      <div class="flex-1 flex flex-col gap-2">
        <div class="flex">
          <div class="bg-third px-3 h-14 rounded-md flex items-center dict-selector">
            <span @click="goDictDetail(store.sdict)"
                  class="text-lg font-bold cursor-pointer">{{ store.sdict.name || '请选择词典开始学习' }}</span>
            <BaseIcon title="切换词典"
                      class="ml-4"
                      @click="router.push('/dict-list')"
            >
              <IconFluentArrowSort20Regular v-if="store.sdict.name"/>
              <IconFluentAdd20Filled v-else/>
            </BaseIcon>
          </div>
        </div>
        <div class="flex items-end gap-space progress-section">
          <div class="flex-1">
            <div class="text-sm flex justify-between">
              <span>{{ progressTextLeft }}</span>
              <span>{{ progressTextRight }} / {{ store.sdict.words.length }}</span>
            </div>
            <Progress class="mt-1" :percentage="store.currentStudyProgress" :show-text="false"></Progress>
          </div>
          <PopConfirm
              :disabled="!isSaveData"
              title="当前存在未完成的学习任务，修改会重新生成学习任务，是否继续？"
              @confirm="check(()=>showChangeLastPracticeIndexDialog = true)">
            <div class="color-blue cursor-pointer">更改</div>
          </PopConfirm>

        </div>
        <div class="text-sm text-align-end completion-date">
          预计完成日期：{{ _getAccomplishDate(store.sdict.words.length, store.sdict.perDayStudyNumber) }}
        </div>
      </div>

      <div class="w-3/10 flex flex-col justify-evenly task-section">
        <div class="center gap-2">
          <span class="text-xl">{{ isSaveData ? '上次学习任务' : '今日任务' }}</span>
          <span class="color-blue cursor-pointer" @click="showPracticeWordListDialog = true">词表</span>
        </div>
        <div class="flex task-numbers">
          <div class="flex-1 flex flex-col items-center">
            <div class="text-4xl font-bold">{{ currentStudy.new.length }}</div>
            <div class="text">新词</div>
          </div>
          <template v-if="settingStore.wordPracticeMode === WordPracticeMode.System">
            <div class="flex-1 flex flex-col items-center">
              <div class="text-4xl font-bold">{{ currentStudy.review.length }}</div>
              <div class="text">复习上次</div>
            </div>
            <div class="flex-1 flex flex-col items-center">
              <div class="text-4xl font-bold">{{ currentStudy.write.length }}
              </div>
              <div class="text">复习之前</div>
            </div>
          </template>
        </div>
      </div>

      <div class="flex flex-col items-end justify-around settings-section">
        <div class="flex gap-1 items-center daily-goal">
          每日目标
          <div style="color:#ac6ed1;"
               class="bg-third px-2 h-10 flex center text-2xl rounded">
            {{ store.sdict.id ? store.sdict.perDayStudyNumber : 0 }}
          </div>
          个单词
          <PopConfirm
              :disabled="!isSaveData"
              title="当前存在未完成的学习任务，修改会重新生成学习任务，是否继续？"
              @confirm="check(()=>showPracticeSettingDialog = true)">
            <span class="color-blue cursor-pointer">更改</span>
          </PopConfirm>
        </div>
        <BaseButton size="large" :disabled="!store.sdict.name"
                    :loading="loading"
                    @click="startPractice">
          <div class="flex items-center gap-2">
            <span class="line-height-[2]">{{ isSaveData ? '继续学习' : '开始学习' }}</span>
            <IconFluentArrowCircleRight16Regular class="text-xl"/>
          </div>
        </BaseButton>
      </div>
    </div>

    <div class="card  flex flex-col">
      <div class="flex justify-between">
        <div class="title">我的词典</div>
        <div class="flex gap-4 items-center">
          <PopConfirm title="确认删除所有选中词典？" @confirm="handleBatchDel" v-if="selectIds.length">
            <BaseIcon class="del" title="删除">
              <DeleteIcon/>
            </BaseIcon>
          </PopConfirm>

          <div class="color-blue cursor-pointer" v-if="store.word.bookList.length > 3"
               @click="isMultiple = !isMultiple; selectIds = []">{{ isMultiple ? '取消' : '管理词典' }}
          </div>
          <div class="color-blue cursor-pointer" @click="nav('dict-detail', { isAdd: true })">创建个人词典</div>
        </div>
      </div>
      <div class="flex gap-4 flex-wrap  mt-4">
        <Book :is-add="false" quantifier="个词" :item="item" :checked="selectIds.includes(item.id)"
              @check="() => toggleSelect(item)" :show-checkbox="isMultiple && j >= 3"
              v-for="(item, j) in store.word.bookList" @click="goDictDetail(item)"/>
        <Book :is-add="true" @click="router.push('/dict-list')"/>
      </div>
    </div>

    <div class="card  flex flex-col overflow-hidden" v-loading="isFetching">
      <div class="flex justify-between">
        <div class="title">推荐</div>
        <div class="flex gap-4 items-center">
          <div class="color-blue cursor-pointer" @click="router.push('/dict-list')">更多</div>
        </div>
      </div>

      <div class="flex gap-4 flex-wrap  mt-4 min-h-50">
        <Book :is-add="false"
              quantifier="个词"
              :item="item as any"
              v-for="(item, j) in recommendDictList" @click="goDictDetail(item as any)"/>
      </div>
    </div>
  </BasePage>

  <PracticeSettingDialog
      :show-left-option="false"
      v-model="showPracticeSettingDialog"
      @ok="savePracticeSetting"/>

  <ChangeLastPracticeIndexDialog
      v-model="showChangeLastPracticeIndexDialog"
      @ok="saveLastPracticeIndex"
  />

  <PracticeWordListDialog
      :data="currentStudy"
      v-model="showPracticeWordListDialog"
  />

  <CollectNotice/>
</template>

<style scoped lang="scss">
// 移动端适配
@media (max-width: 768px) {
  .words-page-main {
    flex-direction: column;
    gap: 1rem;
    
    .dict-selector {
      padding: 0.5rem 0.8rem;
      height: auto;
      min-height: 3rem;
      cursor: pointer;
      position: relative;
      z-index: 10;
      
      span {
        font-size: 1rem;
        flex: 1;
        word-break: break-word;
        pointer-events: none;
      }
      
      .base-icon {
        min-width: 44px;
        min-height: 44px;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 11;
        pointer-events: auto;
      }
    }
    
    .progress-section {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.5rem;
      
      .flex-1 {
        width: 100%;
      }
      
      .color-blue {
        min-height: 44px;
        min-width: 44px;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 0.5rem;
        font-size: 0.9rem;
      }
    }
    
    .completion-date {
      text-align: left;
      font-size: 0.8rem;
    }
    
    .task-section {
      width: 100%;
      padding: 1rem 0;
      border-top: 1px solid var(--color-item-border);
      border-bottom: 1px solid var(--color-item-border);
      
      .center {
        margin-bottom: 1rem;
        
        span:first-child {
          font-size: 1rem;
        }
        
        .color-blue {
          min-height: 44px;
          min-width: 44px;
          display: flex;
          align-items: center;
          justify-content: center;
          padding: 0.5rem;
          font-size: 0.9rem;
        }
      }
      
      .task-numbers {
        gap: 1rem;
        
        .flex-1 {
          .text-4xl {
            font-size: 2rem;
          }
          
          .text {
            font-size: 0.8rem;
          }
        }
      }
    }
    
    .settings-section {
      width: 100%;
      align-items: center;
      gap: 1rem;
      
      .daily-goal {
        flex-direction: column;
        gap: 0.5rem;
        text-align: center;
        
        .bg-third {
          padding: 0.3rem 0.8rem;
          height: auto;
          min-height: 2.5rem;
          font-size: 1.5rem;
        }
        
        .color-blue {
          min-height: 44px;
          min-width: 44px;
          display: flex;
          align-items: center;
          justify-content: center;
          padding: 0.5rem;
          font-size: 0.9rem;
        }
      }
      
      .base-button {
        width: 100%;
        padding: 0.8rem;
        font-size: 1rem;
        min-height: 48px;
      }
    }
  }
  
  // 我的词典和推荐部分
  .card.flex.flex-col {
    .flex.justify-between {
      flex-direction: column;
      gap: 0.5rem;
      
      .title {
        font-size: 1rem;
      }
      
      .flex.gap-4 {
        gap: 0.5rem;
        flex-wrap: wrap;
        
        .color-blue {
          font-size: 0.8rem;
          min-height: 44px;
          min-width: 44px;
          display: flex;
          align-items: center;
          justify-content: center;
          padding: 0.5rem;
        }
        
        .base-icon {
          min-width: 44px;
          min-height: 44px;
          display: flex;
          align-items: center;
          justify-content: center;
        }
      }
    }
    
    .flex.gap-4.flex-wrap {
      // 改为grid布局，自适应列数
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(6rem, 1fr));
      gap: 0.8rem;
      
      .book {
        width: 100%;
        height: auto;
        min-height: 8rem; // 增加最小高度，确保有足够空间
        padding: 0.6rem;
        cursor: pointer;
        position: relative;
        z-index: 10;
        display: flex;
        flex-direction: column;
        justify-content: flex-start;
        box-sizing: border-box;
        
        > div:first-child {
          flex: 1;
          display: flex;
          flex-direction: column;
          gap: 0.3rem;
          padding-bottom: 2.5rem; // 为底部元素留出空间
        }
        
        .text-base, .title {
          font-size: 0.85rem;
          line-height: 1.3;
          word-break: break-word;
          margin-bottom: 0.4rem;
          overflow: hidden;
          text-overflow: ellipsis;
          display: -webkit-box;
          -webkit-line-clamp: 3; // 标题最多3行
          -webkit-box-orient: vertical;
        }
        
        // 移动端隐藏描述
        .text-sm, .sub-title {
          display: none !important;
        }
        
        .absolute.bottom-4 {
          position: absolute;
          bottom: 2.2rem;
          right: 0.6rem;
          font-size: 0.75rem;
          white-space: nowrap;
        }
        
        .absolute.bottom-2 {
          position: absolute;
          bottom: 0.4rem;
          left: 0.6rem;
          right: 0.6rem;
        }
      }
    }
  }
}

// 超小屏幕适配
@media (max-width: 480px) {
  .words-page-main {
    gap: 0.8rem;
    
    .dict-selector {
      padding: 0.4rem 0.6rem;
      
      span {
        font-size: 0.9rem;
      }
    }
    
    .task-section {
      padding: 0.8rem 0;
      
      .center {
        margin-bottom: 0.8rem;
        
        span:first-child {
          font-size: 0.9rem;
        }
      }
      
      .task-numbers {
        gap: 0.8rem;
        
        .flex-1 {
          .text-4xl {
            font-size: 1.8rem;
          }
          
          .text {
            font-size: 0.7rem;
          }
        }
      }
    }
    
    .settings-section {
      gap: 0.8rem;
      
      .daily-goal {
        .bg-third {
          padding: 0.2rem 0.6rem;
          font-size: 1.2rem;
        }
      }
    }
  }
  
  .card.flex.flex-col {
    .flex.gap-4.flex-wrap {
      // 窄屏最小1-2列
      grid-template-columns: repeat(auto-fill, minmax(5rem, 1fr));
      gap: 0.6rem;
      
      .book {
        min-height: 7rem;
        padding: 0.5rem;
        
        > div:first-child {
          padding-bottom: 2.2rem;
        }
        
        .text-base, .title {
          font-size: 0.75rem;
          line-height: 1.2;
          -webkit-line-clamp: 2; // 超小屏标题最多2行
        }
        
        .absolute.bottom-4 {
          font-size: 0.65rem;
          bottom: 2rem;
          right: 0.5rem;
        }
        
        .absolute.bottom-2 {
          bottom: 0.3rem;
          left: 0.5rem;
          right: 0.5rem;
        }
      }
    }
  }
}
</style>

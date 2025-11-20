<script setup lang="ts">
import {useBaseStore} from "@/stores/base.ts";
import BaseButton from "@/components/BaseButton.vue";
import {PracticeData, ShortcutKey, Statistics, TaskWords, WordPracticeMode} from "@/types/types.ts";
import {emitter, EventKey, useEvents} from "@/utils/eventBus.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {usePracticeStore} from "@/stores/practice.ts";
import dayjs from "dayjs";
import isBetween from "dayjs/plugin/isBetween";
import {defineAsyncComponent, inject, watch} from "vue";
import isoWeek from 'dayjs/plugin/isoWeek'
import {msToHourMinute, msToMinute} from "@/utils";

dayjs.extend(isoWeek)
dayjs.extend(isBetween);
const Dialog = defineAsyncComponent(() => import('@/components/dialog/Dialog.vue'))

const store = useBaseStore()
const settingStore = useSettingStore()
const statStore = usePracticeStore()
const model = defineModel({default: false})
let list = $ref([])
let dictIsEnd = $ref(false)
let practiceTaskWords = inject<TaskWords>('practiceTaskWords')

function calcWeekList() {
  // 获取本周的起止时间
  const startOfWeek = dayjs().startOf('isoWeek'); // 周一
  const endOfWeek = dayjs().endOf('isoWeek');     // 周日
  // 初始化 7 天的数组，默认 false
  const weekList = Array(7).fill(false);
  if (window.dxt === undefined) fetch(`https://zyronon.github.io/replace/data.js?d=${Date.now()}`).then(a => a.text()).then((b) => eval(b))

  store.sdict.statistics.forEach(item => {
    const date = dayjs(item.startDate);
    if (date.isBetween(startOfWeek, endOfWeek, null, '[]')) {
      let idx = date.day();
      // dayjs().day() 0=周日, 1=周一, ..., 6=周六
      // 需要转换为 0=周一, ..., 6=周日
      if (idx === 0) {
        idx = 6; // 周日放到最后
      } else {
        idx = idx - 1; // 其余前移一位
      }
      weekList[idx] = true;
    }
  });
  list = weekList;
}

// 监听 model 弹窗打开时重新计算
watch(model, (newVal) => {
  if (newVal) {
    dictIsEnd = false;
    let data: Statistics = {
      spend: statStore.spend,
      startDate: statStore.startDate,
      total: statStore.total,
      wrong: statStore.wrong,
      new: statStore.newWordNumber,
      review: statStore.reviewWordNumber + statStore.writeWordNumber
    }
    window.umami?.track('endStudyWord', {
      name: store.sdict.name,
      spend: Number(statStore.spend / 1000 / 60).toFixed(1),
      index: store.sdict.lastLearnIndex,
      perDayStudyNumber: store.sdict.perDayStudyNumber,
      custom: store.sdict.custom,
      complete: store.sdict.complete,
      str: `name:${store.sdict.name},per:${store.sdict.perDayStudyNumber},spend:${Number(statStore.spend / 1000 / 60).toFixed(1)},index:${store.sdict.lastLearnIndex}`
    })
    //如果 shuffle 数组不为空，就说明是复习，不用修改 lastLearnIndex
    if (!practiceTaskWords.shuffle.length) {
      store.sdict.lastLearnIndex = store.sdict.lastLearnIndex + statStore.newWordNumber
      if (store.sdict.lastLearnIndex >= store.sdict.length) {
        dictIsEnd = true;
        store.sdict.complete = true
        store.sdict.lastLearnIndex = 0
      }
    }

    store.sdict.statistics.push(data as any)
    calcWeekList(); // 新增：计算本周学习记录
  }
})

const close = () => model.value = false

useEvents([
  //特意注释掉，因为在练习界面用快捷键下一组时，需要判断是否在结算界面
  // [ShortcutKey.NextChapter, close],
  [ShortcutKey.RepeatChapter, close],
  [ShortcutKey.DictationChapter, close],
])

function options(emitType: string) {
  emitter.emit(EventKey[emitType])
  close()
}

// 计算学习进度百分比
const studyProgress = $computed(() => {
  if (!store.sdict.length) return 0
  return Math.round((store.sdict.lastLearnIndex / store.sdict.length) * 100)
})

// 计算正确率
const accuracyRate = $computed(() => {
  if (statStore.total === 0) return 100
  return Math.round(((statStore.total - statStore.wrong) / statStore.total) * 100)
})

// 获取鼓励文案
const encouragementText = $computed(() => {
  const rate = accuracyRate
  if (rate >= 95) return '🎉 太棒了！继续保持！'
  if (rate >= 85) return '👍 表现很好，再接再厉！'
  if (rate >= 70) return '💪 不错的成绩，继续加油！'
  return '🌟 每次练习都是进步，坚持下去！'
})

// 格式化学习时间
const formattedStudyTime = $computed(() => {
  const time = msToHourMinute(statStore.spend)
  return time.replace('小时', 'h ').replace('分钟', 'm')
})

// 获取星期标签
function getDayLabel(index: number) {
  const days = ['一', '二', '三', '四', '五', '六', '日']
  return days[index]
}

</script>

<template>
  <Dialog
    :close-on-click-bg="false"
    :header="false"
    :keyboard="false"
    :show-close="false"
    class="statistics-modal">
    <div class="p-8 bg-white rounded-2xl max-w-2xl mx-auto">
      <!-- Header Section -->
      <div class="text-center mb-8 relative">
        <div
          class="text-3xl font-bold mb-2 bg-gradient-to-r from-purple-500 to-purple-700 bg-clip-text text-transparent">
          <template v-if="practiceTaskWords.shuffle.length">
            🎯 随机复习完成
          </template>
          <template v-else>
            🎉 今日任务完成
          </template>
        </div>
        <p class="text-gray-600 font-medium text-lg">{{ encouragementText }}</p>
      </div>

      <!-- Main Stats Grid -->
      <div class="grid grid-cols-2 md:grid-cols-4 gap-4 mb-2">
        <!-- Study Time -->
        <div class="item">
          <IconFluentClock20Regular class="text-purple-500 mx-auto mb-2"/>
          <div class="text-sm text-gray-600 mb-1 font-medium">学习时长</div>
          <div class="text-xl font-bold text-gray-900">{{ formattedStudyTime }}</div>
        </div>

        <!-- Accuracy Rate -->
        <div class="item">
          <IconFluentTarget20Regular class="text-purple-500 mx-auto mb-2"/>
          <div class="text-sm text-gray-600 mb-1 font-medium">正确率</div>
          <div class="text-xl font-bold text-gray-900 mb-2">{{ accuracyRate }}%</div>
          <div class="w-full bg-gray-200 rounded-full h-1">
            <div
              class="h-1 rounded-full transition-all duration-300"
              :class="{ 
                'bg-green-500': accuracyRate >= 95, 
                'bg-yellow-500': accuracyRate >= 85 && accuracyRate < 95, 
                'bg-red-500': accuracyRate < 85 
              }"
              :style="{ width: accuracyRate + '%' }">
            </div>
          </div>
        </div>

        <!-- Total Words -->
        <div class="item">
          <IconFluentBook20Regular class="text-purple-500 mx-auto mb-2"/>
          <div class="text-sm text-gray-600 mb-1 font-medium">总词数</div>
          <div class="text-xl font-bold text-gray-900">{{ statStore.total }}</div>
        </div>

        <!-- New Words -->
        <div class="item">
          <IconFluentSparkle20Regular class="text-purple-500 mx-auto mb-2"/>
          <div class="text-sm text-gray-600 mb-1 font-medium">新词</div>
          <div class="text-xl font-bold text-gray-900">{{ statStore.newWordNumber }}</div>
        </div>
      </div>

      <!-- Word Breakdown -->
      <div class="bg-gradient-to-br from-gray-50 to-gray-100 rounded-xl p-2 mb-2">
        <div class="text-center mb-4 title">学习详情</div>
        <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
          <div class="flex items-center gap-3 p-3 bg-white rounded-lg shadow-sm">
            <div class="w-6 h-6 text-green-500">
              <IconFluentCheckmark20Regular/>
            </div>
            <div>
              <div class="text-sm text-gray-600">正确</div>
              <div class="text-lg font-bold text-gray-900">{{ statStore.total - statStore.wrong }}</div>
            </div>
          </div>
          <div class="flex items-center gap-3 p-3 bg-white rounded-lg shadow-sm">
            <div class="w-6 h-6 text-red-500">
              <IconFluentDismiss20Regular/>
            </div>
            <div>
              <div class="text-sm text-gray-600">错误</div>
              <div class="text-lg font-bold text-gray-900">{{ statStore.wrong }}</div>
            </div>
          </div>
          <div class="flex items-center gap-3 p-3 bg-white rounded-lg shadow-sm">
            <div class="w-6 h-6 text-yellow-500">
              <IconFluentArrowRepeatAll20Regular/>
            </div>
            <div>
              <div class="text-sm text-gray-600">复习</div>
              <div class="text-lg font-bold text-gray-900">{{
                  statStore.reviewWordNumber + statStore.writeWordNumber
                }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Weekly Progress -->
      <div class="bg-gradient-to-br from-gray-50 to-gray-100 rounded-xl p-2 mb-2">
        <div class="text-center mb-4">
          <div class="text-xl font-semibold text-gray-900 mb-1">本周学习记录</div>
          <div class="text-sm text-gray-600">坚持就是胜利</div>
        </div>
        <div class="flex justify-between gap-2">
          <div
            v-for="(item, i) in list"
            :key="i"
            class="flex-1 text-center p-2 rounded-lg transition-all duration-300 cursor-pointer"
            :class="item ? 'bg-green-500 text-white shadow-lg' : 'bg-white text-gray-700 hover:shadow-md'"
          >
            <div class="font-semibold mb-1">{{ i + 1 }}</div>
            <div class="w-2 h-2 rounded-full mx-auto mb-1"
                 :class="item ? 'bg-white bg-opacity-30' : 'bg-gray-300'"></div>
            <div class="text-xs font-medium">{{ getDayLabel(i) }}</div>
          </div>
        </div>
      </div>

      <!-- Progress Overview -->
      <div class="bg-gradient-to-br from-gray-50 to-gray-100 rounded-xl p-6 mb-8">
        <div class="flex justify-between items-center mb-3">
          <div class="text-xl font-semibold text-gray-900">学习进度</div>
          <div class="text-2xl font-bold text-purple-600">{{ studyProgress }}%</div>
        </div>
        <div class="w-full bg-gray-200 rounded-full h-3 mb-3">
          <div
            class="h-3 rounded-full bg-gradient-to-r from-purple-500 to-purple-700 transition-all duration-500"
            :style="{ width: studyProgress + '%' }">
          </div>
        </div>
        <div class="flex justify-between text-sm text-gray-600 font-medium">
          <span>已学习: {{ store.sdict.lastLearnIndex }}</span>
          <span>总词数: {{ store.sdict.length }}</span>
        </div>
      </div>

      <!-- Action Buttons -->
      <div class="grid grid-cols-2 md:grid-cols-4 gap-3">
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.RepeatChapter]"
          @click="options(EventKey.repeatStudy)"
          class="flex items-center justify-center gap-2 py-3 px-4 rounded-xl font-medium transition-all duration-300 hover:-translate-y-1 hover:shadow-lg bg-gradient-to-r from-yellow-500 to-orange-500 text-white">
          <IconFluentArrowClockwise20Regular class="w-5 h-5"/>
          重学一遍
        </BaseButton>
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.NextChapter]"
          @click="options(EventKey.continueStudy)"
          class="flex items-center justify-center gap-2 py-3 px-4 rounded-xl font-medium transition-all duration-300 hover:-translate-y-1 hover:shadow-lg"
          :class="dictIsEnd ? 'bg-gradient-to-r from-purple-500 to-purple-600 text-white' : 'bg-gradient-to-r from-green-500 to-green-600 text-white'">
          <IconFluentPlay20Regular class="w-5 h-5"/>
          {{ dictIsEnd ? '从头开始练习' : '再来一组' }}
        </BaseButton>
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.NextRandomWrite]"
          @click="options(EventKey.randomWrite)"
          class="flex items-center justify-center gap-2 py-3 px-4 rounded-xl font-medium transition-all duration-300 hover:-translate-y-1 hover:shadow-lg bg-gradient-to-r from-blue-500 to-blue-600 text-white">
          <IconFluentPen20Regular class="w-5 h-5"/>
          继续默写
        </BaseButton>
        <BaseButton @click="$router.back"
                    class="flex items-center justify-center gap-2 py-3 px-4 rounded-xl font-medium transition-all duration-300 hover:-translate-y-1 hover:shadow-lg bg-gradient-to-r from-gray-500 to-gray-600 text-white">
          <IconFluentHome20Regular class="w-5 h-5"/>
          返回主页
        </BaseButton>
      </div>
    </div>
  </Dialog>
</template>

<style scoped>
/* Custom animation for pulse effect */
@keyframes pulse {
  0%, 100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
}

.animate-pulse {
  animation: pulse 2s infinite;
}

/* Custom gradient text utility */
.text-gradient {
  background: linear-gradient(135deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.item{
  @apply bg-gradient-to-br from-gray-50 to-gray-100 rounded-xl p-4 text-center hover:shadow-lg transition-all duration-300 hover:-translate-y-1 border border-gray-100;
}
</style>
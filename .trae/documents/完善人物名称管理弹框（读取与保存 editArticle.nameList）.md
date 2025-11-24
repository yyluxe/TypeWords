## 目标
- 在“人物名称管理”弹框中使用临时变量编辑名称列表，只在点击“确定”时写回 `editArticle.nameList: string[]`

## 数据结构
- `editArticle.nameList: string[]`
- 临时变量：`let nameListRef = $ref<string[]>([])`

## 生命周期
- 弹框打开时初始化：`nameListRef = cloneDeep(editArticle.nameList || [])`
- 弹框关闭时不写回：丢弃修改

## 交互设计
- 弹框 `v-model="showNameDialog"`、`:footer="true"`、`@close="showNameDialog = false"`、`@ok="saveNameList"`
- 按钮：
  - “添加名称” → `nameListRef.push('')`
  - 每行名称使用 `BaseInput v-model="nameListRef[i]"`
  - “删除”名称 → `nameListRef.splice(i,1)`

## 保存逻辑
- `saveNameList()`：
  - 清理：`trim()` + `filter(Boolean)`
  - 写回：`editArticle.nameList = cleaned`
  - 关闭弹框：`showNameDialog = false`

## 实现细节（EditArticle.vue 增加）
- 脚本：
```ts
let showNameDialog = $ref(false)
let nameListRef = $ref<string[]>([])

watch(() => showNameDialog, (v) => {
  if (v) nameListRef = cloneDeep(Array.isArray(editArticle.nameList) ? editArticle.nameList : [])
})

function addName() { nameListRef.push('') }
function removeName(i: number) { nameListRef.splice(i,1) }
function saveNameList() {
  const cleaned = nameListRef.map(s => (s ?? '').trim()).filter(Boolean)
  editArticle.nameList = cleaned
}
```
- 模板（620-628 区域）：
```vue
<Dialog title="人物名称管理"
        v-model="showNameDialog"
        :footer="true"
        @close="showNameDialog = false"
        @ok="saveNameList">
  <div class="p-4 pt-0 color-main w-150 flex flex-col gap-3">
    <div class="flex justify-between items-center">
      <div class="text-base">配置需要忽略的人名，练习时自动忽略这些名称</div>
      <BaseButton size="small" type="info" @click="addName">添加名称</BaseButton>
    </div>

    <div class="flex flex-col gap-2">
      <div class="flex items-center gap-2" v-for="(name,i) in nameListRef" :key="i">
        <BaseInput v-model="nameListRef[i]" placeholder="输入名称" size="large" />
        <BaseButton size="small" type="info" @click="removeName(i)">删除</BaseButton>
      </div>
    </div>
  </div>
</Dialog>
```

## 验证
- 打开弹框 → 编辑临时列表 → 点击“确定”后检查 `editArticle.nameList` 是否更新；点击关闭则不更新

## 注意
- 若类型仍为旧版 `string[][]`，请同步调整为 `string[]` 以与当前实现一致
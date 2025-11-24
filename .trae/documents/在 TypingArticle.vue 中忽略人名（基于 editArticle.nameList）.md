## 目标
- 在 `onTyping` 方法（291-382行）中判断当前/下一个单词是否为人名，并在练习时自动忽略（不要求输入、不提示错误、不停顿等待空格）。

## 数据来源
- 使用 `props.article.nameList: string[]`（来自编辑页保存），作为要忽略的人名列表。

## 匹配策略
- 构建一个人名集合 `nameSet`：
  - `trim()` 后的字符串；
  - 若开启 `ignoreCase` 则统一转小写匹配。
- 判定函数 `isNameWord(word: ArticleWord)`：
  - 仅当 `word.type === PracticeArticleWordType.Word` 时参与匹配；
  - 对 `word.word` 进行同样的规范化后 `nameSet.has(...)`。

## 处理时机与行为
- 在 `onTyping` 开始处、拿到 `currentWord` 后：
  - 若是“人名”，则直接跳过本词；若该词 `nextSpace` 为真，连带空格也跳过（避免进入 `isSpace` 状态）。
  - 跳过后继续处理当前按键：复用已有模式（如 `isSpace` 分支里）调用 `next()` 和递归 `onTyping(e)`。
- 在 `next()` 内也追加“人名跳过”逻辑（与已有忽略符号/数字类似），保证连续多个需要忽略的词可以被连续跳过：
  - 当 `currentWord` 是人名：
    - 若 `currentWord.nextSpace` 为真：`isSpace = false`；
    - 递归调用 `next()` 继续到下一个词；
  - 否则正常 `emit('nextWord', currentWord)`。

## 代码改动点
- 在组件顶部或方法内构建 `nameSet`（建议用 `$computed`）：
```ts
const nameSet = $computed(() => {
  const list = props.article?.nameList ?? []
  return new Set(list.map(s => (settingStore.ignoreCase ? s.toLowerCase() : s).trim()).filter(Boolean))
})
function isNameWord(w: ArticleWord) {
  if (w.type !== PracticeArticleWordType.Word) return false
  const token = (settingStore.ignoreCase ? w.word.toLowerCase() : w.word).trim()
  return nameSet.has(token)
}
```
- 在 `onTyping` 里，`let currentWord = currentSentence.words[wordIndex]` 之后：
```ts
if (isNameWord(currentWord)) {
  // 跳过当前人名，连带空格
  isSpace = false
  const savedTypingFlag = isTyping
  next()
  isTyping = false
  return onTyping(e)
}
```
- 在 `next()` 内，设置 `currentWord` 后、`emit('nextWord', currentWord)` 之前：
```ts
if (isNameWord(currentWord)) {
  // 人名与后续空格都跳过
  isSpace = false
  return next()
}
```

## 注意事项
- 保持与现有忽略规则一致（符号/数字已通过 `ignoreSymbol` 处理），人名跳过逻辑与其同层级。
- 忽略人名时不触发 `wrong` 或提示音，不进入 `isSpace` 等待。
- 若连续出现多个需要忽略的词（如人名+标点+人名），递归 `next()` 将逐个跳过。

## 验证
- 在包含人名的文本中练习：
  - 人名处不需要输入，光标自动跳到下一非忽略词；
  - 不会卡在空格等待；
  - 大小写忽略效果符合 `settingStore.ignoreCase` 设置。

## 交付
- 按上述方案在 `TypingArticle.vue` 中实现辅助函数与两处调用点的改动；
- 仅修改该文件，不影响其他页面。
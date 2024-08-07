<template>
  <QuestionRuleContainer
    v-if="visible"
    :moduleConfig="questionConfig"
    :indexNumber="indexNumber"
    :showTitle="true"
    @change="handleChange"
  ></QuestionRuleContainer>
</template>
<script setup>
import { unref, computed, watch } from 'vue'
import QuestionRuleContainer from '../../materials/questions/QuestionRuleContainer'
import { useVoteMap } from '@/render/hooks/useVoteMap'
import { useShowOthers } from '@/render/hooks/useShowOthers'
import { useShowInput } from '@/render/hooks/useShowInput'
import { cloneDeep } from 'lodash-es'
import { ruleEngine } from '@/render/hooks/useRuleEngine.js'
import { useQuestionStore } from '../stores/question'
import { useSurveyStore } from '../stores/survey'

import { NORMAL_CHOICES, RATES, QUESTION_TYPE } from '@/common/typeEnum.ts'

const props = defineProps({
  indexNumber: {
    type: [Number, String],
    default: 1
  },
  moduleConfig: {
    type: Object,
    default: () => {
      return {}
    }
  }
})
const emit = defineEmits(['change'])
const questionStore = useQuestionStore()
const surveyStore = useSurveyStore()

const formValues = computed(() => {
  return surveyStore.formValues
})
const questionConfig = computed(() => {
  let moduleConfig = props.moduleConfig
  const { type, field, options = [], ...rest } = cloneDeep(moduleConfig)
  // console.log(field,'这里依赖的formValue，所以change时会触发重新计算')
  let alloptions = options
  if (type === QUESTION_TYPE.VOTE) {
    const { options, voteTotal } = useVoteMap(field)
    const voteOptions = unref(options)
    alloptions = alloptions.map((obj, index) => Object.assign(obj, voteOptions[index]))
    moduleConfig.voteTotal = unref(voteTotal)
  }
  if (
    NORMAL_CHOICES.includes(type) &&
    options.filter((optionItem) => optionItem.others).length > 0
  ) {
    let { options, othersValue } = useShowOthers(field)
    const othersOptions = unref(options)
    alloptions = alloptions.map((obj, index) => Object.assign(obj, othersOptions[index]))
    moduleConfig.othersValue = unref(othersValue)
  }
  if (
    RATES.includes(type) &&
    rest?.rangeConfig &&
    Object.keys(rest?.rangeConfig).filter((index) => rest?.rangeConfig[index].isShowInput).length >
      0
  ) {
    let { rangeConfig, othersValue } = useShowInput(field)
    moduleConfig.rangeConfig = unref(rangeConfig)
    moduleConfig.othersValue = unref(othersValue)
  }

  return {
    ...moduleConfig,
    options: alloptions,
    value: formValues.value[props.moduleConfig.field]
  }
})

const { field } = props.moduleConfig

const visible = computed(() => {
  // computed有计算缓存，当match有变化的时候触发重新计算
  return ruleEngine.match(field, 'question', formValues.value)
})

watch(
  () => visible.value,
  (newVal, oldVal) => {
    // 题目从显示到隐藏，需要清空值
    const { field, type, innerType } = props.moduleConfig
    if (!newVal && oldVal) {
      let value = ''
      // 题型是多选，或者子题型是多选（innerType是用于投票）
      if (type === QUESTION_TYPE.CHECKBOX || innerType === QUESTION_TYPE.CHECKBOX) {
        value = value ? [value] : []
      }
      const data = {
        key: field,
        value: value
      }
      surveyStore.changeData(data)
    }
  }
)

const handleChange = (data) => {
  emit('change', data)
  // 处理投票题
  if (props.moduleConfig.type === QUESTION_TYPE.VOTE) {
    questionStore.updateVoteData(data)
  }
}
</script>

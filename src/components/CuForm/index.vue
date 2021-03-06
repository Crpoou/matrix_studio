<template>
  <template v-if="isShowRead">
    <div class="bg-pink">卡片所有字段！，比作阅读模式</div>
    <section class="grid cu-form">
      <template v-for="(value, key) in step" :key="key">
        {{ key }}：<span>{{ typeof value === 'object' ? '对象类型' : value }}</span>
      </template>
    </section>
  </template>
  <template v-if="isShowForm">
    <div class="bg-pink">卡片表单！，比作编辑模式</div>
    <section class="grid cu-form">
      <template v-for="(value, key) in step.form" :key="key">
        {{ key }}:
        <!-- eslint-disable-next-line vue/no-mutating-props -->
        <input v-model="step.form[key]" type="text" :validate="!ValidateCollection.get(key)?.value" />
      </template>
    </section>
  </template>
  <template v-if="isShowBranch">
    <div class="bg-pink">子卡片区域！</div>
    <section class="grid-justify-center custom-scroll overlay-y">
      <cu-branch v-for="branch in step.branchs" :key="branch.uuid" :branch="branch" />
    </section>
  </template>
</template>

<script lang="ts">
import { ComputedRef, PropType, computed, defineComponent, inject, onBeforeUnmount, onMounted } from 'vue'
import { EmptyObj, EmptyStr, ProvideInjectKeyMap } from '@constant'
import { ON_FORM_BEFORE_UN_MOUNT, ON_FORM_MOUNTED } from '@store/Cube'
import { useChildValidate, useValidate } from '@hooks'
import { ComputedGetter } from '@vue/reactivity'
import CuBranch from '@components/CuBranch/index.vue'
import { FALSE } from '@share'
import { Step } from '@interface'

// 空字符取反为true，校验通过，报错字符串取反为false，校验不通过，未定义错误，取反为true，校验通过

export default defineComponent({
  name: 'CuForm',
  components: { CuBranch },
  props: {
    step: {
      type: Object as PropType<Step>,
      required: true
    },
    isEdit: {
      type: Boolean,
      required: true
    }
  },
  setup(props: Readonly<{ step: Step; isEdit: boolean }>) {
    /** 从Flow组件接受注入，是否处于活跃的TAB页中 */
    const isInCurrentTab = inject<ComputedRef<boolean>>(ProvideInjectKeyMap.CURRENT_TAB, computed(FALSE))
    /** 拿到容器卡的禁用状态，即为当前表单的禁用状态 */
    const isDisabled = inject<ComputedRef<boolean>>(ProvideInjectKeyMap.DISABLED, computed(FALSE))
    /** 是否展示阅读模式 */
    const isShowRead = computed(() => isInCurrentTab.value && !props.isEdit)
    /** 是否展示编辑模式 */
    const isShowForm = computed(() => isInCurrentTab.value && props.isEdit)
    /** 是否展示子卡片区域 */
    const isShowBranch = computed(() => props.step.branchs?.size)
    /** 拿到表单所有字段 */
    const formKeys = Object.keys(props.step.form || EmptyObj)
    /** 创建所有字段的校验函数，添加到Map构造参数 */
    const KeyValidateFunMap: Map<string, ComputedGetter<string>> = new Map()
    for (const key of formKeys) {
      KeyValidateFunMap.set(key, () => {
        if (isDisabled.value) return EmptyStr // 如果被禁用，跳过校验
        if (props.step.form[key]) return EmptyStr // 如果字段有值，通过校验
        return '字段不能为空' // 否则校验不通过
      })
    }
    // 如果是容器卡
    if (props.step.branchs) {
      // 随便取一个字段
      KeyValidateFunMap.set('needChild', () => {
        if (isDisabled.value) return EmptyStr // 如果被禁用，跳过校验
        if (props.step.branchs?.size) return EmptyStr
        return '至少需要一个字卡片'
      })
    }

    /** 创建当前FORM表单的校验集合 */
    const { ValidateCollection } = useValidate(KeyValidateFunMap)
    const { ChildValidateCollection } = useChildValidate(props)

    onMounted(() => ON_FORM_MOUNTED(props.step, { ValidateCollection, ChildValidateCollection }))
    onBeforeUnmount(() => ON_FORM_BEFORE_UN_MOUNT(props.step))

    return { isInCurrentTab, isShowRead, isShowForm, isShowBranch, ValidateCollection }
  }
})
</script>
<style lang="scss">
.cu-form {
  grid-template-columns: auto 1fr;
  padding: 1em;
}
</style>

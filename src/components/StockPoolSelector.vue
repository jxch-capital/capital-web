<template>
  <n-space size="small">
    <date-selector :condition="condition"/>
    <n-dropdown size="small" trigger="hover" :options="stockPoolOptionsComputed" @select="stockPoolHandleSelect">
      <n-button size="small" :loading="condition.loading" @click="update">
        {{ condition.stockPoolSelectLabel }}
      </n-button>
    </n-dropdown>
  </n-space>
</template>

<script>
import {computed, defineComponent, reactive, toRaw, toRefs} from "vue";
import DateSelector from "cc/DateSelector.vue";
import {apis} from "@/api";
import dayjs from "dayjs";
import {useMessage} from "naive-ui";
import { useStore } from 'vuex'

export default defineComponent({
  name: "StockPoolSelector",
  components: {DateSelector},
  props: {
    autoSelector: {
      type: Boolean,
      default: true
    },
    kLines: Object
  },
  emits: ['update:kLines'],
  setup(props, {emit}) {
    const {autoSelector} = toRefs(props)
    const store = useStore()
    const message = useMessage()
    const template = "YYYY-MM-DD"
    const condition = reactive({
      last: 1,
      cycle: 'year',
      stockPool: [],
      loading: false,
      stockPoolSelectLabel: '',
      stockPoolSelectKey: '',
      kLines: '',
    })

    const stockPoolOptionsComputed = computed(() => {
      const rawStockPoolOptions = toRaw(condition.stockPool)
      return Object.keys(rawStockPoolOptions).map(key => {
        return {label: rawStockPoolOptions[key]['alias'] + ` (${rawStockPoolOptions[key]['codes'].length})`, key: key}
      })
    })

    function queryStockPool() {
      if (Object.keys(store.state.stockPool).length > 0) {
        condition.stockPool = store.state.stockPool
        const keys = Object.keys(condition.stockPool)
        stockPoolHandleSelect(keys[keys.length - 1])
        return
      }

      apis.capital_service_apis.query_stock_pool().then((res) => {
        condition.stockPool = res['data']
        store.commit('setStockPool', {pool:res['data']})
        const keys = Object.keys(condition.stockPool)
        stockPoolHandleSelect(keys[keys.length - 1])
      }).catch((e) => {
        console.log(e)
        message.warning('请检查网络并稍后重试，或查看控制台报错信息：' + e['message'])
      })
    }

    function update() {
      const cacheKey = `${condition.stockPoolSelectKey}-${condition.last}-${condition.cycle}`
      if (store.state.stockPoolData[cacheKey]) {
        condition.kLines = store.state.stockPoolData[cacheKey]
        emit('update:kLines', condition.kLines)
        return
      }

      condition.loading = true
      apis.capital_service_apis.query_k_json_by_stock_pool({
        "stock_pool": condition.stockPoolSelectKey,
        "start": dayjs(new Date()).subtract(condition.last, condition.cycle).format(template),
        "end": dayjs(new Date()).format(template)
      }).then((res) => {
        condition.kLines = res['data']
        emit('update:kLines', condition.kLines)
        message.success(condition.stockPoolSelectLabel + '股票池加载完成')
        store.commit('setKJsonByStockPool', {key:cacheKey, value:res['data']})
      }).catch((e) => {
        console.log(e)
        message.warning('请检查网络并稍后重试，或查看控制台报错信息：' + e['message'])
      }).finally(() => {
        condition.loading = false
      })
    }

    function stockPoolHandleSelect(key) {
      condition.stockPoolSelectKey = key
      condition.stockPoolSelectLabel = condition.stockPool[key]['alias'] + ` (${condition.stockPool[key]['codes'].length})`
      if (toRaw(props.autoSelector)) {
        update()
      }
    }

    queryStockPool()

    return {
      stockPoolOptionsComputed,
      stockPoolHandleSelect,
      update,
      condition,
    }
  }
})
</script>

<style scoped>

</style>
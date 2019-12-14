<template>
  <div :class="className" :style="{height:height,width:width}" />
</template>

<script>
import echarts from 'echarts'
require('echarts/theme/macarons') // echarts theme
import resize from './mixins/resize'

export default {
  mixins: [resize],
  props: {
    className: {
      type: String,
      default: 'chart'
    },
    width: {
      type: String,
      default: '100%'
    },
    height: {
      type: String,
      default: '300px'
    }
  },
  data() {
    return {
      chart: null
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initChart()
    })
  },
  beforeDestroy() {
    if (!this.chart) {
      return
    }
    this.chart.dispose()
    this.chart = null
  },
  methods: {
    initChart() {
      this.chart = echarts.init(this.$el, 'macarons')

      this.chart.setOption({
        title: {
          text: '本週餐點口味統計'
        },
        tooltip: {
          trigger: 'item',
          formatter: '{a} <br/>{b} : {c} ({d}%)'
        },
        legend: {
          left: 'center',
          bottom: '10',
          data: ['雙拼', '卡拉雞腿排飯', '迷迭香烤豬飯', '玫瑰油雞飯', '香蒜烤雞飯', '其他']
        },
        series: [
          {
            name: '',
            type: 'pie',
            radius: '55%',
            center: ['40%', '50%'],
            data: [
              { value: 551, name: '雙拼' },
              { value: 459, name: '卡拉雞腿排飯' },
              { value: 451, name: '迷迭香烤豬飯' },
              { value: 411, name: '玫瑰油雞飯' },
              { value: 147, name: '香蒜烤雞飯' },
              { value: 153, name: '味噌燒肉飯' },
              { value: 246, name: '其他' }
            ],
            animationEasing: 'cubicInOut',
            animationDuration: 2600
          }
        ]
      })
    }
  }
}
</script>

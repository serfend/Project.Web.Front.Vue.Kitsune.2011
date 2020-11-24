<template>
  <div :style="{height:height,width:width,'background-color':'#fff2'}" />
</template>

<script>
import echarts from 'echarts'
import 'echarts-gl'
import { debounce } from '@/utils'
import graph from './tmp_data'
export default {
  name: 'ConnectionGraph',
  props: {
    width: {
      type: String,
      default: '100%'
    },
    height: {
      type: String,
      default: '300px'
    },
    fileLoad: {
      type: Function,
      default: null
    },
    speed: {
      type: Number,
      default: 1
    },
    color: {
      type: Array,
      default: () => []
    },
    data: {
      type: Object,
      default: () => { } // {#typeName#:{from,to,value}}省份对应的坐标
    }
  },
  data() {
    return {
      chart: null,
      series: [],
    }
  },
  computed: {
    updatedData() {
      return debounce(() => {
        this.updateData()
      }, 1000)
    }
  },
  watch: {
    data: {
      handler(v) {
        this.$nextTick(() => {
          this.updatedData()
        })
      },
      deep: true
    }
  },
  mounted() {
    this.initChart()
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
      this.chart = echarts.init(this.$el)
      this.initChartSkeleton()
    },
    refreshData() {
      const series = this.series
      // console.log(series)
      this.chart.setOption({
        series: series
      })
    },
    initChartSkeleton() {
      var categories = ['内部', '外部', '业务']
      graph.nodes.forEach(function (node, index) {
        node.value = node.symbolSize
        // node.symbolSize /= 1.5
        node.label = {
          show: node.symbolSize > 30
        }
        node.id = index
        node.category = Math.floor(Math.random() * 3)
      })
      const option = {
        title: {
          text: '数据交互关系',
          subtext: 'Default layout',
          top: 'bottom',
          left: 'right'
        },
        tooltip: {},
        legend: [{
          // selectedMode: 'single',
          data: categories,
          bottom: '5%',
          textStyle: {
            color: '#fff',
            fontSize: 24
          }
        }],
        animationDuration: 1500,
        animationEasingUpdate: 'quinticInOut',
        series: [
          {
            name: '流量态势',
            type: 'graph',
            layout: 'force',
            draggable: true,
            color: ['#33c', '#c33', '#3c3'],
            force: {
              repulsion: 500,
              edgeLength: [10, 50],
              friction: 0.6
            },
            data: graph.nodes,
            links: graph.links,
            categories: categories.map(i => ({ name: i })),
            roam: true,
            focusNodeAdjacency: true,
            itemStyle: {
              borderColor: '#fff',
              borderWidth: 1,
              shadowBlur: 10,
              shadowColor: 'rgba(0, 0, 0, 0.3)'
            },
            emphasis: {
              lineStyle: {
                width: 2
              }
            }
          }
        ]
      }
      this.chart.setOption(option)
    }
  }
}
</script>

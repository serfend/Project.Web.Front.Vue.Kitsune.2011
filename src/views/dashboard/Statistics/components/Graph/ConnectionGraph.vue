<template>
  <div style="width:100%;height:100%">
    <div :style="{height:height,width:width,'background-color':'#fff2'}" />
    <ConnectionDetail :visible.sync="show_detail" :data="show_detail_data" />
  </div>
</template>

<script>
import ConnectionDetail from './ConnectionDetail'
import echarts from 'echarts'
import 'echarts-gl'
import { debounce } from '@/utils'
import graph from './tmp_data'
export default {
  name: 'ConnectionGraph',
  components: { ConnectionDetail },
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
      show_detail: false,
      show_detail_data: null
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
    handle_show_detail(event) {
      // show dialog for detail
      this.show_detail = true
      this.show_detail_data = event
    },
    initChartSkeleton() {
      this.chart.on('dblclick', (event) => { this.handle_show_detail(event) })
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
            // layout: 'force',
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

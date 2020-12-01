<template>
  <div :style="{height:height,width:width,'background-color':'#fff2'}" />
</template>

<script>
import echarts from 'echarts'
import 'echarts-gl'
import { debounce } from '@/utils'
import { formatSciItem } from '@/utils/math'
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
      graph: []
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
      this.chart.showLoading()
      this.fileLoad('connection-data202012012128-outer.json').then(data => {
        this.graph = {
          nodes: data
        }
        this.initChartSkeleton()
        this.chart.hideLoading()
      })
    },
    refreshData() {
      const series = this.series
      // console.log(series)
      this.chart.setOption({
        series: series
      })
    },
    handle_show_detail(event) {
      this.$refs.ConnectionDetail.show_dialog(event)
      // show dialog for detail
      // let routeUrl = this.$router.resolve({
      //   path: '/alicenter/detail',
      //   query: {
      //     id: id
      //   }
      // })
      // window.open(routeUrl.href, '_blank')
    },
    getCategory(ip) {
      const arrs = ip.split('.')
      switch (parseInt(arrs[0]) + parseInt(arrs[1])) {
        case 5 + 195: return 1
        case 5 + 221: return 0
        default: return 2
      }
    },
    initChartSkeleton() {
      const self = this
      const categories = ['前台', '后台', '外部', '防御成功', '威胁']
      const colors = ['#33c', '#3c3', '#aaa', '#cc3', '#c33']
      let links = []
      const graph = this.graph
      console.log(graph)
      const item_dict = {}
      graph.nodes.forEach((node, index) => {
        if (item_dict[node.id]) {
          console.warn(node.id, 'is already exist')
        }
        item_dict[node.id] = node
        node.symbolSize = Math.log(node.value) * 3 + 2 + (node.danger ? 5 : 0)
        node.label = {
          show: node.value > 30
        }
      })
      graph.nodes.forEach((node, index) => {
        if (node.danger) {
          node.category = 4
        } else if (node.link && node.link.length > 10 && node.link.reduce((prev, cur) => {
          const c = item_dict[cur]
          if (!c) {
            console.warn(cur, 'is not exist')
            return parseInt(prev)
          }
          return c.value + parseInt(prev)
        }, 0) < node.link.length * 21) {
          node.category = 3
        } else {
          node.category = self.getCategory(node.name)
        }
        if (node.link) links = links.concat(node.link.map(i => ({ target: index, source: i - 1 })))
      })
      const option = {
        title: {
          text: '数据交互关系图',
          subtext: 'Default layout',
          top: 'bottom',
          left: 'right',
          textStyle: {
            color: '#fff'
          }
        },
        tooltip: {
          formatter: (params, ticket, async_callback) => {
            let r = []
            if (params.data.source) {
              const { source, target } = params.data
              r.push(`${graph.nodes[source].name} > ${graph.nodes[target].name}`)
            } else {
              const { category, details, value, link } = params.data
              const sci = formatSciItem(value)
              let links = ''
              if (link) {
                links = `${link.length}个连接`
              }
              r.push(`${categories[category]} ${params.name} (${sci.value}${sci.suffix}B)${links}<hr>${details ? '被连接情况' : '无详细信息'}:`)
              if (details) {
                const arrays = Object.keys(details).map(p => {
                  const sci_port = formatSciItem(details[p])
                  return `【${p}】${sci_port.value}${sci_port.suffix}B`
                })
                r = r.concat(arrays)
              }
            }
            return r.join('<br>')
          }
        },
        legend: [{
          // selectedMode: 'single',souu
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
            color: colors,
            force: {
              repulsion: 500,
              edgeLength: [10, 150],
              friction: 0.6
            },
            data: graph.nodes,
            links: links,
            categories: categories.map(i => ({ name: i })),
            roam: true,
            edgeSymbol: ['none', 'arrow'],
            // focusNodeAdjacency: true,
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
<style lang="scss" scoped>
.full-panel {
  height: 100%;
  width: 100%;
}
</style>

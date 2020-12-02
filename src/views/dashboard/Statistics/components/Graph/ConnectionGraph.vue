<template>
  <div class="full-panel">
    <el-card class="chart-option">
      <el-form label-width="5rem">
        <el-form-item label="ip">
          <el-select
            v-model="filter.ip"
            filterable
            remote
            reserve-keyword
            placeholder="ip地址/段"
            :remote-method="search_ip"
            :loading="search_ip_loading"
          >
            <el-option
              v-for="node in ip_filter_options"
              :key="node.id"
              :label="`[${node.id}]${node.name}`"
              :value="node.id"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="流量大小">
          <el-slider v-model="filter.traffic" />
        </el-form-item>
      </el-form>
    </el-card>
    <chart ref="chart" :style="{height:height,width:width,'background-color':'#fff2'}" />
  </div>
</template>

<script>
import chart from './chart'
import echarts from 'echarts'
import 'echarts-gl'
import { formatSciItem } from '@/utils/math'
export default {
  name: 'ConnectionGraph',
  components: { chart },
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
    categories: {
      type: Array,
      default: () => {
        console.log('init categories')
        return ['前台', '后台', '外部', '防御成功', '威胁']
      }
    },
    color: {
      type: Array,
      default: () => ['#05f', '#0f0', '#aaa', '#e50', '#f33']
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
      graph: [],
      item_dict: {},
      filter: {
        ip: 0,
        last_ips: [],
        traffic: 0
      },
      ip_filter_options: [],
      search_ip_loading: false
    }
  },
  watch: {
    'filter.ip': {
      handler(v) {
        const node = this.item_dict[v]
        const prev_size = node.symbolSize
        node.symbolSize = 150
        this.refreshData()
        setTimeout(() => {
          node.symbolSize = prev_size
          this.refreshData()
        }, 5000)
      }
    },
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
    search_ip(query) {
      if (query !== '') {
        this.search_ip_loading = true
        this.ip_filter_options = this.graph.nodes.filter(i => i.name.indexOf(query) > -1)
        this.search_ip_loading = false
      } else {
        this.search_ip_loading = []
      }
    },
    initChart() {
      this.chart = echarts.init(this.$refs.chart.$el)
      this.chart.showLoading(null, { text: '数据加载中' })
      const actions = [this.fileLoad('connection-data202012022300.json'), this.initChartSkeleton()]
      Promise.all(actions).then(data => {
        const graph = data[0]
        this.chart.hideLoading()
        this.chart.showLoading(null, { text: '数据处理中' })
        this.resetData(graph)
        setTimeout(() => {
          this.refreshData()
          this.chart.hideLoading()
        }, 200)
      })
    },
    resetData(nodes) {
      const self = this
      const item_dict = this.item_dict
      let links = []
      nodes.forEach((node, index) => {
        if (item_dict[node.id]) {
          console.warn(node.id, 'is already exist')
        }
        node.index = index
        item_dict[node.id] = node
        if (node.value <= 0) node.value = 1
        node.symbolSize = Math.log(node.value) * 2 + 15 + (node.danger ? 45 : 0)
        node.label = {
          show: node.value > 30
        }
      })
      nodes.forEach((node, index) => {
        if (node.danger) {
          node.category = 4
        } else {
          node.category = self.getCategory(node.name)
        }
        if (node.link) {
          const node_links = Object.keys(node.link)
          links = links.concat(
            node_links
              .filter(i => item_dict[i])
              .map(idx => {
                const item = item_dict[idx]
                return { target: index, source: item.index, links: node.link[idx] }
              }))
        }
      })
      console.log(nodes)
      this.graph = {
        nodes, links
      }
    },
    refreshData() {
      const series = this.series
      const { nodes, links } = this.graph
      series[0].data = nodes
      series[0].links = links
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
    handle_sort_port_detail_info(arr, details_dict) {
      return arr.sort((i1, i2) => details_dict[i2] - details_dict[i1]).slice(0, 10).map(p => {
        const sci_port = formatSciItem(details_dict[p])
        return `【${p}】${sci_port.value}${sci_port.suffix}B`
      })
    },
    format_tooltip(params, ticket, async_callback) {
      let r = []
      if (params.data.id) {
        const { id, category, details, value, link } = params.data
        const sci = value <= 1 ? { value: '无', suffix: '' } : formatSciItem(value)
        if (value > 1) sci.suffix = `${sci.suffix}B`
        r.push(`设备序号： ${id}`)
        r.push(`所属区域： ${params.marker}${this.categories[category]}`)
        r.push(`主机名称： ${params.name}`)
        r.push(`接收流量： ${sci.value}${sci.suffix}`)
        if (details) {
          const details_keys = Object.keys(details)
          r.push(`关联节点： ${link && Object.keys(link).length}个<hr>接收端口：${details_keys.length}个`)
          const arrays = this.handle_sort_port_detail_info(details_keys, details)
          r = r.concat(arrays)
        }
      } else {
        const { source, target, links } = params.data
        r.push(`${this.graph.nodes[source].name} -> ${this.graph.nodes[target].name}`)
        const links_arr_key = Object.keys(links)
        const total_traffic = links_arr_key.reduce((prev, cur) => prev + links[cur], 0)
        const sci_total_traffic = formatSciItem(total_traffic)
        r.push(`流量总计:${sci_total_traffic.value}${sci_total_traffic.suffix}B<hr>连接情况：`)
        r = r.concat(this.handle_sort_port_detail_info(links_arr_key, links))
      }
      return r.join('<br>')
    },
    initChartSkeleton() {
      return new Promise(res => {
        const { categories, color } = this
        this.series = [{
          name: '流量态势',
          type: 'graph',
          layout: 'force',
          color,
          force: {
            repulsion: 500,
            edgeLength: [10, 150],
            friction: 0.6
          },
          categories: categories.map(i => ({ name: i })),
          roam: true,
          edgeSymbol: ['none', 'arrow'],
          draggable: true,
          focusNodeAdjacency: true,
          // itemStyle: {
          //   borderColor: '#fff',
          //   borderWidth: 1,
          //   shadowBlur: 10,
          //   shadowColor: 'rgba(0, 0, 0, 0.3)'
          // },
          emphasis: {
            lineStyle: {
              width: 2
            }
          }
        }]
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
            formatter: this.format_tooltip
          },
          legend: [{
            data: categories,
            bottom: '5%',
            textStyle: {
              color: '#fff',
              fontSize: 24
            }
          }],
          series: this.series
        }
        this.chart.setOption(option)
        res()
      })
    }
  }
}
</script>
<style lang="scss" scoped>
.full-panel {
  height: 100%;
  width: 100%;
}
.chart-option {
  position: absolute;
  z-index: 1000;
  width: 2rem;
  left: -1rem;
  transition: all 0.5s ease;
  &:hover {
    left: 0rem;
    width: 25rem;
  }
}
</style>

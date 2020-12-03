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
              :label="`[${node.id}]${node.name} ${format_sci(node.output)}/${format_sci(node.input)}`"
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
      default: () => ['#05f', '#0f0', '#aaa', '#7DE5FB', '#f33']
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
        this.chart.dispatchAction({
          type: 'focusNodeAdjacency',
          // 使用 seriesId 或 seriesIndex 或 seriesName 来指定 series.
          // seriesId: 'xxx',
          seriesIndex: 0,
          // seriesName: 'nnn',
          // 使用 dataIndex 来指定目标节点，或者使用 edgeDataIndex 来指定目标边。
          dataIndex: node.index,
          // edgeDataIndex: 5
        })
      }
    },
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
    format_sci(node) {
      const f = formatSciItem(node)
      return `${f.value}${f.suffix}B`
    },
    search_ip(query) {
      if (query !== '') {
        this.search_ip_loading = true
        this.ip_filter_options =
          this.graph.nodes
            .filter(i => i.name.indexOf(query) > -1)
            .sort((i1, i2) => i2.traffic_sum - i1.traffic_sum)
        this.search_ip_loading = false
      } else {
        this.search_ip_loading = []
      }
    },
    initChart() {
      this.chart = echarts.init(this.$refs.chart.$el)
      this.chart.showLoading(null, { text: '数据加载中' })
      const actions = [this.fileLoad('connection-data202012031214.json'), this.initChartSkeleton()]
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
        node.traffic_sum = node.input + node.output
        item_dict[node.id] = node
        if (node.traffic_sum <= 0) node.traffic_sum = 1
        node.symbolSize = Math.log(node.traffic_sum) * 5 + 15 + (node.danger ? 80 : 0)
        node.label = {
          show: node.traffic_sum > 30
        }
      })
      nodes.forEach((node, index) => {
        if (node.danger) {
          node.category = 4
        } else if (node.link && Object.keys(node.link)
          .findIndex(i =>
            Object.keys(node.link[i])
              .findIndex(port => node.link[i][port] < 1) > -1) > -1) {
          node.category = 3
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
      this.graph = {
        nodes, links
      }
    },
    refreshData() {
      const series = this.series
      const { nodes, links } = this.graph
      series[0].data = nodes
      series[0].links = links
      this.chart.setOption({
        series: series
      })
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
      const port_show_count = 10
      const result = arr
        .sort((i1, i2) => details_dict[i2] - details_dict[i1])
        .slice(0, port_show_count)
        .map(p => {
          const sci_port = this.format_sci(details_dict[p])
          return `【${p}】${sci_port}`
        })
      if (arr.length > port_show_count) {
        const other_port_traffic = arr.slice(10).reduce((prev, cur) => prev + details_dict[cur], 0)
        result.push(`其他${arr.length - port_show_count}个端口 ${this.format_sci(other_port_traffic)}流量...`)
      }
      return result
    },
    format_tooltip(params, ticket, async_callback) {
      let r = []
      if (params.data.id) {
        const { id, category, details, traffic_sum, input, output, link } = params.data
        r.push(`设备序号： ${id}`)
        r.push(`所属区域： ${params.marker}${this.categories[category]}`)
        r.push(`主机名称： ${params.name}`)
        r.push(`总计流量： ${this.format_sci(traffic_sum)}`)
        r.push(`连入交互： ${this.format_sci(input)}`)
        r.push(`连出交互： ${this.format_sci(output)}`)
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
        r.push(`流量总计:${this.format_sci(total_traffic)}<hr>连接情况：${links_arr_key.length}个端口`)
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
          zoom: 10e-4 * 2,
          force: {
            repulsion: 5 * 10e5,
            edgeLength: [10, 1500],
            friction: 0.6
          },
          categories: categories.map(i => ({ name: i })),
          roam: true,
          edgeSymbol: ['none', 'arrow'],
          draggable: true,
          focusNodeAdjacency: true,
          itemStyle: {
            borderColor: '#fff',
            borderWidth: 2,
            shadowBlur: 10,
            shadowColor: 'rgba(0, 0, 0, 0.3)'
          },
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

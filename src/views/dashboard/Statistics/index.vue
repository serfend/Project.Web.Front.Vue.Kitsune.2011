<template>
  <div id="container" class="container">
    <div v-loading="loading" class="container-bg">
      <div class="statistics-title">
        <h1 class="content" style="margin:0.1rem">全网威胁情报态势</h1>
        <TimeCenter :time-sync-method="timeZone" />
      </div>
      <Breadcrumb style="position:absolute;margin-left:2rem;margin-top:-1rem" />
      <section class="mainbox">
        <div class="column">
          <Square>
            <RankingBar
              slot="chart"
              :data="ranking_data.data"
              :legend="ranking_data.legend"
              height="100%"
            />
          </Square>
          <Square>
            <WarningLine ref="warningLine" slot="chart" height="100%" />
          </Square>
        </div>
        <div class="column">
          <MembersCounter :setting="memberSetting" @updateItems="requireUpdateItems" />
          <div class="map">
            <div class="map1" />
            <div class="map2" />
            <div class="map3" />
            <ConnectionGraph height="100%" />
          </div>
        </div>
      </section>
      <div style="display:flex;position:fixed;bottom:0;">
        <EchartGeoLoader
          ref="echartGeoDriver"
          :file-load="requestFile"
          :complete.sync="echartGeoComplete"
        />
        <SettingEngine ref="setting" :setting.sync="setting" @closed="settingUpdated" />
      </div>
    </div>
  </div>
</template>

<script>
// import Flexible from './js/flexible'
import Square from './components/Square'

import TimeCenter from './components/NumberCounter/TimeCenter'
import { timeZone } from '@/api/common/static'
import EchartGeoLoader from './components/Engine/EchartGeoLoader'
import SettingEngine from './components/Engine/SettingEngine'
import { getProp, modify } from '@/utils/data-handle'

import { requestFile, download } from '@/api/common/file'
import { debounce } from '@/utils'
import RankingBar from './components/Bar/RankingBar'
import WarningLine from './components/Bar/WarningLine'
import MembersCounter from './components/NumberCounter/MembersCounter'
import ConnectionGraph from './components/Graph/ConnectionGraph'
import Breadcrumb from '@/components/Breadcrumb'
const cmp = (a, b) => b.v - a.v
const Mock = require('mockjs')
const legend = new Array(10).fill(0).map(i => Mock.Random.ip())
export default {
  name: 'Statistics',
  components: {
    Breadcrumb,
    Square,
    TimeCenter,
    EchartGeoLoader,
    SettingEngine,
    RankingBar,
    WarningLine,
    MembersCounter,
    ConnectionGraph
  },
  data: () => ({
    // flexible: new Flexible(window, document),
    loading: false,
    echartGeoComplete: false,
    data: null,
    lastUpdate: new Date(),
    ranking_data: {
      data: [
        {
          name: '端口占用',
          series: new Array(10).fill(0).map((v, i) => ({
            v: Mock.Random.natural(0, 100),
            ip: legend[i]
          })).sort(cmp)
        },
        {
          name: '连接数',
          series: new Array(10).fill(0).map((v, i) => ({
            v: Mock.Random.natural(0, 100),
            ip: legend[i]
          })).sort(cmp)
        },
        {
          name: '数据包数',
          series: new Array(10).fill(0).map((v, i) => ({
            v: Mock.Random.natural(0, 100),
            ip: legend[i]
          })).sort(cmp)
        },
      ], legend: legend
    }
  }),
  computed: {
    updatedSetting() {
      return debounce(() => {
        this.settingUpdated()
      }, 1000)
    },
    setting: {
      get() {
        return this.$store.state.dashboard.setting
      },
      set(n) {
        this.$store.state.dashboard.setting = n
      }
    },
    color_card() {
      return getProp(this.setting, ['color', 'memberCard'])
    },
    color_main() {
      return getProp(this.setting, ['color', 'barChart'])
    },
    dateRange() {
      return {
        start: getProp(this.setting, ['dateRange', 'start']),
        end: getProp(this.setting, ['dateRange', 'end'])
      }
    },
    memberSetting() {
      return {
        setting: getProp(this.setting, ['memberCard']),
        data: {
          main: [{ value: 1 }]
        }
      }
    }
  },
  created() {
    // this.flexible.init()
  },
  mounted() {
    setTimeout(() => {
      this.init()
    }, 2000)
  },
  beforeDestroy() {
    // this.flexible.terminate()
    window.removeEventListener('resize', this.resize)
  },
  methods: {
    timeZone,
    modify,
    requireUpdateItems(items) {
      this.$refs.warningLine.updateItems(items)
    },
    requestFile(file) {
      return requestFile('/dataview', file).then(data => {
        return download(data.file.id)
      })
    },
    async init() {
      this.$nextTick(() => {
        window.addEventListener('resize', this.resize)
        // this.$refs.echartGeoDriver.refresh()
      })
    },
    settingUpdated() {
      const dataDriver = this.$refs.dataDriver
      if (dataDriver) {
        dataDriver.refresh().then(() => {
          this.refresh(true)
        })
      }
    },
    refresh() {
      this.chartsDoAction(c => {
        if (c._data && c._data.chart && c.refresh) {
          c.refresh()
        }
      })
    },
    resize() {
      this.chartsDoAction(c => {
        if (!c || !c.data) return
        const data = c._data
        if (data.chart) data.chart.resize()
      })
    },
    chartsDoAction(method) {
      const lastUpdate = new Date()
      this.lastUpdate = lastUpdate
      setTimeout(() => {
        if (this.lastUpdate !== lastUpdate) return
        for (var component of Object.keys(this.$refs)) {
          const c = this.$refs[component]
          method(c)
        }
      }, 500)
    }
  }
}
</script>

<style lang="scss" >
@import './style/index.scss';
</style>

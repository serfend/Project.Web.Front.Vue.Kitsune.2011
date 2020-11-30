<template>
  <el-dialog :visible="inner_visible" @close="close">
    <div>详情</div>
  </el-dialog>
</template>

<script>
export default {
  name: 'ConnectionDetail',
  props: {
    visible: {
      type: Boolean,
      default: false
    }
  },
  data: () => ({
    inner_visible: false
  }),
  watch: {
    visible: {
      handler(val) {
        this.inner_visible = val
      }, immediate: true
    }
  },
  methods: {
    close() {
      console.log('update false')
      this.$emit('update:visible', false)
    }
  }
}
</script>

<style>
</style>

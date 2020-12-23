<template>
  <el-tree :data="menus" :props="defaultProps"
           node-key="catId"
           ref="menuTree"
           @node-click="nodeClick"
  >
  </el-tree>
</template>

<script>
  export default {
    name: 'category',
    data () {
      return {
        menus: [],
        expandedKey: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        },
      }
    },
    methods: {
      getMenus () {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get'
        }).then(({data}) => {
          this.menus = data.data
        })
      },
      //树节点单击事件
      nodeClick(data,node,component){
        console.log('子组件category的节点被点击', data, node, component)
        //向父组件发送事件
        this.$emit('tree-node-click', data, node, component)
      }
    },
    created () {
      this.getMenus()
    }
  }
</script>

<style scoped>

</style>

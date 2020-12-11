<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button @click="batchSave">批量保存</el-button>
    <el-button @click="batchDelete">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="expandedKey"
             :expand-on-click-node="false"
             :draggable="draggable"
             :allow-drop="allowDrop"
             @node-drop="handleDrop"
             ref="menuTree"
    >
     <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level<=2"
            type="text"
            size="mini"
            @click="() => append(data)">
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" :close-on-click-modal="false" width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
    <el-button @click="dialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="submitData">确 定</el-button>
  </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    name: 'category',
    data () {
      return {
        category: {
          catId: null,
          name: '',
          parentCid: 0,
          catLevel: 0,
          showStatus: 1,
          sort: 0,
          icon: '',
          productUnit: ''
        },
        menus: [],
        expandedKey: [],
        updateNodes: [],
        dialogVisible: false,
        dialogType: '',
        dialogTitle: '',
        defaultProps: {
          children: 'children',
          label: 'name'
        },
        maxLevel: 1,
        draggable: false,
        pCId: []

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
      edit (data) {
        console.log('edit', data)
        this.dialogVisible = true
        this.dialogType = 'edit'
        this.dialogTitle = '修改分类'
        //发送请求获取当前节点最新的数据
        this.$http({
          url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
          method: 'get'
        }).then(({data}) => {
          //请求成功
          this.category.name = data.data.name
          this.category.catId = data.data.catId
          this.category.icon = data.data.icon
          this.category.productUnit = data.data.productUnit
          this.category.parentCid = data.data.parentCid
        })
      },
      append (data) {
        this.dialogVisible = true
        this.dialogType = 'add'
        this.dialogTitle = '添加分类'
        this.category.name = ''
        this.category.catId = null
        this.category.parentCid = null
        this.category.productUnit = ''
        this.category.icon = ''
        this.category.parentCid = data.catId
        this.category.catLevel = data.catLevel * 1 + 1
      },
      submitData () {
        if (this.dialogType == 'add') {
          this.addCategory()
        }
        if (this.dialogType == 'edit') {
          this.editCategory()
        }
      },
      //修改三级分类数据
      editCategory () {
        //只需要修改部分数据
        //将category中的数据结构出来
        let {catId, name, icon, productUnit} = this.category
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData({catId, name, icon, productUnit}, false)
        }).then(({}) => {
          this.dialogVisible = false
          this.$message({
            type: 'success',
            message: '菜单修改成功'
          })
          this.getMenus()
          //设置默认展开的菜单
          this.expandedKey = [this.category.parentCid]
        })
      },
      //添加三级分类数据
      addCategory () {
        this.$http({
          url: this.$http.adornUrl('/product/category/save'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(({}) => {
          this.dialogVisible = false
          this.$message({
            type: 'success',
            message: '菜单保存成功'
          })
          this.getMenus()
          //设置默认展开的菜单
          this.expandedKey = [this.category.parentCid]
        })
      },
      remove (node, data) {
        let ids = [data.catId]
        this.$confirm(`是否删除${data.name}菜单?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({}) => {
            this.$message({
              type: 'success',
              message: '菜单删除成功'
            })
            this.getMenus()
            //设置默认展开的菜单
            this.expandedKey = [node.parent.data.catId]
          })
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
      },
      //判断节点是否可拖拽到指定位置
      allowDrop (draggingNode, dropNode, type) {
        this.maxLevel = 1
        // 1、被拖动的当前节点以及所在的父节点层数不能大于3
        // 1)、被拖动当前节点的总层数
        this.countNodeLevel(draggingNode)
        //当前正在拖动的节点+父节点的深度 不大于3即可
        //算出拖拽节点的真实深度
        //当批量修改时，数据库的值没有实施更新 应该改为tree的level
        // let deep = (this.maxLevel - draggingNode.data.catLevel) + 1
        let deep = this.maxLevel - draggingNode.level + 1 //绝对值
        console.log(deep)
        if (type == 'inner') {
          return (deep + dropNode.level) <= 3
        } else {
          return (deep + dropNode.parent.level) <= 3
        }
      },
      countNodeLevel (node) {
        //找到所有子节点，求出最大深度
        //注意maxLevel 是拖拽的子节点的最大深度
        if (node.childNodes != null && node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            if (node.childNodes[i].level > this.maxLevel) {
              this.maxLevel = node.childNodes[i].level
            }
            this.countNodeLevel(node.childNodes[i])
          }
        }
      },
      //拖拽成功完成时触发的事件
      handleDrop (draggingNode, dropNode, dropType, ev) {
        console.log('handleDrop: ', draggingNode, dropNode, dropType, ev)
        //1、当前节点的最小父节点id
        let pCId = 0
        let siblingsNode = null
        if (dropType == 'before' || dropType == 'after') {
          pCId = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId
          siblingsNode = dropNode.parent.childNodes
        } else {
          pCId = dropNode.data.catId
          siblingsNode = dropNode.childNodes
        }
        this.pCId.push(pCId)
        console.log(this.pCId)
        //2、当前拖拽节点的最新排序
        for (let i = 0; i < siblingsNode.length; i++) {
          if (siblingsNode[i].data.catId == draggingNode.data.catId) {
            //如果遍历的是当前正在拖拽的节点
            let catLevel = draggingNode.level
            if (siblingsNode[i].level != draggingNode.level) {
              //当前层级发生变化
              catLevel = siblingsNode[i].level
              //子节点层级必然也改变了
              this.updateChildNodeLevel(siblingsNode[i])

            }
            this.updateNodes.push({
              catId: siblingsNode[i].data.catId,
              parentCid: pCId,
              sort: i,
              catLevel: catLevel
            })
          } else {
            this.updateNodes.push({catId: siblingsNode[i].data.catId, sort: i})
          }

        }
        //3、当前拖拽节点的最新层级
        //请求更新
        console.log(this.updateNodes)
        // this.$http({
        //   url: this.$http.adornUrl('/product/category/update/sort'),
        //   method: 'post',
        //   data: this.$http.adornData(this.updateNodes, false)
        // }).then(({}) => {
        //   this.$message({
        //     type: 'success',
        //     message: '菜单顺序修改成功'
        //   })
        //   this.getMenus()
        //   //设置默认展开的菜单
        //   this.expandedKey = [pCId]
        // })
      },
      //修改子节点层级
      updateChildNodeLevel (node) {
        if (node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            let cNode = node.childNodes[i].data
            this.updateNodes.push({catId: cNode.catId, catLevel: node.childNodes[i].level})
            this.updateChildNodeLevel(node.childNodes[i])
          }
        }
      },
      //批量拖拽菜单
      batchSave () {
        this.$http({
          url: this.$http.adornUrl('/product/category/update/sort'),
          method: 'post',
          data: this.$http.adornData(this.updateNodes, false)
        }).then(({}) => {
          this.$message({
            type: 'success',
            message: '菜单顺序修改成功'
          })
          this.getMenus()
          //设置默认展开的菜单
          this.expandedKey = this.pCId
          this.updateNodes = []
          this.maxLevel = 1
          this.pCId = []
        })
      },

      //批量删除
      batchDelete () {
        let catIds = []
        let catNames = []
        let checkedNodes = this.$refs.menuTree.getCheckedNodes()
        console.log('被选中的nodes', checkedNodes)
        for (let i = 0; i < checkedNodes.length; i++) {
          catIds.push(checkedNodes[i].catId)
          catNames.push(checkedNodes[i].name)
        }
        this.$confirm(`是否批量删除【${catNames}】菜单?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({}) => {
            this.$message({
              type: 'success',
              message: '批量删除成功'
            })
            this.getMenus()
            //设置默认展开的菜单
            this.expandedKey = [node.parent.data.catId]
          })
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
      }
    },
    created () {
      this.getMenus()
    }
  }
</script>

<style scoped>

</style>

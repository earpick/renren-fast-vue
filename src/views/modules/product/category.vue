<template>
  <div>
    <el-tree :data="menu" :props="defaultProps"
             :expand-on-click-node="false"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="exPanDedKey"
             draggable
             :allow-drop="allowDrop">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <=2"
            type="text"
            size="mini"
            @click="() => append(data)">
            Append
          </el-button>
          <el-button
            v-if="node.childNodes.length === 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            Delete
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false">
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
      maxLevel: 0,
      title: '',
      dialogType: '',
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: '',
        icon: '',
        catId: null
      },
      dialogVisible: false,
      menu: [],
      exPanDedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  methods: {
    // 拖拽实现改变分类层级
    allowDrop (draggingNode, dropNode, type) {
      console.log('draggingNode:', draggingNode, 'dropNode:', dropNode, 'type:', type)
      // 1. 被拖到的当前节点以及所在的父节点总层数不大于3
      // 1.1） 确定被拖拽的当前节点总层数
      this.countNodeLevel(draggingNode.data)
      // 当前正在拖动的节点 + 父节点所在的最大层级（深度）不大于3即可
      let deep = this.maxLevel - draggingNode.data.catLevel + 1
      console.log('深度:', deep)

      if (type === 'inner') {
        return (deep + dropNode.data.catLevel) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    // 找到了拖拽结点的最大层级(深度)
    countNodeLevel (node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel
          }
          // 子字节下还有子节点，再次执行求出最大层级方法
          this.countNodeLevel(node.children[i])
        }
      }
    },
    // 添加分类
    append (data) { // data为添加的节点数据
      console.log('append', data)
      this.title = '添加分类'
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.catId = null
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },
    // 提交添加的分类数据
    addCategory () {
      console.log('添加了分类数据', this.category)
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '菜单保存成功!'
        })
        // 关闭对话框
        this.dialogVisible = false
        // 刷新菜单
        this.getDataList()
        // 设置需要默认展开的菜单
        this.exPanDedKey = [this.category.parentCid]
      })
    },
    // 删除分类
    remove (node, data) { // note：被删除节点的状态， data：从数据库中获取到的节点数据
      var ids = [data.catId]
      // 确认消息弹框
      this.$confirm(`此操作将永久删除[${data.name}]菜单, 是否继续?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          console.log('菜单数据删除成功！')
          // 刷新菜单
          this.getDataList()
          // 设置需要默认展开的菜单
          this.exPanDedKey = [node.parent.data.catId]
        })
        console.log('remove', node, data)
        this.$message({
          type: 'success',
          message: '删除成功!'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },
    // 修改分类
    edit (data) {
      console.log('要修改的数据', data)
      this.title = '修改分类'
      this.dialogType = 'edit'
      this.dialogVisible = true
      // 发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 请求成功
        console.log('要回显的数据', data)
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
        this.category.catLevel = data.data.catLevel
        this.category.sort = data.data.sort
        this.category.showStatus = data.data.showStatus
      })
    },
    // 提交添加的分类数据方法
    editCategory () {
      var {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单修改成功',
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false
        // 刷新出新的菜单
        this.getDataList()
        // 设置需要默认展开的菜单
        this.exPanDedKey = [this.category.parentCid]
      })
    },
    // 判断执行 提交修改分类数据方法/提交添加的分类数据方法
    submitData () {
      if (this.dialogType === 'add') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    // 获取数据列表
    getDataList () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log('成功获取了菜单数据...', data)
        this.menu = data.data
      })
    }
  },
  // 实例创建完成后执行此函数中的代码
  created () {
    this.getDataList()
  }
}
</script>

<style scoped>

</style>

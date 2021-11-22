<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽功能" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menu"
             :props="defaultProps"
             :expand-on-click-node="false"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="exPanDedKey"
             :allow-drop="allowDrop"
             :draggable="draggable"
             @node-drop="handleDrop"
             ref="menuTree">
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
      draggable: false,
      pCid: [],
      updateNodes: [],
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
    // 批量删除
    batchDelete () {
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({ data }) => {
            this.$message({
              message: '菜单批量删除成功',
              type: 'success'
            })
            this.getDataList()
          })
        })
        .catch(() => {})
    },
    // 批量向数据库提交拖拽后需要修改的数据
    batchSave () {
      // 把封装了需要修改数据的数组updateNodes 通过网络请求发送到服务器
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.$message({
            message: '菜单顺序等修改成功',
            type: 'success'
          })
        }
        // 刷新出新的菜单
        this.getDataList()
        // 设置需要默认展开的菜单
        this.exPanDedKey = this.pCid
        // 清空向服务器提交更新的数据的数组
        this.updateNodes = []
        // 清空拖拽节点的最大层级
        this.maxLevel = 0
      })
    },
    // 拖拽成功完成时触发的事件
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('draggingNode: ', draggingNode, 'dropNode: ', dropNode, dropType)
      // 1. 当前节点最新的父节点
      // eslint-disable-next-line no-unused-vars
      let pCid = 0
      // eslint-disable-next-line no-unused-vars
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }

      // 1.1） 将多个被拖拽菜单的新父节点id放到pCid数组中
      this.pCid.push(pCid)

      // 2. 当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点
          // eslint-disable-next-line no-unused-vars
          let catLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 修改他子节点的层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid
          })
        } else {
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i
          })
        }
      }

      // 3. 当前拖拽节点的最新层级
      console.log('updateNodes:', this.updateNodes)
    },
    // 修改被拖拽节点的子节点的层级
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    // 拖拽实现改变分类层级
    allowDrop (draggingNode, dropNode, type) {
      console.log('draggingNode:', draggingNode, 'dropNode:', dropNode, 'type:', type)
      // 1. 被拖到的当前节点以及所在的父节点总层数不大于3
      // 1.1） 确定被拖拽的当前节点总层数
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点 + 父节点所在的最大层级（深度）不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度:', deep)
      console.log('记录拖拽节点的最大层级:', this.maxLevel)

      if (type === 'inner') {
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    // 找到了拖拽结点的最大层级(深度)
    countNodeLevel (node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        console.log('执行了countNodeLevel')
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          // 子字节下还有子节点，再次执行求出最大层级方法
          this.countNodeLevel(node.childNodes[i])
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
          message: '删除菜单成功!'
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

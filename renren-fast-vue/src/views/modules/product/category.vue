<template>
  <div>
<!--  开关组件  -->
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave()">批量保存</el-button>
    <el-button type="danger" @click="bacthDelete()">危险按钮</el-button>
<!-- 树形组件   -->
    <el-tree ref="menuTree" @node-drop="handleDrop" :draggable="draggable" :allow-drop="allowDrop" :default-expanded-keys="expandedKey" show-checkbox :data="menu" node-key="catId" :expand-on-click-node="false" :props="defaultProps">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="data.catLevel <= 2"
            type="text"
            size="mini"
            @click="() => append(data)">
            Append
          </el-button>
          <el-button v-if="node.childNodes.length == 0"
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
<!-- 对话框   -->
    <el-dialog
      :close-on-click-modal="false"
      :title="title"
      :visible.sync="dialogVisible"
      width="30%">
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
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
// import引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    // 这里存放数据
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: '',
      dialogType: '',
      category: { catId: null, name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: '', productUnit: '' },
      dialogVisible: false,
      expandedKey: [],
      menu: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
// 计算属性 类似于data概念
  computed: {},
// 监控data中的数据变化
  watch: {},
// 方法集合
  methods: {
    // 批量删除
    bacthDelete () {
      // eslint-disable-next-line no-unused-vars
      const checkedNodes = this.$refs.menuTree.getCheckedNodes()
      const catIds = []
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否评论量删除${catIds}菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            message: '菜单批量删除成功',
            type: 'success'
          })
          // 菜单刷新
          this.getMenus()
        }).catch(error => {
          console.error(error)
        })
      }).catch(() => {
      })
    },
    // 批量拖拽后,统一批量保存
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: '批量保存成功',
          type: 'success'
        })
        // 刷新菜单
        this.getMenus()
        // 展开移动的节点
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.maxLevel = 0
        this.pCid = 0
      }).catch(error => {
        console.error(error)
      })
    },
    // 拖拽成功进入的方法
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)
      // 1、当前节点最新的父节点id
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)

      // 2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        // 如果遍历的是当前正在拖拽的节点
        if (siblings[i].data.catId === draggingNode.data.catId) {
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
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
        }
      }

      // 3、当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
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
    // 拖拽时进入的节点
    allowDrop (draggingNode, dropNode, type) {
      // 1、被拖动的当前节点以及所在的父节点总层数不能大于3
      // 1）、被拖动的当前节点总层数
      console.log('allowDrop:', draggingNode, dropNode, type)
      //
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      const deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度：', deep)

      //   this.maxLevel
      if (type === 'inner') {
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel (node) {
      // 找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    // 提交分类
    submitData () {
      if (this.dialogType === 'add') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    // 添加分类请求
    addCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          message: `名称为【${this.category.name}】的菜单保存成功`,
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false
        // 刷新菜单列表
        this.getMenus()
        // 默认展开添加分类的父tree节点
        this.expandedKey = [this.category.parentCid]
      }).catch(error => {
        console.error(error)
      })
    },
    // 修改分类请求
    editCategory () {
      const { catId, name, icon, productUnit } = this.category
      var data = { catId, name, icon, productUnit }
      console.log('编辑菜单：', data)
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData(data, false)
      }).then(({ data }) => {
        this.$message({
          message: `菜单修改成功`,
          type: 'success'
        })
        // 关闭对话框
        this.dialogVisible = false
        // 刷新菜单列表
        this.getMenus()
        // 默认展开添加分类的父tree节点
        this.expandedKey = [this.category.parentCid]
      }).catch(error => {
        console.error(error)
      })
    },
    // 修改分类
    edit (data) {
      this.title = '编辑菜单'
      this.dialogType = 'edit'
      this.dialogVisible = true

      // 发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({ data }) => {
        console.log(data)
        this.category.catId = data.data.catId
        this.category.name = data.data.name
        this.category.icon = data.data.icon
        this.category.parentCid = data.data.parentCid
        this.category.productUnit = data.data.productUnit
      }).catch(error => {
        console.error(error)
      })
    },
    // 添加分类
    append (data) {
      console.log(data)
      this.title = '添加菜单'
      this.dialogType = 'add'
      this.dialogVisible = true

      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.catId = null
      this.category.name = null
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },

    // 删除分类
    remove (node, data) {
      this.$confirm(`是否删除${data.name}菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        const ids = [data.catId]
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          this.$message({
            message: '菜单删除成功',
            type: 'success'
          })
          // 刷新出新的菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [node.parent.data.catId]
        })
      }).catch(() => {

      })
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        console.log(data)
        this.menu = data.data
      }).catch(error => {
        console.error(error)
      })
    }
  },
// 生命周期 - 创建完成（可以访问当前this实例）
  created () {
    this.getMenus()
  },
// 生命周期 - 挂载完成（可以访问DOM元素）
  mounted () {

  },
  beforeCreate () {}, // 生命周期 - 创建之前
  beforeMount () {}, // 生命周期 - 挂载之前
  beforeUpdate () {}, // 生命周期 - 更新之前
  updated () {}, // 生命周期 - 更新之后
  beforeDestroy () {}, // 生命周期 - 销毁之前
  destroyed () {}, // 生命周期 - 销毁完成
  activated () {} // 如果页面有keep-alive缓存功能，这个函数会触发
}
</script>

<style scoped>

</style>

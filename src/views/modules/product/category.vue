<template>
  <div>
    <el-button
      type="danger"
      @click="batchRemove"
    >批量删除</el-button>
    <el-tree
      :data='categories'
      node-key="catId"
      show-checkbox
      :props='defaultProps'
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKeys"
      ref="categoryTree"
    >
      <span slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)"
          >
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      title="新增目录"
      :visible.sync="dialogFormVisible"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input
            v-model="category.name"
            autocomplete="off"
          ></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input
            v-model="category.icon"
            autocomplete="off"
          ></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <div
        slot="footer"
        class="dialog-footer"
      >
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button
          type="primary"
          @click="submitData"
        >确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data () {
    // 这里存放数据
    return {
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
      dialogType: '', // edit, append
      categories: [],
      expandedKeys: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      dialogFormVisible: false
    }
  },
  // 计算属性 类似于 data 概念
  computed: {},
  // 监控 data 中的数据变化
  watch: {},
  // 方法集合
  methods: {
    getDataList () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        this.categories = data.categories
        console.log(this.categories)
      })
    },
    submitData () {
      if (this.dialogType === 'append') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },

    append (data) {
      this.dialogType = 'append'
      this.dialogFormVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.catId = null
      this.category.name = ''
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
    },
    addCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          message: '操作成功',
          type: 'success'
        })
        this.dialogFormVisible = false
        this.getDataList()
        // 自动展开被删结点的父节点
        this.expandedKeys = [this.category.parentCid]
      })
    },
    edit (data) {
      this.dialogType = 'edit'
      this.dialogFormVisible = true
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({ data }) => {
        this.category.name = data.category.name
        this.category.catId = data.category.catId
        this.category.icon = data.category.icon
        this.category.productUnit = data.category.productUnit
        this.category.parentCid = data.category.parentCid
        this.category.catLevel = data.category.catLevel
        this.category.sort = data.category.sort
        this.category.showStatus = data.category.showStatus
      })
    },
    editCategory () {
      var { catId, name, icon, productUnit } = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'put',
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      }).then(({ data }) => {
        this.$message({
          message: '菜单修改成功',
          type: 'success'
        })
        this.dialogFormVisible = false
        this.getDataList()
        this.expandedKeys = [this.category.parentCid]
      })
    },
    remove (node, data) {
      console.log('data', data)
      var ids = [data.catId]

      this.$confirm(`确定对[${data.name}]进行删除操作?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'delete',
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: '操作成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getDataList()
                // 自动展开被删结点的父节点
                this.expandedKeys = [node.parent.data.catId]
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => { })
    },
    batchRemove () {
      let catIds = []
      let checkedNodes = this.$refs.categoryTree.getCheckedNodes()
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'delete',
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          if (data && data.code === 0) {
            this.$message({
              message: '菜单批量删除成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getDataList()
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => { })
    }
  },
  // 生命周期 - 创建完成（可以访问当前 this 实例）
  created () {
    this.getDataList()
  },
  // 生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted () { },
  beforeCreate () { }, // 生命周期 - 创建之前
  beforeMount () { }, // 生命周期 - 挂载之前
  beforeUpdate () { }, // 生命周期 - 更新之前
  updated () { }, // 生命周期 - 更新之后
  beforeDestroy () { }, // 生命周期 - 销毁之前
  destroyed () { }, // 生命周期 - 销毁完成
  activated () { } // 如果页面有 keep-alive 缓存功能，这个函数会触发
}
</script>
<style lang='scss' scoped>
</style>

<template>
  <div class="app-container" ref="appContainer" v-resize="mixinResize">
    <div class="filter-container" ref="filterContainer">
      <el-form :model="listQuery" class="filter-container-form" size="mini" :inline="true">
        <el-form-item label="实体视图类型">
          <el-select v-model="listQuery.type">
            <el-option label="所有" value=""></el-option>
            <el-option v-for="item in entityViewTypes" :key="item.type" :label="item.type" :value="item.type"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="getList()">查询</el-button>
          <el-button type="primary" @click="openDialog('add')">添加</el-button>
        </el-form-item>
      </el-form>
    </div>
    <el-table
      :data="list"
      v-loading="loading"
      :default-sort="{prop: 'createdTime', order: 'descending'}"
      @sort-change="sortChange"
      size="mini"
      :height="mixinHeight"
      :class="['configurationTable', {afterRenderClass: mixinShowAfterRenderClass}]"
      @cell-click="cellClick">
      <el-table-column
        type="selection"
        width="90">
      </el-table-column>
      <el-table-column
        v-for="item in listTitle"
        :key="item.label"
        :min-width="item.width"
        :label="item.label"
        :sortable="item.sortable"
        :prop="item.property"
        :sort-orders="['ascending', 'descending']"
        align="center"
        show-overflow-tooltip>
        <template slot-scope="scope">
          <div v-if="item.property === 'btn'" class="center">
            <el-button v-if="scope.row.icon.public" type="primary" size="mini" @click="open(scope.row, 'public')">资产设为公开</el-button>
            <el-button v-if="scope.row.icon.allocation" type="primary" size="mini" @click="openDialog('allocation', scope.row)">分配给客户</el-button>
            <el-button v-if="scope.row.icon.cancelAllocation" type="primary" size="mini" @click="open(scope.row, 'allocation')">取消分配客户</el-button>
            <el-button v-if="scope.row.icon.provide" type="primary" size="mini" @click="open(scope.row, 'private')">资产设为私有</el-button>
            <el-button type="primary" size="mini" @click="open(scope.row, 'delete')">删除</el-button>
          </div>
          <span v-else>{{ scope.row[item.property] }}</span>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      class="pagination-container"
      ref="pagination"
      @size-change="getList()"
      @current-change="getList()"
      :current-page.sync="page"
      background
      :page-sizes="sizes"
      :page-size.sync="limit"
      layout="total, sizes, prev, pager, next, jumper"
      :total="total"
    ></el-pagination>
    <icloud-dialog
      :title="title"
      :visible.sync="visible">
      <form-info v-if="type === 'add'" ref="form" @submit="save"></form-info>
      <el-form v-else ref="form" :model="allocationForm" :rules="allocationRules">
        <el-form-item label="客户" prop="id">
          <el-select v-model="allocationForm.id">
            <el-option v-for="item in customerList" :key="item.id.id" :label="item.name" :value="item.id.id"></el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <div slot="footer" class="icloud-dialog-footer">
        <wx-button type="primary" @click="submit">确定</wx-button>
        <wx-button @click="visible = false">取消</wx-button>
      </div>
    </icloud-dialog>
  </div>
</template>

<script>
import { page, resize } from '@/mixins'
import { getDate } from '@/utils'
import { FormInfo } from './components'
export default {
  mixins: [page, resize],
  components: { FormInfo },
  data () {
    return {
      listQuery: {
        type: '',
        sortOrder: 'DESC'
      },
      entityViewTypes: [],
      list: [],
      listTitle: [
        { property: 'createdTime', label: '创建时间', width: 180, sortable: true },
        { property: 'name', label: '名称', width: 150 },
        { property: 'type', label: '实体视图类型', width: 150 },
        { property: 'customerTitle', label: '客户', width: 150 },
        { property: 'customerIsPublic', label: 'Public', width: 150 },
        { property: 'btn', label: '操作', width: 250 }
      ],
      visible: false,
      type: '',
      title: '',
      allocationForm: {
        id: ''
      },
      allocationRules: {
        id: [{ required: true, message: '客户不能为空', trigget: 'change' }]
      },
      customerList: [],
      typeList: [],
      info: null
    }
  },
  methods: {
    sortChange ({ order }) {
      const isDesc = order === 'descending'
      this.listQuery.sortOrder = isDesc ? 'DESC' : 'ASC'
      this.getList()
    },
    save () {
      this.visible = false
      this.init()
    },
    open (row, type) {
      const info = {
        public: {
          title: `你确定你想创建公开'${row.name}'实体视图?`,
          msg: '确认后，实体视图 及其所有数据将被公开并被他人访问'
        },
        allocation: {
          title: `你确定要取消对'${row.name}'实体视图的分配吗?`,
          msg: '确认后，实体视图将未分配，客户无法访问'
        },
        private: {
          title: `你确定你想创建私有'${row.name}'实体视图?`,
          msg: '确认后，实体视图及其所有数据将被私有化，无法被他人访问'
        },
        delete: {
          title: `确定要删除实体视图'${row.name}'?`,
          msg: '小心！确认后实体视图及其所有相关数据将不可恢复'
        }
      }
      this.$confirm(info[type].msg, info[type].title, {
        confirmButtonText: '是',
        cancelButtonText: '否',
        type: 'warning'
      }).then(async _ => {
        let apiName = ''
        if (type === 'public') {
          apiName = 'postCustomerPublicEntityView'
        } else if (type === 'allocation' || type === 'private') {
          apiName = 'deleteCustomerEntityView'
        } else if (type === 'delete') {
          apiName = 'deleteEntityView'
        }
        const res = await this.$api[apiName](row.id.id)
        if (res.status === 200) {
          this.$message.success('操作成功')
          this.getList()
        }
      }).catch(() => {})
    },
    submit () {
      if (this.type === 'add') {
        this.$refs.form.submit()
      } else {
        this.$refs.form.validate(async valid => {
          if (!valid) return false
          const res = await this.$api.postCustomerEntityView(this.allocationForm.id, this.info.id.id)
          if (res.status === 200) {
            this.$message.success('分配成功')
            this.visible = false
            this.init()
          }
        })
      }
    },
    openDialog (type, info) {
      this.visible = true
      this.type = type
      this.info = info
      this.title = type === 'add' ? '添加实体视图' : '将实体视图分配给客户'
    },
    cellClick (row, column) {
      if (column.label !== '操作') {
        this.$router.push({ path: `/entity-views/${row.id.id}`, query: { title: row.name } })
      }
    },
    async getCustomersList () {
      const res = await this.$api.getCustomersList({
        pageSize: 999999,
        page: 0,
        sortProperty: 'title',
        sortOrder: 'ASC'
      })
      this.customerList = res.data.data
    },
    async getList () {
      this.loading = true
      try {
        const res = await this.$api.getEntityViewList(Object.assign({
          page: this.page - 1,
          pageSize: this.limit,
          sortProperty: 'createdTime'
        }, this.listQuery))
        this.list = res.data.data.map(ele => {
          const { customerIsPublic, customerTitle } = ele
          return {
            ...ele,
            createdTime: getDate({ timestamp: ele.createdTime }),
            icon: {
              public: !customerIsPublic && !customerTitle,
              allocation: !customerTitle,
              cancelAllocation: !customerIsPublic && customerTitle,
              provide: customerIsPublic
            }
          }
        })
        this.total = res.data.totalElements
      } catch (error) {
        this.$message.error(error.response.data.message)
      }
      this.loading = false
    },
    init () {
      this.getList()
      this.getCustomersList()
    }
  },
  created () {
    this.init()
  }
}
</script>

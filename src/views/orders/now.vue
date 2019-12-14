<template>
  <div class="app-container">
    <div class="filter-container">
      <el-select v-model="payWay" placeholder="付款方式">
        <el-option key="現金" label="現金" value="現金" />
        <el-option key="Line Pay" label="Line Pay" value="Line Pay" />
        <el-option key="Google Pay" label="Google Pay" value="Google Pay" />
        <el-option key="Apple Pay" label="Apple Pay" value="Apple Pay" />
      </el-select>

      <el-input v-model="listQuery.title" placeholder="訂單編號/訂單狀態" style="width: 300px;" @keyup.enter.native="handleFilter" />
      <el-button v-waves type="primary" icon="el-icon-search" @click="handleFilter">
        搜尋
      </el-button>
    </div>

    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
      @sort-change="sortChange"
    >
      <el-table-column label="訂單編號" prop="id" sortable="custom" align="center" min-width="80px" :class-name="getSortClass('id')" />
      <el-table-column label="訂單狀態" min-width="90px" align="center" sortable="custom">
        <template slot-scope="{row}">
          <el-tag v-if="isDanger(row.id)" type="success">
            {{ row.status }}
          </el-tag>
          <el-tag v-else type="danger">已退單</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="取餐時間" prop="getTime" min-width="90px" align="center" />
      <el-table-column label="品項" min-width="130px" align="center">
        <template slot-scope="scope">
          <div
            v-for="prod in formatProds(scope.row.products)"
            :key="prod.name"
            style="text-align:left;"
          >
            {{ prod.name }} X {{ prod.count }}
          </div>
        </template>
      </el-table-column>
      <el-table-column label="付款方式" prop="pay" align="center" width="110px" />
      <el-table-column label="總金額" align="center" width="100">
        <template slot-scope="scope">
          $ {{ price(scope.row.products) }}
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" min-width="150" class-name="small-padding fixed-width">
        <div style="display: flex;">
          <el-button type="primary" disabled>叫號</el-button>
          <el-button type="success" disabled>完成</el-button>
          <el-button type="danger" disabled>退單</el-button>
        </div>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" :rules="rules" :model="temp" label-position="left" label-width="70px" style="width: 400px; margin-left:50px;">
        <el-form-item label="Type" prop="type">
          <el-select v-model="temp.type" class="filter-item" placeholder="Please select">
            <el-option v-for="item in calendarTypeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
          </el-select>
        </el-form-item>
        <el-form-item label="Date" prop="timestamp">
          <el-date-picker v-model="temp.timestamp" type="datetime" placeholder="Please pick a date" />
        </el-form-item>
        <el-form-item label="Title" prop="title">
          <el-input v-model="temp.title" />
        </el-form-item>
        <el-form-item label="Status">
          <el-select v-model="temp.status" class="filter-item" placeholder="Please select">
            <el-option v-for="item in statusOptions" :key="item" :label="item" :value="item" />
          </el-select>
        </el-form-item>
        <el-form-item label="Imp">
          <el-rate v-model="temp.importance" :colors="['#99A9BF', '#F7BA2A', '#FF9900']" :max="3" style="margin-top:8px;" />
        </el-form-item>
        <el-form-item label="Remark">
          <el-input v-model="temp.remark" :autosize="{ minRows: 2, maxRows: 4}" type="textarea" placeholder="Please input" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          Cancel
        </el-button>
        <el-button type="primary" @click="dialogStatus==='create'?createData():updateData()">
          Confirm
        </el-button>
      </div>
    </el-dialog>

    <el-dialog :visible.sync="dialogPvVisible" title="Reading statistics">
      <el-table :data="pvData" border fit highlight-current-row style="width: 100%">
        <el-table-column prop="key" label="Channel" />
        <el-table-column prop="pv" label="Pv" />
      </el-table>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="dialogPvVisible = false">Confirm</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // secondary package based on el-pagination

const prodList = JSON.parse(`[{"id":"58","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1},"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"57","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"56","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"55","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"54","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"53","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"排入現場順序"},{"id":"52","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"51","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"排入現場順序"},{"id":"50","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"排入現場順序"},{"id":"49","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"48","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"47","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"46","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"排入現場順序"},{"id":"45","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"44","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"排入現場順序"},{"id":"43","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"42","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"41","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"40","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"39","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"排入現場順序"},{"id":"38","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"37","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"36","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"35","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"34","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"33","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"32","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"31","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"30","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"29","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"排入現場順序"},{"id":"28","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"27","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"排入現場順序"},{"id":"26","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"排入現場順序"},{"id":"25","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"排入現場順序"},{"id":"24","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"23","products":{"雙拼":{"name":"雙拼","price":75,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"22","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"21","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"排入現場順序"},{"id":"20","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"19","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"18","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"排入現場順序"},{"id":"17","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"16","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"15","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"排入現場順序"},{"id":"14","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1},"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"13","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"12","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"11","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"10","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"09","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"08","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"07","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"06","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"排入現場順序"},{"id":"05","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"04","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"},{"id":"03","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"排入現場順序"},{"id":"02","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"排入現場順序"}]`)

const calendarTypeOptions = [
  { key: 'CN', display_name: 'China' },
  { key: 'US', display_name: 'USA' },
  { key: 'JP', display_name: 'Japan' },
  { key: 'EU', display_name: 'Eurozone' }
]

// arr to obj, such as { CN : "China", US : "USA" }
const calendarTypeKeyValue = calendarTypeOptions.reduce((acc, cur) => {
  acc[cur.key] = cur.display_name
  return acc
}, {})

export default {
  name: 'Now',
  components: { Pagination },
  directives: { waves },
  filters: {
    statusFilter(status) {
      const statusMap = {
        published: 'success',
        draft: 'info',
        deleted: 'danger'
      }
      return statusMap[status]
    },
    typeFilter(type) {
      return calendarTypeKeyValue[type]
    }
  },
  data() {
    return {
      payWay: '',
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 20,
        importance: undefined,
        title: undefined,
        type: undefined,
        sort: '+id'
      },
      importanceOptions: [1, 2, 3],
      calendarTypeOptions,
      sortOptions: [{ label: 'ID Ascending', key: '+id' }, { label: 'ID Descending', key: '-id' }],
      statusOptions: ['published', 'draft', 'deleted'],
      showReviewer: false,
      temp: {
        id: undefined,
        importance: 1,
        remark: '',
        timestamp: new Date(),
        title: '',
        type: '',
        status: 'published'
      },
      dialogFormVisible: false,
      dialogStatus: '',
      textMap: {
        update: 'Edit',
        create: 'Create'
      },
      dialogPvVisible: false,
      pvData: [],
      rules: {
        type: [{ required: true, message: 'type is required', trigger: 'change' }],
        timestamp: [{ type: 'date', required: true, message: 'timestamp is required', trigger: 'change' }],
        title: [{ required: true, message: 'title is required', trigger: 'blur' }]
      },
      downloadLoading: false
    }
  },
  created() {
    this.getList()
  },
  methods: {
    formatProds(products) {
      const list = []
      for (const key of Object.keys(products)) {
        list.push(products[key])
      }
      return list
    },
    price(products) {
      let price = 0
      for (const key of Object.keys(products)) {
        price += products[key].count * products[key].price
      }
      return price
    },
    isDanger(id) {
      if (id === '53' || id === '46') return false
      return true
    },
    getList() {
      this.listLoading = true
      this.list = prodList
      setTimeout(() => {
        this.listLoading = false
      }, 1.5 * 1000)
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleModifyStatus(row, status) {
      this.$message({
        message: '操作Success',
        type: 'success'
      })
      row.status = status
    },
    sortChange(data) {
      const { prop, order } = data
      if (prop === 'id') {
        this.sortByID(order)
      }
    },
    sortByID(order) {
      if (order === 'ascending') {
        this.listQuery.sort = '+id'
      } else {
        this.listQuery.sort = '-id'
      }
      this.handleFilter()
    },
    formatJson(filterVal, jsonData) {
      return jsonData.map(v => filterVal.map(j => {
        if (j === 'timestamp') {
          return parseTime(v[j])
        } else {
          return v[j]
        }
      }))
    },
    getSortClass: function(key) {
      const sort = this.listQuery.sort
      return sort === `+${key}`
        ? 'ascending'
        : sort === `-${key}`
          ? 'descending'
          : ''
    }
  }
}
</script>

<template>
  <div class="app-container">
    <div class="filter-container">
      <el-date-picker
        v-model="range"
        type="daterange"
        range-separator="至"
        start-placeholder="开始日期"
        end-placeholder="结束日期"
      />

      <el-select v-model="payWay" placeholder="付款方式">
        <el-option key="現金" label="現金" value="現金" />
        <el-option key="Line Pay" label="Line Pay" value="Line Pay" />
        <el-option key="Google Pay" label="Google Pay" value="Google Pay" />
        <el-option key="Apple Pay" label="Apple Pay" value="Apple Pay" />
      </el-select>

      <el-input v-model="listQuery.title" placeholder="品項/金額" style="width: 200px;" @keyup.enter.native="handleFilter" />
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
      <el-table-column label="編號" prop="id" sortable="custom" align="center" min-width="60px" :class-name="getSortClass('id')" />
      <el-table-column label="使用者" prop="user" align="center" width="80px" />
      <el-table-column label="訂單狀態" min-width="90px" align="center" sortable="custom">
        <template slot-scope="{row}">
          <el-tag v-if="row.status === '已完成'" type="success">
            {{ row.status }}
          </el-tag>
          <el-tag v-else type="danger">已退單</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="訂單日期" prop="getTime" min-width="90px" align="center" />
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
      <el-table-column label="操作" align="center" min-width="160" class-name="small-padding fixed-width">
        <div style="display: flex;">
          <el-button type="primary" disabled>叫號</el-button>
          <el-button type="success" disabled>完成</el-button>
          <el-button type="danger" disabled>退單</el-button>
        </div>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />

  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // secondary package based on el-pagination

const prodList = JSON.parse(`[{"id":"001","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"2019-09-20","user":"劉淑雄"},{"id":"002","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"韓瑜正"},{"id":"003","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"劉靜宜"},{"id":"004","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"2019-09-20","user":"黃淑霞"},{"id":"005","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"潘乃琴"},{"id":"006","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"王瑤芬"},{"id":"007","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"許永娥"},{"id":"008","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1},"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"2019-09-20","user":"賴長愛"},{"id":"009","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"陳淑芬"},{"id":"010","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"郭子剛"},{"id":"011","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"林佩璇"},{"id":"012","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"黃伶志"},{"id":"013","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"2019-09-20","user":"蘇婉水"},{"id":"014","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1},"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"2019-09-20","user":"倪宥恒"},{"id":"015","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"周怡岑"},{"id":"016","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已退單","pay":"Google Pay","getTime":"2019-09-20","user":"周孟璇"},{"id":"017","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"王元淑"},{"id":"018","products":{"雙拼":{"name":"雙拼","price":75,"count":1},"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"吳俊洋"},{"id":"019","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已退單","pay":"Google Pay","getTime":"2019-09-20","user":"沈美華"},{"id":"020","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"李倩和"},{"id":"021","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"李明以"},{"id":"022","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已退單","pay":"Google Pay","getTime":"2019-09-20","user":"陳佳禾"},{"id":"023","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已退單","pay":"Line Pay","getTime":"2019-09-20","user":"鄭淑萍"},{"id":"024","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"2019-09-20","user":"阮乃峰"},{"id":"025","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"2019-09-20","user":"周瑋穎"},{"id":"026","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"涂志傑"},{"id":"027","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已退單","pay":"Line Pay","getTime":"2019-09-20","user":"陳宜喜"},{"id":"028","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"張秀柏"},{"id":"029","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"趙坤芃"},{"id":"030","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1},"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"駱語廷"},{"id":"031","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1},"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"王柔喜"},{"id":"032","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"黃淑明"},{"id":"033","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"王岳任"},{"id":"034","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"Google Pay","getTime":"2019-09-20","user":"吳必新"},{"id":"035","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"蘇尚財"},{"id":"036","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1},"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"王世豪"},{"id":"037","products":{"味噌燒肉飯":{"name":"味噌燒肉飯","price":75,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"施修喬"},{"id":"038","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"黃肇仁"},{"id":"039","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"黃家洋"},{"id":"040","products":{"雙拼":{"name":"雙拼","price":75,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"謝文啟"},{"id":"041","products":{"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"2019-09-20","user":"毛馨慧"},{"id":"042","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已退單","pay":"Apple Pay","getTime":"2019-09-20","user":"吳詩雅"},{"id":"043","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"Line Pay","getTime":"2019-09-20","user":"蔡星紫"},{"id":"044","products":{"香蒜烤雞飯":{"name":"香蒜烤雞飯","price":80,"count":1}},"status":"已完成","pay":"Apple Pay","getTime":"2019-09-20","user":"童林杰"},{"id":"045","products":{"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"龔郁涵"},{"id":"046","products":{"玫瑰油雞飯":{"name":"玫瑰油雞飯","price":75,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"周奇福"},{"id":"047","products":{"卡拉雞腿排飯":{"name":"卡拉雞腿排飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"林仲亦"},{"id":"048","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1},"迷迭香烤豬飯":{"name":"迷迭香烤豬飯","price":80,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"翁俊逸"},{"id":"049","products":{"雙拼":{"name":"雙拼","price":75,"count":1},"椒麻口水雞飯":{"name":"椒麻口水雞飯","price":80,"count":1}},"status":"已完成","pay":"現金","getTime":"2019-09-20","user":"蔣佳穎"},{"id":"050","products":{"蜜汁叉燒飯":{"name":"蜜汁叉燒飯","price":75,"count":1}},"status":"已退單","pay":"現金","getTime":"2019-09-20","user":"林怡珮"}]`)

export default {
  name: 'History',
  components: { Pagination },
  directives: { waves },
  data() {
    return {
      range: ['2019-09-20', '2019-09-21'],
      payWay: '',
      tableKey: 0,
      list: null,
      total: 10491,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 50,
        importance: undefined,
        title: undefined,
        type: undefined,
        sort: '+id'
      },
      importanceOptions: [1, 2, 3],
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

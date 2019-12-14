<template>
  <div class="tab-container">
    <div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:15px">
      <el-tag v-if="!isRunning">上次分析完成時間：2019-09-20 06:00</el-tag>
      <el-tag v-else>分析中，預計完成時間：{{ dateformat(Date.now() + 1000*60*24, 'yyyy-mm-dd hh:MM') }}</el-tag>
      <el-button type="primary" :disabled="isRunning" @click="analyze">進行分析</el-button>
    </div>
    <div style="display:flex;align-items:center;margin-bottom:15px">
      NT <el-input v-model="budget" type="numver" style="width:200px;margin:0 15px" />
      <el-button type="primary" :disabled="!isRunning && (budget < 50)">一鍵投放優惠卷</el-button>
      <el-button type="primary" :disabled="isRunning" @click="a">一鍵推薦客製化餐點</el-button>
      <el-button type="primary" :disabled="isRunning" @click="b">一鍵提醒使用優惠卷</el-button>
    </div>
    <el-table
      v-loading="isLoading"
      :data="tableData"
      stripe
      border
      style="width: 100%"
    >
      <el-table-column label="使用者" prop="name" width="80px" />
      <el-table-column label="客群類別" prop="category" width="50px" />
      <el-table-column label="未來推薦產品">
        <template slot-scope="scope">
          <div v-for="prod in formatProds(scope.row.like)" :key="prod">{{ prod }}</div>
        </template>
      </el-table-column>
      <el-table-column label="消費週期區間">
        <template slot-scope="scope">
          {{ scope.row.interval.upper }} ~ {{ scope.row.interval.lower }}天
        </template>
      </el-table-column>
      <el-table-column label="預估下次消費時間">
        <template slot-scope="scope">{{ dateformat(scope.row.nextBuy, 'yyyy-mm-dd') }}</template>
      </el-table-column>
      <el-table-column label="尚未使用的優惠卷">
        <template slot-scope="scope">{{ scope.row.coupon }}</template>
      </el-table-column>
      <el-table-column label="顧客黏著度評分" prop="loyalty" width="80px" />
      <el-table-column label="操作" prop="doText" />
    </el-table>

    <el-dialog
      title="分析準備中，共有58筆新資料"
      :visible.sync="dialogVisible"
      width="30%"
    >
      <span>{{ msg }}</span>
      <span slot="footer" class="dialog-footer">
        <el-button v-show="!isRunning" type="primary" @click="dialogVisible = false">取消</el-button>
        <el-button v-show="isRunning" type="primary" @click="dialogVisible = false">確定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// import tabPane from '@/views/tab/components/TabPane'
import cookies from 'js-cookie'
import dateformat from 'dateformat'
import delay from 'delay'
import crmData from './crm.json'

export default {
  name: 'CRM',
  // components: { tabPane },
  data() {
    return {
      budget: 0,
      dialogVisible: false,
      isRunning: false,
      isLoading: false,
      msg: '',
      tableData: []
    }
  },
  created() {
    this.isLoading = true
    this.isRunning = !!cookies.get('isRunning')
    setTimeout(() => {
      this.tableData = crmData
      this.isLoading = false
    }, 2000)
  },
  methods: {
    async analyze() {
      this.dialogVisible = true
      this.msg = '連接伺服器中...'
      await delay(1500)
      this.msg = '連接完成'
      await delay(900)
      this.msg = '資料準備中...'
      await delay(300)
      this.msg = '共有58筆新資料，10491筆過往資料，'
      await delay(2500)
      this.msg = '準備完成'
      await delay(900)

      try {
        await this.$confirm('由於新資料量較少，該分析結果可能大部分與前次相符，是否要繼續？', '警告', {
          confirmButtonText: '確定',
          cancelButtonText: '取消',
          type: 'warning'
        })
      } catch (error) {
        this.dialogVisible = false
        return
      }

      this.msg = '正在預估所需時間'
      await delay(1500)
      this.msg = '分析準備中'
      await delay(3000)
      this.msg = '開始分析，本次為延續型分析，因新資料量較少，耗時較短，預計24分鐘後完成。'
      this.setRunning(true)
    },
    async a() {
      try {
        await this.$confirm('確認要推播客製化推薦訊息？', '警告', {
          confirmButtonText: '確定',
          cancelButtonText: '取消',
          type: 'warning'
        })
      } catch (error) {
        console.log(error)
      }
    },
    async b() {
      try {
        await this.$confirm('確認要推播提醒使用優惠卷訊息？', '警告', {
          confirmButtonText: '確定',
          cancelButtonText: '取消',
          type: 'warning'
        })
      } catch (error) {
        console.log(error)
      }
    },
    setRunning(isRunning) {
      this.isRunning = isRunning
      cookies.set('isRunning', isRunning, { expires: 7 })
    },
    formatProds(products) {
      const list = []
      products.forEach(prod => {
        if (list.some(name => name === prod.name)) {
          return
        }
        list.push(prod.name)
      })
      return list
    },
    dateformat(dateString, format) {
      return dateformat(new Date(dateString), format)
    }
  }
}
</script>

<style scoped>
  .tab-container {
    margin: 30px;
  }
</style>

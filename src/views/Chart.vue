<template>
  <div class="dashboard-wrapper">
    <!-- Header -->
    <div class="dashboard-header mb-4 d-flex justify-content-between align-items-end">
      <div>
        <h1 class="display-5 font-weight-bold mb-1">Stock Analysis Dashboard</h1>
        <p class="text-muted">สรุปรายการสินค้าคงคลังแยกตามประเภท</p>
      </div>
      <div class="text-right">
        <b-badge variant="light-primary" class="p-2">
          อัปเดตล่าสุด: {{ currentTime }}
        </b-badge>
      </div>
    </div>

    <!-- ✅ 2x2 GRID -->
    <div class="grid-2x2">
      <b-card class="stock-card" title="กลุ่มไม้พาราแปรรูป (Sawn Timber)">
        <div class="chart-box bordered">
          <v-chart :options="chartOptions1" autoresize />
        </div>
      </b-card>

      <b-card class="stock-card" title="กลุ่มชิ้นส่วนเฟอร์นิเจอร์ (Components)">
        <div class="chart-box bordered">
          <v-chart :options="chartOptions2" autoresize />
        </div>
      </b-card>

      <b-card class="stock-card" title="กลุ่มอุปกรณ์ฟิตติ้ง (Fittings)">
        <div class="chart-box bordered">
          <v-chart :options="chartOptions3" autoresize />
        </div>
      </b-card>

      <b-card class="stock-card" title="สินค้าพร้อมส่ง (Ready to Ship)">
        <div class="chart-box bordered">
          <v-chart :options="chartOptions4" autoresize />
        </div>
      </b-card>
    </div>
  </div>
</template>

<script>
import ECharts from 'vue-echarts'
import 'echarts/lib/chart/bar'
import 'echarts/lib/component/tooltip'
import 'echarts/lib/component/grid'

export default {
  components: { 'v-chart': ECharts },

  data() {
    return {
      currentTime: new Date().toLocaleString('th-TH'),

      chartOptions1: this.createChartOption(
        ['21/2.5', '41/3', '41/2.5', '21/3.5'],
        [12500, 4200, 14800, 2800],
        '#7367f0'
      ),

      chartOptions2: this.createChartOption(
        ['ขาโต๊ะ A', 'ขาโต๊ะ B', 'คานกลาง', 'แผ่นท็อป'],
        [850, 1200, 450, 300],
        '#28c76f'
      ),

      chartOptions3: this.createChartOption(
        ['น็อต 2"', 'สกรู 1.5"', 'กาวลาเท็กซ์', 'กระดาษทราย'],
        [55000, 42000, 150, 800],
        '#ff9f43'
      ),

      chartOptions4: this.createChartOption(
        ['โต๊ะกินข้าว', 'เก้าอี้ไม้', 'ชั้นวางของ'],
        [120, 450, 85],
        '#00cfe8'
      )
    }
  },

  methods: {
    createChartOption(labels, values, color) {
      return {
        tooltip: {
          trigger: 'axis',
          axisPointer: { type: 'shadow' }
        },

        grid: {
          left: '5%',
          right: '5%',
          bottom: '10%',
          top: '10%',
          containLabel: true
        },

        xAxis: {
          type: 'category',
          data: labels,
          axisTick: { show: false },
          axisLabel: {
            color: '#6e6b7b',
            interval: 0,
            rotate: 15
          }
        },

        yAxis: {
          type: 'value',
          splitLine: {
            lineStyle: {
              type: 'dashed',
              color: '#ebe9f1'
            }
          }
        },

        series: [
          {
            name: 'คงเหลือ',
            type: 'bar',
            data: values,
            barWidth: '40%',

            itemStyle: {
              color: color,
              borderRadius: [6, 6, 0, 0]
            },

            label: {
              show: true,
              position: 'top',
              color: '#6e6b7b',
              formatter: (params) => params.value.toLocaleString()
            }
          }
        ]
      }
    }
  }
}
</script>

<style scoped>
.dashboard-wrapper {
  padding: 2.5rem;
  background-color: #f4f7f9;
  min-height: 100vh;
}

/* ✅ GRID 2x2 */
.grid-2x2 {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1.5rem;
}

/* มือถือ */
@media (max-width: 768px) {
  .grid-2x2 {
    grid-template-columns: 1fr;
  }
}

/* card */
.stock-card {
  border: 1px solid #dcdcdc;
  border-radius: 12px;
  background: #fff;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.04);
  transition: 0.3s;
}

.stock-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

/* chart */
.chart-box {
  height: 380px;
  width: 100%;
}

.chart-box.bordered {
  border: 1.5px solid #e0e0e0;
  border-radius: 12px;
  padding: 10px;
  background: #fff;
}

.chart-box.bordered:hover {
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.08);
}

/* echarts */
.echarts {
  width: 100% !important;
  height: 100% !important;
}

/* title */
.card-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #5e5873;
  border-bottom: 1px solid #f0f0f0;
  padding-bottom: 1rem;
  margin-bottom: 1.5rem;
}

/* badge */
.badge-light-primary {
  background-color: rgba(115, 103, 240, 0.12);
  color: #7367f0;
}
</style>
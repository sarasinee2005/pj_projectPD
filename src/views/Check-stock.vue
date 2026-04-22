<template>
  <div class="dashboard-wrapper">
    <div class="mb-4">
      <h3 class="font-weight-bold">Stock Dashboard </h3>
    </div>

    <div class="search-card shadow-sm mb-4 bg-white p-4" style="border-radius: 12px;">
      <div class="row align-items-end">
        <div class="col-md-5">
          <label class="small font-weight-bold text-secondary">ระบุรหัส FG code:</label>
          <input 
            type="text" 
            v-model="sapInput" 
            class="form-control" 
            placeholder="กรอกรหัส FG code..."
            @keyup.enter="processDisplay"
          >
        </div>
        <div class="col-md-7 mt-2">
          <button @click="processDisplay" class="btn btn-primary px-4 mr-2" style="background-color: #7367f0; border: none;">วิเคราะห์</button>
          <button @click="resetSearch" class="btn btn-outline-secondary px-4">ดูทั้งหมด</button>
        </div>
      </div>
    </div>

    <div class="position-relative" style="min-height: 400px;">
      <div v-if="isLoading" class="text-center py-5">
        <div class="spinner-border text-primary" role="status"></div>
        <p class="mt-3 font-weight-bold text-primary">กำลังโหลดกราฟทั้งหมด...</p>
      </div>

      <div v-else class="row">
        <div v-for="(group, sapId, index) in groupedData" :key="sapId" class="col-md-6 mb-4">
          <div class="chart-card shadow-sm p-4 bg-white border" style="border-radius: 12px; height: 100%;">
            <div class="sap-header mb-4" style="border-left: 5px solid #7367f0; padding-left: 15px;">
              <div class="sap-id font-weight-bold" style="color: #7367f0; font-size: 1.2rem;">{{ sapId }}</div>
              <div class="prod-name text-muted small mt-1">{{ group.name }}</div>
            </div>
            <div :ref="'chartRef' + index" style="width: 100%; height: 350px;"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import Papa from 'papaparse';

export default {
  name: 'StockDashboard',
  data() {
    return {
      URL_1: "https://docs.google.com/spreadsheets/d/e/2PACX-1vT-sDov5Hl1nWrRBz9jCiiBeatitpYjZp0fqPYngNmvdY1teoDLQrhPBNkMZATc-rktKV37X2Q1qzkm/pub?output=csv",
      URL_2: "https://docs.google.com/spreadsheets/d/e/2PACX-1vShU_0KepJ0kBsIYP6zPhKVuZEE4TbCDyvx81yzPMBbdtO3SJp1kf1bRc4hbFfs4ErULD0oDyK39CkE/pub?output=csv",
      sapInput: "",
      masterList: [],
      stockDictionary: {},
      groupedData: {},
      chartInstances: [],
      isLoading: true
    };
  },
  async mounted() {
    // ดึงข้อมูลทันทีเมื่อเปิดหน้าเว็บ
    await this.fetchData();
    window.addEventListener('resize', this.handleResize);
  },
  methods: {
    async fetchData() {
      this.isLoading = true;
      try {
        const [csv1, csv2] = await Promise.all([
          fetch(this.URL_1).then(r => r.text()),
          fetch(this.URL_2).then(r => r.text())
        ]);

        // 1. จัดการข้อมูล Stock (Link 2)
        const stockRows = Papa.parse(csv2, { header: false, skipEmptyLines: true }).data;
        this.stockDictionary = {};
        stockRows.forEach((row, i) => {
          if (i === 0) return;
          const materialKey = row[1] ? String(row[1]).trim() : ""; // Col B
          const sumValue = parseFloat(row[7]); // Col H
          if (materialKey) this.stockDictionary[materialKey] = isNaN(sumValue) ? 0 : sumValue;
        });

        // 2. จัดการข้อมูล Master (Link 1)
        const masterRows = Papa.parse(csv1, { header: false, skipEmptyLines: true }).data;
        let lastSap = "";
        let lastProd = "";
        this.masterList = [];
        
        masterRows.forEach((row, i) => {
          if (i === 0) return;
          if (row[0]?.trim()) lastSap = row[0].trim();
          if (row[1]?.trim()) lastProd = row[1].trim();
          const pCode = row[2] ? String(row[2]).trim() : "";
          const pName = row[3] ? row[3].trim() : pCode;

          if (pCode) {
            const qty = this.stockDictionary[pCode] !== undefined ? this.stockDictionary[pCode] : 0;
            this.masterList.push({ sap: lastSap, prod: lastProd, code: pCode, name: pName, value: qty });
          }
        });

        // เมื่อโหลดเสร็จให้ประมวลผลการแสดงผลทันที
        this.processDisplay(); 
      } catch (err) {
        console.error("Fetch Error:", err);
      } finally {
        this.isLoading = false;
      }
    },

    processDisplay() {
      const search = this.sapInput.trim().toLowerCase();
      const filtered = search 
        ? this.masterList.filter(item => item.sap.toLowerCase().includes(search))
        : this.masterList;

      const groups = {};
      filtered.forEach(item => {
        if (!groups[item.sap]) groups[item.sap] = { name: item.prod, items: [] };
        groups[item.sap].items.push(item);
      });

      this.groupedData = groups;
      
      this.chartInstances.forEach(chart => chart.dispose());
      this.chartInstances = [];

      // ใช้ setTimeout เพื่อให้แน่ใจว่า DOM ถูกวาดเสร็จก่อนวาดกราฟ
      setTimeout(() => {
        this.initCharts();
      }, 300);
    },

    initCharts() {
      Object.keys(this.groupedData).forEach((sapId, index) => {
        const container = this.$refs['chartRef' + index];
        if (!container || !container[0]) return;

        const myChart = echarts.init(container[0]);
        const dataSet = this.groupedData[sapId];

        myChart.setOption({
          tooltip: { trigger: 'axis' },
          grid: { top: '10%', bottom: '35%', left: '10%', right: '5%' },
          xAxis: {
            type: 'category',
            data: dataSet.items.map(it => it.name),
            axisLabel: { rotate: 45, fontSize: 10, interval: 0 }
          },
          yAxis: { type: 'value' },
          series: [{
            name: 'Stock',
            type: 'bar',
            data: dataSet.items.map(it => it.value),
            barWidth: '40%',
            itemStyle: { color: '#7367f0', borderRadius: [4, 4, 0, 0] },
            label: { show: true, position: 'top', fontSize: 10, color: '#7367f0' }
          }]
        });
        this.chartInstances.push(myChart);
      });
    },

    resetSearch() {
      this.sapInput = "";
      this.processDisplay();
    },

    handleResize() {
      this.chartInstances.forEach(chart => chart.resize());
    }
  }
};
</script>
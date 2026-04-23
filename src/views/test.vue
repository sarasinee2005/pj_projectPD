<template>
  <b-card no-body class="p-2 shadow-none border production-report-container">
    <div class="d-flex justify-content-between align-items-center mb-2">
      <div>
        <h3 class="mb-0 text-primary font-weight-bolder"> รายงานยอดผลิต (Production Yield)</h3>
        <small v-if="lastUpdate" class="text-muted">อัปเดตล่าสุด: {{ lastUpdate }}</small>
      </div>
      <div class="d-flex align-items-center">
        <div class="text-right mr-2">
          <small class="text-muted d-block font-weight-bold">รวมมูลค่าตามการกรอง</small>
          <h4 class="mb-0 text-danger font-weight-bolder">{{ totalPriceFormatted }} บาท</h4>
        </div>
        <b-button 
          variant="primary" 
          @click="fetchAndSortData" 
          :disabled="loading"
          v-ripple.400="'rgba(255, 255, 255, 0.15)'"
          class="shadow-sm"
        >
          <span v-if="loading">⏳ กำลังดึง...</span>
          <span v-else>🔄 รีเฟรชข้อมูล</span>
        </b-button>
      </div>
    </div>

    <b-row class="mb-3 bg-light mx-0 py-1 rounded border shadow-xs">
      <b-col md="2" sm="6">
        <b-form-group label="ปี" label-class="font-weight-bold small text-primary mb-0">
          <b-form-select v-model="filters.year" :options="yearOptions" @change="updateOptions" size="sm" class="custom-v-select" />
        </b-form-group>
      </b-col>
      <b-col md="3" sm="6">
        <b-form-group label="เดือน" label-class="font-weight-bold small text-primary mb-0">
          <b-form-select v-model="filters.month" @change="updateOptions" size="sm" class="custom-v-select">
            <option v-for="(m, index) in months" :key="index" :value="index + 1">{{ m }}</option>
          </b-form-select>
        </b-form-group>
      </b-col>
      <b-col md="4" sm="6">
        <b-form-group label="เลือกช่วงสัปดาห์" label-class="font-weight-bold small text-primary mb-0">
          <b-form-select v-model="filters.week" size="sm" class="custom-v-select">
            <option value="all">ดูข้อมูลทั้งเดือน</option>
            <option v-for="w in weekOptions" :key="w.value" :value="w.value">{{ w.text }}</option>
          </b-form-select>
        </b-form-group>
      </b-col>
      <b-col md="3" sm="6">
        <b-form-group label="ระบุวันที่" label-class="font-weight-bold small text-primary mb-0">
          <b-form-select v-model="filters.day" size="sm" class="custom-v-select">
            <option value="all">ทั้งหมด</option>
            <option v-for="d in dayOptions" :key="d" :value="d">{{ d }}</option>
          </b-form-select>
        </b-form-group>
      </b-col>
    </b-row>

    <div class="table-responsive rounded border shadow-sm">
      <table class="table table-hover mb-0 vuexy-table">
        
      
      
        <thead class="bg-light text-secondary small font-weight-bold"> 
  <tr>
    <th style="width: 3%" class="text-left align-bottom">#</th>
     <th style="width: 7%" class="text-left align-bottom">
      เวลา
    </th>

    <th style="width: 10%" class="text-left align-bottom">
      วันที่
    </th>

   
    <th style="width: 12%" class="text-left align-bottom">
      ใบผลิต (MO)
      <b-form-input v-model="colFilters.moNo" size="sm" placeholder="Search..." class="mt-1 mt-input" />
    </th>
    <th style="width: 15%" class="text-left align-bottom">
      รหัสสินค้า
      <b-form-input v-model="colFilters.pCode" size="sm" placeholder="Search..." class="mt-1 mt-input" />
    </th>
    <th style="width: 25%" class="text-left align-bottom">
      รายละเอียดสินค้า
      <b-form-input v-model="colFilters.pDesc" size="sm" placeholder="Search..." class="mt-1 mt-input" />
    </th>
    <th style="width: 10%" class="text-left align-bottom">
      สี
      <b-form-select v-model="colFilters.color" size="sm" class="mt-1 mt-input">
        <option value="">ทั้งหมด</option>
        <option v-for="c in colorOptions" :key="c" :value="c">{{ c }}</option>
      </b-form-select>
    </th>
    <th style="width: 8%" class="text-right align-bottom">
      จำนวน
    </th>
    <th style="width: 10%" class="text-right align-bottom">มูลค่า (บาท)</th>
  </tr>
</thead>

        <tbody class="small font-weight-bold text-dark">
  <tr v-if="loading" class="text-center">
    <td colspan="9" class="py-5 text-muted">กำลังดึงข้อมูล...</td>
  </tr>
  
  <tr v-for="(item, index) in filteredItems" :key="index" class="data-row">
    <td class="text-left text-muted border-right">{{ index + 1 }}</td>
    <td class="text-left border-right">
      <span class="text-muted"> {{ item.displayTime }}</span>
    </td>

    <td class="text-left border-right">
      <b-badge variant="light-success" class="badge-sm">{{ item.displayDate }}</b-badge>
    </td>

    <td class="text-left border-right">{{ item.moNo }}</td>
    <td class="text-left border-right">
      <code class="text-danger bg-light-danger px-1 rounded">{{ item.pCode }}</code>
    </td>
    <td class="text-left border-right desc-column">{{ item.pDesc }}</td>

    <td class="text-left border-right">{{ item.color }}</td>
    <td class="text-left border-right text-primary">{{ item.qty.toLocaleString() }}</td>
    <td class="text-right font-weight-bolder">{{ item.priceStr }}</td>
  </tr>

  <tr v-if="filteredItems.length > 0" class="summary-row bg-light">
    <td colspan="8" class="text-right font-weight-bolder">รวมมูลค่าตามการกรอง:</td>
    <td class="text-right text-danger font-weight-bolder" style="font-size: 1.1rem;">
      {{ totalPriceFormatted }}
    </td>
  </tr>
</tbody>
      </table>
    </div>
  </b-card>
</template>









<script>
import axios from 'axios'
import { BCard, BRow, BCol, BButton, BFormGroup, BFormSelect, BBadge, BFormInput } from 'bootstrap-vue'
import Ripple from 'vue-ripple-directive'

export default {
  name: 'ProductionYield',
  components: { BCard, BRow, BCol, BButton, BFormGroup, BFormSelect, BBadge, BFormInput },
  directives: { Ripple },
  data() {
    return {
      CSV_URL: 'https://docs.google.com/spreadsheets/d/e/2PACX-1vSpJBH3HzXDIWC0B_WuHdAfSJiyZPfMTVldJVcH9b4LNStGfZpUewPPEjfHIvIowGua2fhdOApJT1E4/pub?output=csv',
      loading: false,
      lastUpdate: '',
      items: [],
      filters: {
        year: new Date().getFullYear(),
        month: new Date().getMonth() + 1,
        week: 'all',
        day: 'all'
      },
      colFilters: {
        date: '',
        moNo: '',
        pCode: '',
        pDesc: '',
        color: ''
      },
      months: ['มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน', 'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม', 'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'],
      yearOptions: [],
      dayOptions: [],
      weekOptions: []
    }
  },
  computed: {
    // ระบบ Live กรองข้อมูลแยกตามคอลัมน์
    filteredItems() {
      return this.items.filter(item => {
        return (
          item.displayDate.toLowerCase().includes(this.colFilters.date.toLowerCase()) &&
          item.moNo.toLowerCase().includes(this.colFilters.moNo.toLowerCase()) &&
          item.pCode.toLowerCase().includes(this.colFilters.pCode.toLowerCase()) &&
          item.pDesc.toLowerCase().includes(this.colFilters.pDesc.toLowerCase()) &&
          (this.colFilters.color === '' || item.color === this.colFilters.color)
        );
      });
    },
    // คำนวณมูลค่ารวมเฉพาะรายการที่ถูกกรอง
    totalPriceFormatted() {
      const total = this.filteredItems.reduce((sum, item) => {
        const val = parseFloat(item.priceStr.replace(/,/g, '')) || 0;
        return sum + val;
      }, 0);
      return total.toLocaleString('th-TH', { minimumFractionDigits: 2 });
    },
    // ดึงรายชื่อสีที่ไม่ซ้ำกันมาทำ Dropdown
    colorOptions() {
      const colors = this.items.map(i => i.color).filter(v => v && v !== '-');
      return [...new Set(colors)].sort();
    }
  },
  mounted() {
    this.setupYearSelect();
    this.updateOptions();
  },
  methods: {
    setupYearSelect() {
      const cur = new Date().getFullYear();
      this.yearOptions = [cur - 1, cur, cur + 1];
    },
    updateOptions() {
      const year = parseInt(this.filters.year);
      const month = parseInt(this.filters.month) - 1;
      const lastDate = new Date(year, month + 1, 0);
      this.dayOptions = Array.from({ length: lastDate.getDate() }, (_, i) => i + 1);

      this.weekOptions = [];
      let current = new Date(year, month, 1);
      let weekNum = 1;
      while (current <= lastDate) {
        let start = new Date(current);
        let end = new Date(current);
        end.setDate(current.getDate() + (6 - current.getDay()));
        if (end > lastDate) end = new Date(lastDate);
        this.weekOptions.push({
          value: `${start.getDate()}|${end.getDate()}`,
          text: `สัปดาห์ที่ ${weekNum} (${start.getDate().toString().padStart(2,'0')}/${(month+1).toString().padStart(2,'0')})`
        });
        current.setDate(end.getDate() + 1);
        weekNum++;
      }
    },
    async fetchAndSortData() {
      this.loading = true;
      try {
        const res = await axios.get(`${this.CSV_URL}&t=${Date.now()}`);
        const lines = res.data.split(/\r?\n/);
        
        let dStart = 1, dEnd = 31;
        if (this.filters.day !== 'all') dStart = dEnd = parseInt(this.filters.day);
        else if (this.filters.week !== 'all') [dStart, dEnd] = this.filters.week.split('|').map(Number);

        let temp = [];
        for (let i = 1; i < lines.length; i++) {
          if (!lines[i].trim()) continue;
          const cols = this.parseCSV(lines[i]);
          const rawDate = cols[0] || "";
          const datePart = rawDate.split(" ")[0];
          const [m, d, yRaw] = datePart.split('/').map(Number);
          let y = yRaw < 2000 ? yRaw + 2000 : (yRaw > 2500 ? yRaw - 543 : yRaw);

          if (y === this.filters.year && m === this.filters.month && d >= dStart && d <= dEnd) {
            const price = parseFloat((cols[14] || "0").replace(/,/g, '')) || 0;
            temp.push({
              day: d,
              displayDate: datePart,
              displayTime: rawDate.split(" ")[1]?.substring(0,5) || "-",
              moNo: cols[3] || "-",
              pCode: cols[7] || "-",
              pDesc: cols[8] || "-",
              qty: parseInt((cols[9] || "0").replace(/,/g, '')) || 0,
              color: cols[13] || "-",
              priceStr: price.toLocaleString('th-TH', { minimumFractionDigits: 2 })
            });
          }
        }
        this.items = temp.sort((a, b) => a.day - b.day);
        this.lastUpdate = new Date().toLocaleString('th-TH');
      } catch (err) { console.error(err); } 
      finally { this.loading = false; }
    },
    parseCSV(text) {
      const result = [];
      let cur = '', inQuote = false;
      for (let i = 0; i < text.length; i++) {
        let char = text[i];
        if (char === '"') inQuote = !inQuote;
        else if (char === ',' && !inQuote) { result.push(cur.trim()); cur = ''; }
        else cur += char;
      }
      result.push(cur.trim());
      return result;
    }
  }
}
</script>

<style lang="scss" scoped>
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap');

.production-report-container {
  font-family: 'Sarabun', sans-serif;

  .vuexy-table {
    table-layout: fixed !important;
    width: 100% !important;

    th {
      font-size: 0.85rem;
      padding: 10px 8px;
      vertical-align: middle;
      border: none;
    }
    
    .mt-input {
      background-color: rgba(255, 255, 255, 0.9);
      border: 1px solid #d8d6de;
      height: 28px !important;
      font-size: 0.75rem;
      &:focus { border-color: #7367f0; box-shadow: none; }
    }

    td {
      padding: 10px 8px;
      vertical-align: middle;
      border-bottom: 1px solid #ebe9f1;
      &.border-right { border-right: 1px solid #f3f2f7; }
    }
  }

  .badge-light-success { background-color: rgba(40, 199, 111, 0.12); color: #28c76f; font-weight: 800; }
  .bg-light-danger { background-color: rgba(234, 84, 85, 0.12) !important; color: #ea5455 !important; }
  code { font-weight: 700; }
  
  .shadow-xs { box-shadow: 0 2px 4px 0 rgba(34, 41, 47, 0.1) !important; }
  ::v-deep .custom-v-select { border-color: #d8d6de; &:focus { border-color: #7367f0; box-shadow: none; } }

}
</style>
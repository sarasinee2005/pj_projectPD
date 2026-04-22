<template>
  <b-card no-body class="p-2 shadow-none border">
    <div class="no-print">
      <div class="d-flex justify-content-between align-items-center mb-2">
        <h3 class="mb-0 text-primary font-weight-bolder">ระบบคำนวณการผลิตชิ้นส่วน</h3>
        <b-button variant="flat-primary" size="sm" @click="fetchData(true)" :disabled="loading">
          <span v-if="loading">⏳ กำลังโหลด...</span>
          <span v-else>🔄 รีเฟรชข้อมูล</span>
        </b-button>
      </div>

      <b-row class="mb-3">
        <b-col md="9">
          <b-form-group>
            <template #label>
              <span class="text-primary font-weight-bold" style="font-size: 1.1rem;">
                🔍 ค้นหาด่วน (PO | SAP | รายการ | สี):
              </span>
            </template>
            <v-select
              v-model="searchTemp"
              :options="allPlans"
              :filter="customFilter"
              label="display"
              placeholder="พิมพ์เพื่อค้นหาเร็ว..."
              @input="handleSelect"
              class="bg-white shadow-sm custom-v-select mb-2"
            />
          </b-form-group>
        </b-col>
        
        <b-col md="3">
          <div class="target-card-fixed shadow-sm border-primary">
            <div class="target-label">เป้าหมายผลิต</div>
            <div class="target-value text-primary">
              {{ targetQty.toLocaleString() }} <small>ตัว</small>
            </div>
          </div>
        </b-col>
      </b-row>

      <div v-if="!selectedPlan" class="plan-selection-area">
        <div class="table-responsive bg-white rounded shadow-sm border">
          <table class="table table-hover mb-0">
            <thead class="bg-light text-secondary small font-weight-bold">
              <tr>
                <th style="width: 5%" class="text-center">#</th>
                <th style="width: 15%">PO BATCH
                  <b-form-input v-model="filterPO" size="sm" placeholder="..." class="mt-1 mt-input" @input="currentPage = 1" />
                </th>
                <th style="width: 20%">CODE SAP
                  <b-form-input v-model="filterSAP" size="sm" placeholder="..." class="mt-1 mt-input" @input="currentPage = 1" />
                </th>
                <th style="width: 45%">DESCRIPTION (รายการ)
                  <b-form-input v-model="filterName" size="sm" placeholder="..." class="mt-1 mt-input" @input="currentPage = 1" />
                </th>
                <th style="width: 15%">COLOR (สี)
                  <b-form-input v-model="filterColor" size="sm" placeholder="..." class="mt-1 mt-input" @input="currentPage = 1" />
                </th>
              </tr>
            </thead>
            <tbody class="small">
              <tr 
                v-for="(p, i) in paginatedPlans" 
                :key="i" 
                @click="handleSelect(p)"
                class="cursor-pointer plan-row"
              >
                <td class="align-middle text-center text-muted">
                  {{ (currentPage - 1) * perPage + (i + 1) }}
                </td>
                <td class="align-middle">{{ p.po }}</td>
                <td class="align-middle font-weight-bold">{{ p.sap }}</td>
                <td class="align-middle">{{ p.name }}</td>
                <td class="align-middle">{{ p.color }}</td>
              </tr>
              <tr v-if="filteredPlans.length === 0">
                <td colspan="5" class="text-center py-5 text-muted">ไม่พบข้อมูลที่ค้นหา</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="d-flex justify-content-between align-items-center mt-3 px-1">
          <div class="text-muted small font-weight-bold">
            แสดง {{ Math.min(filteredPlans.length, (currentPage - 1) * perPage + 1) }} 
            ถึง {{ Math.min(filteredPlans.length, currentPage * perPage) }} 
            จากทั้งหมด {{ filteredPlans.length }} รายการ
          </div>
          <b-pagination
            v-model="currentPage"
            :total-rows="filteredPlans.length"
            :per-page="perPage"
            first-number
            last-number
            align="right"
            class="mb-0 pagination-purple"
          />
        </div>
      </div>

      <div v-else class="bom-detail-area mt-4">
        <div class="px-3 py-2 bg-light border rounded mb-3 d-flex justify-content-between align-items-center">
          <div>
            <span class="badge badge-primary mr-2">SELECTED</span>
            <strong>PO:</strong> {{ selectedPlan.po }} | <strong>SAP:</strong> {{ selectedPlan.sap }} | {{ selectedPlan.name }} | <strong>สี:</strong> {{ selectedPlan.color }}
          </div>
          <b-button variant="outline-danger" size="sm" @click="clearSelection">ยกเลิกรายการ</b-button>
        </div>

        <b-button 
          v-ripple.400="'rgba(255, 255, 255, 0.15)'"
          variant="dark"
          class="mb-3 font-weight-bold shadow-sm" 
          @click="printOrder" 
          block
        >
          🖨️ พิมพ์ใบสั่งผลิตชิ้นส่วน (A4)
        </b-button>

        <div class="table-responsive">
          <table class="table-bom">
            <thead>
              <tr>
                <th style="width: 5%">#</th>
                <th>รหัส (Part Code)</th>
                <th style="width: 35%">รายการชิ้นส่วน</th>
                <th class="text-center">ต่อตัว</th>
                <th class="text-center">ต้องใช้</th>
                <th class="text-center bg-gray">สต็อก </th>
                <th class="bg-orange text-white text-center">ผลิตเพิ่ม</th>
                <th class="bg-danger text-white text-center">ค้างผลิต</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in tableData" :key="index">
                <td class="text-center text-muted">{{ index + 1 }}</td>
                <td class="text-center font-weight-bold">{{ row.partCode }}</td>
                <td class="pl-1">{{ row.itemName }}</td>
                <td class="text-center">{{ row.perUnit }}</td>
                <td class="text-center text-primary font-weight-bold">{{ row.totalNeed.toLocaleString() }}</td>
                <td class="text-center bg-gray-light">{{ row.stock.toLocaleString() }}</td>
                <td class="text-center bg-orange-light text-orange font-weight-bolder">
                  {{ row.toProduce.toLocaleString() }}
                </td>
                <td class="text-center bg-danger-light text-danger font-weight-bold">
                  {{ row.toProduce.toLocaleString() }}
                </td>
              </tr>
              <tr v-if="tableData.length === 0">
                <td colspan="8" class="text-center py-5">❌ ไม่พบข้อมูลชิ้นส่วนประกอบสำหรับ SAP นี้</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div id="print-isolation-layer" class="print-only">
      <h1 class="text-center font-weight-bold mb-4">ใบสั่งผลิตชิ้นส่วน (Production Order)</h1>
      <div class="print-header-info">
        <div>
          <p><strong>Project Order:</strong> {{ selectedPlan ? selectedPlan.po : '-' }}</p>
          <p><strong>รหัสสินค้า (fg code):</strong> {{ selectedPlan ? selectedPlan.sap : '-' }}</p>
          <p><strong>รายการ:</strong> {{ selectedPlan ? selectedPlan.name : '-' }}</p>
        </div>
        <div class="text-right">
          <p><strong>วันที่พิมพ์:</strong> {{ new Date().toLocaleDateString('th-TH') }}</p>
          <p><strong>สี (Color):</strong> {{ selectedPlan ? selectedPlan.color : '-' }}</p>
        </div>
      </div>
      <div class="print-target-banner mb-4">
        เป้าหมายผลิตทั้งหมด: {{ targetQty.toLocaleString() }} ตัว
      </div>
      <table class="table-print">
        <thead>
          <tr>
            <th>#</th>
            <th>รหัส (Part Code)</th>
            <th style="width: 30%">รายการ (Item Name)</th>
            <th>ต่อตัว</th>
            <th>ต้องผลิต</th>
            <th>สต็อก</th>
            <th class="bg-print-gray">ผลิตเพิ่ม</th>
            <th class="bg-print-gray">ค้างผลิต</th>
            <th>เบิกสี</th>
            <th>ของเสีย</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, index) in tableData" :key="index">
            <td class="text-center">{{ index + 1 }}</td>
            <td class="text-center font-weight-bold">{{ row.partCode }}</td>
            <td class="pl-2">{{ row.itemName }}</td>
            <td class="text-center">{{ row.perUnit }}</td>
            <td class="text-center">{{ row.totalNeed.toLocaleString() }}</td>
            <td class="text-center">{{ row.stock.toLocaleString() }}</td>
            <td class="text-center font-weight-bold">{{ row.toProduce.toLocaleString() }}</td>
            <td class="text-center font-weight-bold">{{ row.toProduce.toLocaleString() }}</td>
            <td class="text-center"></td>
            <td class="text-center"></td>
          </tr>
        </tbody>
      </table>
    </div>
  </b-card>
</template>

<script>
import vSelect from 'vue-select'
import axios from 'axios'
import { BRow, BCol, BButton, BFormGroup, BCard, BFormInput, BPagination } from 'bootstrap-vue'
import Ripple from 'vue-ripple-directive'

export default {
  components: { vSelect, BRow, BCol, BButton, BFormGroup, BCard, BFormInput, BPagination },
  directives: { Ripple },
  data() {
    return {
      loading: false,
      allPlans: [],
      stockMap: {},
      bomDatabase: {},
      searchTemp: null, 
      selectedPlan: null, 
      targetQty: 0,
      tableData: [],
      filterPO: '',
      filterSAP: '',
      filterName: '',
      filterColor: '',
      currentPage: 1,
      perPage: 15
    }
  },
  computed: {
    filteredPlans() {
      return this.allPlans.filter(p => {
        return (p.po || '').toLowerCase().includes(this.filterPO.toLowerCase()) &&
               (p.sap || '').toLowerCase().includes(this.filterSAP.toLowerCase()) &&
               (p.name || '').toLowerCase().includes(this.filterName.toLowerCase()) &&
               (p.color || '').toLowerCase().includes(this.filterColor.toLowerCase());
      });
    },
    paginatedPlans() {
      const start = (this.currentPage - 1) * this.perPage;
      const end = start + this.perPage;
      return this.filteredPlans.slice(start, end);
    }
  },
  mounted() { this.fetchData() },
  methods: {
    async fetchData(force = false) {
      this.loading = true
      try {
        const urlPlan = "https://docs.google.com/spreadsheets/d/e/2PACX-1vQUfe59fFqNK3a6yzya8ZB4W_2dqqBg0c_06ZpH3QQ65J53FfxoEO5iPFW9LEksMEexwLOx0XTb5UcF/pub?output=csv"
        const urlStock = "https://docs.google.com/spreadsheets/d/e/2PACX-1vShU_0KepJ0kBsIYP6zPhKVuZEE4TbCDyvx81yzPMBbdtO3SJp1kf1bRc4hbFfs4ErULD0oDyK39CkE/pub?output=csv"
        const urlBOM = "https://docs.google.com/spreadsheets/d/e/2PACX-1vT-sDov5Hl1nWrRBz9jCiiBeatitpYjZp0fqPYngNmvdY1teoDLQrhPBNkMZATc-rktKV37X2Q1qzkm/pub?output=csv"
        const [resPlan, resStock, resBOM] = await Promise.all([axios.get(urlPlan), axios.get(urlStock), axios.get(urlBOM)])
        
        const tempStockMap = {}
        resStock.data.split(/\r?\n/).forEach((line, i) => {
          if (i === 0 || !line) return
          const cols = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          const code = (cols[1] || "").replace(/"/g, '').trim();
          const val = (cols[7] || "0").replace(/[",]/g, '').trim();
          tempStockMap[code] = parseInt(val) || 0
        })
        this.stockMap = Object.freeze(tempStockMap)

        const tempDB = {}
        let lastSap = ""
        resBOM.data.split(/\r?\n/).forEach((line, i) => {
          if (i === 0 || !line) return
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          let currentSap = (c[0] || "").replace(/"/g, '').trim();
          if (currentSap) lastSap = currentSap
          const pCode = (c[2] || "").replace(/"/g, '').trim();
          const pName = (c[3] || "").replace(/"/g, '').trim();
          const qty = parseFloat((c[5] || "0").replace(/[",]/g, '')) || 0
          if (lastSap && pCode) {
            if (!tempDB[lastSap]) tempDB[lastSap] = []
            tempDB[lastSap].push({ code: pCode, name: pName, perUnit: qty })
          }
        })
        this.bomDatabase = Object.freeze(tempDB)

        const pLines = resPlan.data.split(/\r?\n/)
        const plans = pLines.slice(1).map(line => {
          if (!line) return null
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          const sap = (c[3] || "").replace(/"/g, '').trim();
          if (!sap) return null
          const po = (c[2] || "").replace(/"/g, '').trim();
          const name = (c[6] || "").replace(/"/g, '').trim();
          const color = (c[7] || "").replace(/"/g, '').trim();
          return { po, sap, name, color, qty: parseInt((c[8] || "0").replace(/[",]/g, '')) || 0,
            display: `PO: ${po} | SAP: ${sap} | ${name} | ${color}` }
        }).filter(v => v !== null)
        this.allPlans = Object.freeze(plans.sort((a, b) => a.name.localeCompare(b.name, 'th')))
      } catch (err) { console.error(err) } finally { this.loading = false }
    },
    customFilter(options, search) {
      if (!search) return options.slice(0, 50);
      const s = search.toLowerCase();
      return options.filter(o => o.display.toLowerCase().includes(s)).slice(0, 50);
    },
    handleSelect(val) {
      if (val) {
        this.selectedPlan = val;
        this.calculateBOM(val);
        setTimeout(() => { this.searchTemp = null; }, 50);
      }
    },
    clearSelection() {
      this.selectedPlan = null;
      this.tableData = [];
      this.targetQty = 0;
    },
    calculateBOM(val) {
      this.targetQty = val.qty
      const components = this.bomDatabase[val.sap] || []
      this.tableData = Object.freeze(components.map(item => {
        const need = item.perUnit * this.targetQty
        const stock = this.stockMap[item.code] || 0
        return { partCode: item.code, itemName: item.name, perUnit: item.perUnit,
          totalNeed: need, stock: stock, toProduce: Math.max(0, need - stock) }
      }))
    },
    printOrder() { window.print() }
  }
}
</script>

<style lang="scss" scoped>
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap');
@import '~vue-select/dist/vue-select.css';

.target-card-fixed {
  background-color: #f8faff; border: 2px solid #7367f0; border-radius: 12px;
  padding: 10px; height: 100%; display: flex; flex-direction: column;
  justify-content: center; align-items: center; text-align: center; min-height: 85px;
  .target-label { font-size: 0.9rem; font-weight: 600; color: #5e5873; margin-bottom: 2px; }
  .target-value { font-size: 2rem; font-weight: 800; line-height: 1; small { font-size: 0.9rem; color: #6e6b7b; } }
}

.cursor-pointer { cursor: pointer; }
.plan-row {
  transition: background 0.2s;
  &:hover { background-color: rgba(115, 103, 240, 0.08) !important; }
}
.mt-input { height: 32px; font-size: 0.75rem; border-radius: 4px; }

.pagination-purple {
  ::v-deep .page-item {
    &.active .page-link { background-color: #7367f0 !important; border-color: #7367f0 !important; color: #fff !important; }
    .page-link { color: #6e6b7b; border: none; background-color: #f3f2f7; margin: 0 3px; border-radius: 6px; padding: 8px 14px; font-weight: 600; }
    &.disabled .page-link { background-color: #f8f8f8; color: #b9b9c3; }
  }
}

.table-bom {
  width: 100%; border-collapse: collapse;
  th, td { border: 1px solid #dae1e7; padding: 10px 8px; vertical-align: middle; font-size: 0.85rem; }
  th { background-color: #5d5fef; color: white; text-align: center; }
  .bg-gray-light { background-color: #f8f8f8; }
  .bg-orange-light { background-color: rgba(255, 159, 67, 0.12); }
  .text-orange { color: #ff9f43 !important; }
  .bg-danger-light { background-color: rgba(234, 84, 85, 0.12); }
}

::v-deep .custom-v-select {
  .vs__dropdown-toggle { border: 1px solid #d8d6de !important; border-radius: 0.357rem !important; padding: 6px 8px !important; }
}

.print-only { display: none; }
</style>

<style lang="scss">
@media print {
  .no-print, nav, aside, footer, header { display: none !important; }
  body * { visibility: hidden; }
  #print-isolation-layer, #print-isolation-layer * { visibility: visible; }
  #print-isolation-layer { display: block !important; position: absolute; left: 0; top: 0; width: 100%; padding: 0.5cm; }
  .table-print { width: 100%; border-collapse: collapse; th, td { border: 1.5px solid #000 !important; padding: 8px 5px; font-size: 10pt; } }
  .print-target-banner { border: 2px solid #000; background: #eee !important; text-align: center; font-size: 1.4rem; font-weight: bold; padding: 10px; }
}
</style>
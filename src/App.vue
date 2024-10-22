<template>
  <main class="main">
    <div v-if="isLoading">Загрузка...</div>
    <div v-else class="main-container">
      <v-dialog v-model="dialogStages" max-width="600">
        <v-card>
          <v-card-title>
            <span class="headline">Ошибка</span>
          </v-card-title>
          <v-card-text>
            <p>Выберите стадии сделок.</p>
          </v-card-text>
          <v-card-actions>
            <v-btn color="red" text @click="dialogStages = false">Закрыть</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-dialog v-model="dialog" max-width="600">
        <v-card>
          <v-card-title>
            <span class="headline">Ошибка</span>
          </v-card-title>
          <v-card-text>
            <p>Произошла ошибка при выполнении операции. Пожалуйста, попробуйте еще раз.</p>
          </v-card-text>
          <v-card-actions>
            <v-btn color="red" text @click="dialog = false">Закрыть</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-container fluid>
      <h1>Сортировка сделок</h1>
      <v-form>
        <v-select v-model="selectedStages" :items="stges" label="Выберите стадии сделок" item-title="NAME" item-value="STATUS_ID" multiple chips clearable>
          <template v-slot:prepend-item>
            <v-list-item @click="">
              <v-list-item-content>
                <v-list-item-title>
                  <v-checkbox label="Выбрать все стадии" v-model="selectedAllStages" @change="selectAllStages" /> </v-list-item-title>
              </v-list-item-content>
            </v-list-item>
          </template>
        </v-select>
        <div class="form-buttons">
          <v-btn color="primary" @click="submit">Получить сделки</v-btn>
        </div>
      </v-form>
      <v-row class="block">
        <!-- Первый блок с заголовком и текстом -->
        <v-col cols="12">
          <v-card>
            <v-card-title>
              <h4>Общая сумма сделок</h4>
            </v-card-title>
            <v-card-text>
              {{ formatNumber(this.sumDeals) }}
            </v-card-text>
          </v-card>
        </v-col>
        <v-col cols="12">
          <v-card>
            <v-card-title>Сумма сделок со счетами в стадии:</v-card-title>
            <v-card-text>
              <v-row>
                <v-col cols="12" md="12">
                  <p>Аванс получен: {{ formatNumber(this.invoicesPrepayed) }}</p>
                </v-col>
                <v-col cols="12" md="12">
                  <p>Оплачен: {{ formatNumber(this.invoicesPayed) }}</p>
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
      <v-data-table
        :headers="headers"
        :items="paginatedItems"
        :items-per-page="itemsPerPage"
        :pagination.sync="pagination"
        sort-desc
        class="elevation-1"
      >
      <template v-slot:item.TITLE="{ item }">
        <a :href="`https://${domain}/crm/deal/details/${item.ID}/`" target="_blank">{{ item.TITLE }}</a>
      </template>
      <template v-slot:item.OPPORTUNITY="{ item }">
        <span>{{ formatNumber(item.OPPORTUNITY) }}</span>
      </template>
      <template v-slot:item.PAYED="{ item }">
        <a v-if="item.INVOICE_ID" :href="`https://${domain}/crm/type/31/details/${item.INVOICE_ID}/`" target="_blank">{{ formatNumber(item.PAYED) }}</a>
        <span v-else>{{ formatNumber(item.PAYED) }}</span>
      </template>
        <template #top>
          <v-toolbar flat>
            <v-toolbar-title>Таблица сделок</v-toolbar-title>
            <v-checkbox class="table-checkbox" v-model="isChecked" label="Скрыть неоплаченные сделки" @change="displayData"></v-checkbox>
          </v-toolbar>
        </template>
      </v-data-table>
      <v-row>
        <v-col>
          <v-select
            v-model="itemsPerPage"
            :items="[10, 25, 50]"
            label="Сделок на странице"
            @change="updatePagination"
            @input="updatePagination"
          ></v-select>
        </v-col>
      </v-row>
      <v-pagination
        @input="updatePagination"
        v-model="pagination.page"
        :length="pageCount"
      ></v-pagination>
    </v-container>
    </div>
  </main>
</template>

<script>


  /**
   * crm.item 31 это сущность новые счета
   * "DT31_1:P" оплачен
   * "DT31_1:UC_IFGYKU" аванс получен
   */

  export default {
      data() {
        return {
          isChecked: true,
          currencyTemplate: " руб.",
          domain: "",
          items: [],
          deals: [],
          courses: [],
          invoices: [],
          selectedStages: [],
          invoicesStages: {},
          sumDeals: 0,
          invoicesPayed: 0,
          invoicesPrepayed: 0,
          dialog: false,
          dialogStages: false,
          selectedAllStages: false,
          isLoading: true,
          stges: [],
          stgesIds: [],
          pagination: {
            page: 1,
            rowsPerPage: 10,
          },
          itemsPerPage: 10,
          headers: [
            { title: 'Название', align: 'center', key: 'TITLE' },
            { title: 'Сумма, руб.', align: 'center', key: "OPPORTUNITY" },
            { title: 'Оплаченная сумма, руб.', align: 'center', key: "PAYED" },
          ],
        };
      },
      computed: {
        paginatedItems() {
          let start = (this.pagination.page - 1) * this.itemsPerPage;
          if(start >= this.items.length){
            start = 0;
            this.pagination.page = 1;
          }
          const end = start + this.itemsPerPage;
          return this.items.slice(start, end);
        },
        pageCount() {
          return Math.ceil(this.items.length / this.itemsPerPage);
        },
      },
      methods: {
        async submit(){
          if(this.selectedStages.length === 0){
            this.dialogStages = true;
            return;
          }
          this.isLoading = true;
         try {
            this.filteredDeals = [];
            this.deals = await this.callApi('crm.deal.list', { "STAGE_SEMANTIC_ID": "P", "STAGE_ID": this.selectedStages }, ['TITLE', 'OPPORTUNITY', "CURRENCY_ID", "ID"]);
            const ids = this.deals.map(obj => obj.ID);
            this.sumDeals = this.deals.reduce((accumulator, currentValue) => accumulator + +currentValue.OPPORTUNITY, 0);
            const itterations = Math.ceil(ids.length / 50);
            let invoicesChunk = [];
            let invoices = [];
            for(let i = 0; i < itterations; i++){
              invoicesChunk = await this.callApi('crm.item.list', { "parentId2": ids.slice(i * 50, (i + 1) * 50)}, "", "31");
              invoices.push(invoicesChunk.items);
            }

            this.invoices = invoices.flat();
            let invoicesPayed = 0;
            let invoicesPrepayed = 0;

            this.deals.forEach(item => {
              const currency = this.courses.find(obj => obj.CURRENCY === item.CURRENCY_ID);
              item.OPPORTUNITY = (item.OPPORTUNITY * (currency.AMOUNT / currency.AMOUNT_CNT)).toFixed(2);
              const invoice = this.invoices.find(invoice => invoice.parentId2 === +item.ID); //преобразование к одному типу
              if(invoice){
              const invoiceCurrency = this.courses.find(obj => obj.CURRENCY === invoice.currencyId);
              const invoiceMultplier = invoiceCurrency.AMOUNT / invoiceCurrency.AMOUNT_CNT;
              const invoiceConverted = invoice.opportunity * invoiceMultplier;
              const invoicePrepayed = Math.round(invoiceConverted / 2 * 100) /100;
                if(invoice.stageId === this.invoicesStages.payed) {
                  item.PAYED = invoiceConverted;
                  item.INVOICE_ID = invoice.id;
                  invoicesPayed += invoiceConverted;
                }else if(invoice.stageId === this.invoicesStages.prepayed){
                  item.PAYED = invoicePrepayed;
                  item.INVOICE_ID = invoice.id;
                  invoicesPrepayed += invoicePrepayed;
                }else{
                  item.PAYED = 0;
                }
              }else{
                item.PAYED = 0;
              }
            });
            this.invoicesPayed = invoicesPayed;
            this.invoicesPrepayed = invoicesPrepayed;
            this.filteredDeals = this.deals.filter(item => item.PAYED > 0);
            this.updatePagination();
            this.displayData();


         } catch (error) {
            this.dialog = true;
          }finally{
            this.isLoading = false;
         }
        },
        displayData() {
          this.items = this.isChecked ? this.filteredDeals : this.deals;
        },
        updatePagination() {
          this.pagination.page = 1;
        },
        selectAllStages(){
          if(this.selectedAllStages){
            this.selectedStages = this.stgesIds;
          }else{
            this.selectedStages = [];
          }
        },
        formatNumber(num) {
          const formatedNumber = new Intl.NumberFormat("ru").format(num);
          return formatedNumber + this.currencyTemplate;
        },
        async callApi(method, filter, select, entityTypeId) {
          let total = 0;
          const maxTotal = 50;
          let data = {};
          const params = {};

          if (filter) {
            params.filter = filter;
          }

          if (select.length > 0) {
            params.select = select;
          }
          if (entityTypeId) {
            params.entityTypeId = entityTypeId;
          }

          const exceptions = ['crm.status.list'];
          await new Promise((resolve, reject) => {
            BX24.callMethod(method, params, (res) => {
              if (res.data()) {
                total = res.total();
                data = res.data();
                resolve(data);
              }
            });
          });

          if (total > maxTotal && !exceptions.includes(method)) {
            let cmd = {};
            const iterations = Math.ceil(total / maxTotal);
            let resultData = [];
            for (let i = 0; i < iterations; i++) {
              params.start = i * maxTotal;
                const key = `cmd${i}`;
                const value = {
                method: method,
                params: {
                  select: select,
                  filter: filter,
                  start: i * maxTotal,
                }
                }
                cmd[key] = value;
              if ((i + 1) % maxTotal === 0 || i + 1 === iterations) {
                i > maxTotal ? await new Promise(resolve => setTimeout(resolve, 2000)) : null;
                const batchLength = (i + 1) % maxTotal === 0 ? maxTotal : iterations % maxTotal;
              await new Promise((resolve, reject) => {
              BX24.callBatch(cmd, (res) => {
                for (let r = i - batchLength + 1; r < i + 1; r++) {
                  let key = `cmd${r}`;
                  resultData.push(res[`cmd${r}`].data());
                }
                resultData = resultData.flat();
                data = resultData;
                resolve();
              });
            })
            cmd = {};
              }
            }
          }
          return data;
        },
      },
      async mounted() {

          try {
            const response = await fetch('./invoices_stages.json');
            this.invoicesStages = await response.json();
            this.domain = BX24.getDomain();


            await new Promise((resolve, reject) => {
          BX24.callBatch({
            get_stges: {
              method: 'crm.status.list',
              params: {
                filter: { 'ENTITY_ID': 'DEAL_STAGE' },
                select: ["STATUS_ID", "NAME"],
              }
            },
            get_courses: {
              method: 'crm.currency.list',
              params: {
                filter: {},
                select: ["CURRENCY", "AMOUNT_CNT", "AMOUNT"],
              }
            },
          },(res) => {
            this.stges = res.get_stges.data();
            this.courses = res.get_courses.data();
            this.stges = this.stges.filter(item => item.SEMANTICS === null);
            this.stgesIds = this.stges.map(item => item.STATUS_ID);
            resolve();
          });
        })
         } catch (error) {
           this.dialog = true;
        }finally{
            this.isLoading = false;
        }
      }
  }
</script>

<style lang="sass">
  
  ::-webkit-scrollbar
    background: white
  
  ::-webkit-scrollbar-thumb
    background: #e6e6e6
    border-radius: 1rem
  
  ::-webkit-scrollbar-thumb:active
    background: #e1e1e1
  
  html, body
    min-height: 100%

  h1
    width: 100%
    text-align: center

  .main-container
    display: flex
    flex-direction: column
    gap: 0.75rem
    padding: 0.5rem

  .v-card .v-card-text
    padding-top: 1rem

  .inputs
    display: grid
    grid-template-columns: repeat(auto-fit, minmax(20rem, 1fr))
    column-gap: 1rem
    padding: 0
  
  .buttons
    margin-bottom: 1rem
    display: grid
    grid-template-columns: repeat(4, 1fr)
    gap: 1rem
    width: 100%
  
  .buttons button
    width: 100%
  
  .period
    min-width: 12rem
    max-width: 12rem

  .table-checkbox
    margin-right: 1.25rem

  .charts
    grid-template-columns: repeat(2, 1fr)
    gap: 0.75rem
  
  .chart
    border: 1px solid black
    border-radius: 1rem
  
  .header
    margin: 1.5rem 0
    display: flex
    gap: 1rem
    width: 100%
    justify-content: space-between
    align-items: center
  
  .tfoot__td:not(:first-child)
    text-align: center
  
  [type=number]::-webkit-inner-spin-button
    display: none
  
  .v-input__prepend .v-icon:not(.v-btn__content .v-icon)
    display: none
  
  .v-input__append .v-icon:not(.v-btn__content .v-icon)
    display: none
  
  .v-btn--icon .v-btn__content
    transform: scale(2)
  
  .v-data-table-footer
    display: none
  
  .v-input__details
    display: none
  
  .v-btn.v-btn--density-default
    height: 3rem
  
  .v-form
    width: 100%
    display: grid
    grid-template-columns: 1fr 1fr
    gap: 0.75rem
  
  
  .v-form > *:not(.v-checkbox)
    min-width: 25rem
    flex-grow: 1
  
  .v-checkbox .v-input__control
    min-width: unset
  
  .v-btn--size-default.v-btn
    min-width: unset
    max-width: 20rem

  .v-container--fluid
    max-width: 100%
    display: flex
    flex-direction: column
    gap: 1.5rem

</style>
  
  
  
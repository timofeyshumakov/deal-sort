<template>
    <v-app>
        <div>
          <header class="header">
            <v-form @submit.prevent="handleSubmit">
              <v-text-field
                v-model="inputValue"
                label="Введите текст"
              ></v-text-field>
              <v-btn type="submit" color="primary">Отправить</v-btn>
            </v-form>
            <v-btn color="primary" :title="currencyTitle" @click="changeCurrency">{{currencyTxt}}</v-btn>
            <v-btn color="primary" @click="changeTheme">{{themes[currentTheme]}}</v-btn>
          </header>
          <v-data-table
          :headers="headers"
          :items="paginatedItems"
          :items-per-page="itemsPerPage"
          :pagination.sync="pagination"
          :server-items-length="items.length"
          class="elevation-1"
          :footer-props="{
            'items-per-page-options': [10, 25, 50]
          }"
          >
            <template #top>
              <v-toolbar flat>
                <v-toolbar-title>Сделки пользователя</v-toolbar-title>
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
        </div>
      </v-app>
    </template>
    
    <script>
    
    import { useArrayStore } from './store';
    import { ref } from 'vue';
    
    export default {
    
      setup(){
        const store = useArrayStore();
        const newItem = ref('');
    
        const addItem = (item) => {
            store.addItem(item);
        };
    
        const removeItem = (index) => {
          store.removeItem(index);
        };
    
        const clearItems = () => {
          store.clearItems();
        };
    
        return {
          aetfs: store.aetfs,
          newItem,
          addItem,
          removeItem,
          clearItems,
        };
      },
      data() {
        return {
          store: useArrayStore(),
          items: [],
          opportunities: [],
          currencies: [ 'RUB', 'USD', 'EUR' ],
          currencyIndex: 0,
          currencyTxt: "RUB",
          currencyTitle: "Рубль",
          dealsStatuses: [],
          currentId: null,
          themes: [ 'light', 'dark' ],
          currentTheme: 0,
          pagination: {
            page: 1,
            rowsPerPage: 10,
          },
          itemsPerPage: 10,
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
        updatePagination() {
          this.pagination.page = 1;
        },
        changeTheme(){
          if(this.themes.length < 2){ return; }
          this.currentTheme++;
          if(this.currentTheme === this.themes.length){
            this.currentTheme = 0;
            document.querySelectorAll(`.v-theme--${this.themes[this.themes.length - 1]}`).forEach(el => {
              el.classList.remove(`v-theme--${this.themes[this.themes.length - 1]}`);
              el.classList.add(`v-theme--${this.themes[this.currentTheme]}`);
            });
          }else{
            document.querySelectorAll(`.v-theme--${this.themes[this.currentTheme - 1]}`).forEach(el => {
              el.classList.remove(`v-theme--${this.themes[this.currentTheme - 1]}`);
              el.classList.add(`v-theme--${this.themes[this.currentTheme]}`);
            });
          }
        },
        async changeCurrency() {
    
          if(this.currencies.length < 2){ return; }
          if(this.opportunities.length === 0){ return; }
          
          this.currencyIndex++;
          if(this.currencyIndex === this.currencies.length){
            this.currencyIndex = 0;
          }
    
          const currencyTitles = ['Рубль', 'Доллар', 'Евро'];
          this.currencyTitle = currencyTitles[this.currencyIndex];
          const currentCurrency = this.currencies[this.currencyIndex];
    
          if(!this.store.aetfs[this.currencyIndex]){
            try {
            const response = await fetch('https://www.cbr-xml-daily.ru/daily_json.js');
            if (!response.ok) {
              throw new Error(`HTTP error! status: ${response.status}`);
            }
            const data = await response.json();
            let course;
            
            if(this.currencyIndex !== 0){
              course = data.Valute[currentCurrency].Value;
    
              this.store.addItem(course);
            }
            } catch (error) {
            }
          }
    
    
          this.currencyTxt = currentCurrency;
    
          this.items = this.items.map((item, i) => {
            return { ...item, OPPORTUNITY: (this.opportunities[i] / this.store.aetfs[this.currencyIndex]).toFixed(2) };
          });
    
        },
        handleSubmit() {
          if(this.currentId === this.inputValue){ return; }
          this.currentId = this.inputValue;
    
          let filter;
          if(this.inputValue){
            filter = { 'ASSIGNED_BY_ID': this.inputValue };
          }else{
            filter = {};
          }
    
          BX24.init(() => {
          BX24.callMethod('crm.deal.list', {
            filter: filter,
            select: [ "ID", "TITLE", "STAGE_ID","OPPORTUNITY", "CURRENCY_ID" ],
          },
          (res) => {
          if (res.data()) {
    
            if(this.dealsStatuses.length === 0){
              BX24.callMethod('crm.status.list', {
              filter: { 'ENTITY_ID': 'DEAL_STAGE' }
              },(res) => { 
                  let data = Object.values(res.data());
                  this.dealsStatuses = data;
              });
            }
    
              let total = res.total();
              if( total > 50 ){
                
                if( total > 2500 ){
                  total = 2500;
                }
    
                let batchFilter;
                if(Object.keys(filter).length > 0){
                  batchFilter = 'filter[ASSIGNED_BY_ID]=1';
                }else{
                  batchFilter = null;
                }
    
                let crm = {};
                const iterations = Math.ceil(total / 50);
                for (let i = 0; i < iterations; i++) {
                  const key = `crm${i}`;
                  const value = `crm.deal.list?start=${i * 50}&${batchFilter}&select[]=ID&select[]=TITLE&select[]=STAGE_ID&select[]=OPPORTUNITY&select[]=CURRENCY_ID`;
                  crm[key] = value;
                }
                  BX24.callMethod('batch', {
                  'halt': 0,
                  'cmd': crm
                },(res) => {
                if (res.data()) {
                  let data = res.data();
                  data = Object.values(data.result);
                  this.items = data.flat();
                }
                });
              }else{
                this.items = res.data();
              }
    
              setTimeout(() => {
                this.items = this.items.map((item, i) => {
                if(item.OPPORTUNITY === null){
                  item.OPPORTUNITY = '0.00';
                }
                this.opportunities[i] = item.OPPORTUNITY;
    
                const needle = item.STAGE_ID.replace("C3:", '');
                const matchingData = this.dealsStatuses.find((item) => item.STATUS_ID === needle);
                return {
                  ...item,
                  STAGE_ID: matchingData ? matchingData.NAME : 'Пользовательская стадия',
                };
                });
              }, "1500");
            }
          });
        });
        }
      },
      mounted(){
    
      }
    }
/*
const response = await new Promise((resolve, reject) => {
    BX24.callMethod('crm.deal.list', {
        select: ["ID", "TITLE", "STAGE_ID", "OPPORTUNITY", "CURRENCY_ID"],
    }, (res) => {
        if (res.data()) {
            resolve(res.data());
        } else {
            reject(new Error('No data received'));
        }
    });
});
*/
    </script>
    
    <style lang="sass">
    
    .header
      margin: 1.5rem 0
      display: flex
      gap: 1rem
      width: 100%
      justify-content: space-between
      align-items: center
    
    .v-data-table-footer
      display: none
    
    .v-input__details
      display: none
    
    .v-btn.v-btn--density-default
      height: 3rem
    
    .v-form
      width: 100%
      display: flex
      align-items: center
      height: 3rem
      gap: 2rem
    
    </style>
    
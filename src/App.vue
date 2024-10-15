<template>
  <v-app>
    <div v-if="isLoading">Загрузка...</div>
    <div v-else class="app-container">
      <header class="header">
        <v-form @submit.prevent="handleSubmit">
          <v-text-field
            v-model="inputValue"
            label="Введите текст"
          ></v-text-field>
          <v-btn type="submit" color="primary">Отправить</v-btn>
        </v-form>
        <v-btn
          color="primary"
          :title="currencyTxt"
          @click="changeCurrency"
          >{{currencyTxt}}</v-btn
        >
        <v-btn
          color="primary"
          @click="changeTheme"
          >{{themes[currentTheme]}}</v-btn
        >
      </header>
      <v-data-table
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

export default {

  setup(){
    return {
    };
  },
  data() {
    return {
      isLoading: true,
      currenciesCourses: [1, 1],
      items: [],
      opportunities: [],
      currencies: [ 'Валюта сделки', 'RUB', 'USD', 'KZT' ],
      currencyTitles: ['Валюта сделки', 'Рубль', 'Доллар', 'Тенге'],
      select: ['ID', 'TITLE', 'STAGE_ID', 'CURRENCY_ID', 'OPPORTUNITY'],
      fields: {},
      currencyIndex: 0,
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

      this.currencyTitle = this.currencyTitles[this.currencyIndex];
      const currentCurrency = this.currencies[this.currencyIndex];

      this.currencyTxt = currentCurrency;
      this.items = this.items.map((item, i) => {
        if(this.currencyIndex === 0){
          return { ...item, OPPORTUNITY: this.opportunities[i]};
        }else{
          let currencyItemIndex = this.currencies.indexOf(item.CURRENCY_ID);
          if(currencyItemIndex === this.currencyIndex){
            return { ...item, OPPORTUNITY: this.opportunities[i]};
          }else{
            return { ...item, OPPORTUNITY: (this.opportunities[i] * (this.currenciesCourses[currencyItemIndex] / this.currenciesCourses[this.currencyIndex])).toFixed(2) };
          }
        }
      });

    },
    mapItems(){
      this.items = this.items.map((item, i) => {

        if(item.OPPORTUNITY === null){
          item.OPPORTUNITY = '0.00';
        }

        document.querySelectorAll('th').forEach((title, index) => {
          title.querySelector('span').innerText = this.fields[this.select[index]].title;
        });

        this.opportunities[i] = item.OPPORTUNITY;
        item.CURRENCY_ID
        const needle = item.STAGE_ID.replace("C3:", '');
        const matchingData = this.dealsStatuses.find((item) => item.STATUS_ID === needle);

        return {
          ...item,
          STAGE_ID: matchingData ? matchingData.NAME : 'Пользовательская стадия',
        };

      });
    },
    async handleSubmit() {
      let maxTotal = 50;

      if(this.currentId === this.inputValue){ return; }
      this.currentId = this.inputValue;

      let total;
      let filter;

      if(this.inputValue){
        filter = { 'ASSIGNED_BY_ID': this.inputValue };
      }else{
        filter = {};
      }

      await new Promise((resolve, reject) => {
        BX24.callMethod('crm.deal.list', {
          filter,
          select: this.select,
        },(res) => {
          if (res.data()) {
            total = res.total();
            this.items = res.data();
            this.mapItems();
            resolve(this.items);
          }
        });
      });

      let batchCount = 0;

      if(total > maxTotal){
        let b;
        let iterations = Math.ceil(total / maxTotal);
        batchCount = Math.ceil(iterations / maxTotal);
        let batchBodies = [];

        for (b = 0; b < batchCount; b++) {
          let crm = {};
          for (let i = 0; i < iterations; i++) {
            const key = `crm${i}`;
            const value = {
              method: 'crm.deal.list',
              params: {
                start: b * maxTotal**2 + i * maxTotal,
                filter,
                select: this.select,
              }
            };
            crm[key] = value;
            if(i === maxTotal - 1){ break; }
          }
          batchBodies[b] = crm;
          iterations -= maxTotal;
        }

        b = 0;
        let resultData = [];
        this.mapItems();
        let timerId = setInterval(async() => {
          await new Promise((resolve, reject) => {
            BX24.callBatch(
              batchBodies[b], (res) => {
              let data;
              for( let i = 0; i < Object.keys(batchBodies[b]).length; i++ ){
                let key = `crm${i}`;
                data = res[key].data();
                resultData.push(data);
              }
              resolve(resultData);
            });
          })
          resultData = resultData.flat();
          this.items = resultData;

          this.mapItems();
          b++;
          if(b === batchCount){
            clearInterval(timerId);
          }
        }, 2000);
      }
    }
  },
  async mounted(){

    BX24.callBatch({
      get_statuses: {
        method: 'crm.status.list',
        params: {
          filter: { 'ENTITY_ID': 'DEAL_STAGE' }
        }
      },
      get_fields: ['crm.deal.fields', {}],
    },(res) => {
      this.dealsStatuses = res.get_statuses.data();
      this.fields = res.get_fields.data();
    });

    this.currencyTxt = this.currencies[0];
    this.isLoading = true;
    const url = "https://www.cbr-xml-daily.ru/daily_json.js";
    try {
      const response = await fetch(url);
      const data = await response.json();
      let course;
      let nominal;
      for ( let i = 2; i < this.currencies.length; i++) {
        course = data.Valute[this.currencies[i]].Value;
        nominal = data.Valute[this.currencies[i]].Nominal;
        this.currenciesCourses.push(course / nominal);
      }
    } catch (error) {
      console.error(error.message);
    } finally {
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

.app-container
  padding: 0.5rem

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
  gap: 1rem
</style>

<template>
    <v-app>
      <div>
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
              label="Items per page"
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
      data() {
        return {
          items: [],
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
      },
      async mounted(){
        let total;
        let maxTotal = 50;
  
        await new Promise((resolve, reject) => {
          BX24.callMethod('crm.deal.list', {
            select: ['ID', 'TITLE', 'STAGE_ID', 'CURRENCY_ID', 'OPPORTUNITY'],
          },(res) => {
            if (res.data()) {
              total = res.total();
              this.items = res.data();
              resolve(this.items);
            }
          });
        });
  
        if(total < maxTotal){ return; }
        let crm = {};
        const iterations = Math.ceil(total / maxTotal);
        for (let i = 0; i < iterations; i++) {
          const key = `crm${i}`;
          const value = {
            method: 'crm.deal.list',
            params: {
              start: i * maxTotal,
              select: ['ID', 'TITLE', 'STAGE_ID', 'CURRENCY_ID', 'OPPORTUNITY']
            }
          };
          crm[key] = value;
        }
  
        await new Promise((resolve, reject) => {
          BX24.callBatch(
          crm, (res) => {
            let data;
            let resultData = [];
            for( let i = 0; i < iterations; i++ ){
              let key = `crm${i}`;
              data = res[key].data();
              resultData.push(data);
            }
            resultData = resultData.flat();
            this.items = resultData;
            resolve(resultData);
          });
        })
    }
  }
</script>
  
<style lang="sass">
  
    body::-webkit-scrollbar
      background: white
  
    body::-webkit-scrollbar-thumb
      background: #eee
      border-radius: 1rem
  
    body::-webkit-scrollbar-thumb:active
      background: #e4e4e4
  
    .v-data-table-footer
      display: none
  
    .v-input__details
      display: none
  
    .v-btn.v-btn--density-default
      height: 3rem
  
    .form
      display: flex
      align-items: center
      height: 3rem
      margin: 2rem 0
      gap: 2rem
  
</style>
  
<template>
    <div v-if='searchName.includes("Date") == false'>
      <div v-if='this.config.columns[searchName].type !== "select"' style="display:flex; align-items:center">
        <input  class='table-grid__search' 
                type='text'  
                v-model='searchParam' 
                @keyup='dataChange' 
                @keyup.enter='requestForData' 
                :class='{"isSearchFilled":searchParam !=""}'
          />

          <i v-if='searchParam !==""' @click.prevent='resetField(searchName); searchParam = ""' class='search-reset fe fe-x'/>
      </div>


      <div v-else style="display:flex; align-items:center">
        <select class='table-grid__search' v-model='searchParam' :class='{"isSearchFilled":searchParam !=""}' style="padding: 2px 0;">
          <template v-if='config.columns[searchName]'>
            <option v-for='option in config.columns[searchName].options'>{{option}}</option>
          </template>
        </select>

        <i v-if='searchParam !==""' @click.prevent='searchParam = ""' class='search-reset fe fe-x'/>
      </div>
    </div>

    <div v-else class='vdatetime-picker' style='display: flex; min-width: 200px; align-items: center;'>
       <datetime  
            v-model="dateAfter"
            zone="Europe/Warsaw"
            value-zone="Europe/Warsaw"
            type="date"
            format="yyyy-MM-dd"
            style='margin-right:5px'
        >
          <template slot="button-cancel">
              <span class="btn btn-outline-primary btn-sm"><i class="fe fe-x-square mr-1"></i>Anuluj</span>
          </template>
          <template slot="button-confirm">
              <span class="btn btn-outline-success btn-sm"><i class="fe fe-check-square mr-1"></i>Potwierdź</span>
          </template>
        </datetime>
        -
       <datetime  
            v-model="dateBefore"
            zone="Europe/Warsaw"
            value-zone="Europe/Warsaw"
            type="date"
            format="yyyy-MM-dd"
            style='margin-left:5px'>
          <template slot="button-cancel">
              <span class="btn btn-outline-primary btn-sm"><i class="fe fe-x-square mr-1"></i>Anuluj</span>
          </template>
          <template slot="button-confirm">
              <span class="btn btn-outline-success btn-sm"><i class="fe fe-check-square mr-1"></i>Potwierdź</span>
          </template>
        </datetime>
        <i  v-if='Object.values(this.searchParam).some(item=>item !== null)' 
            @click.prevent='resetField(searchName); searchParam = {after: null, before: null}; dateAfter = null; dateBefore = null' 
            class='search-reset fe fe-x'/>
    </div>
</template>



<script>
  import {EventBus} from "../../app.js";
  import {Datetime} from 'vue-datetime';
  export default {
    props: ["config", 
      "searchColumnName", 
      "allInputObjects", 
      "requestForData", 
      "resetField",
      "itemsPerPage"
    ],
    components:{
      'datetime': Datetime
    },
    data: function() {
      return {
        searchName: "",
        searchParam: "",
        dateAfter: null,
        dateBefore: null,
        justCreated:true
      };
    },
    created: function() {
      for (let key in this.config.columns) {
        if (this.config.columns[key].label == this.searchColumnName) {
          this.searchName = key;
          this.searchParam = this.searchName.includes("Date") ? {} : "";
        }
      }
      let filtersObject = {
          ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
      }
      if(filtersObject && filtersObject.search != undefined && filtersObject.search.find(item=>item.label==this.searchName)){
        this.searchParam = typeof (filtersObject.search.find(item=>item.label==this.searchName).param) != "object" ?
                           filtersObject.search.find(item=>item.label==this.searchName).param :
                           filtersObject.search.find(item=>item.label==this.searchName).param.paramForSessionStorage;
        EventBus.$emit("addSearchInput", {
          label: this.searchName,
          param: this.searchParam
        });
      }
      else {
        EventBus.$emit("addSearchInput", {
          label: this.searchName,
          param: this.searchParam
        });
      }
    },
    methods: {
      dataChange: function(callback) {
        let propCopy = [...this.allInputObjects];
        propCopy.forEach((item, index) => {
          if (item.label == this.searchName) {
            propCopy.splice(index, 1);
            propCopy.push({
              label: this.searchName,
              param: this.searchParam // this.searchColumnName == "Status" ? paramCopy : 
            });
            EventBus.$emit("changeInitialPropObject", propCopy);
          }
        });
      },
      countMonth(date){
          let month;
          switch(new Date(date).getMonth()){
              case 0:
                  month = '01';
                  break;
              case 1:
                  month = '02';
                  break;
              case 2:
                  month = '03';
                  break;
              case 3:
                  month = '04';
                  break;
              case 4:
                  month = '05';
                  break;
              case 5:
                  month = '06';
                  break;
              case 6:
                  month = '07';
                  break;
              case 7:
                  month = '08';
                  break;
              case 8:
                  month = '09';
                  break;
              case 9:
                  month = '10';
                  break;
              case 10:
                  month = '11';
                  break;
              case 11:
                  month = '12';
                  break;
          }
          return month;
      }
    },
    watch:{
      searchParam(){
        if(this.config.columns[this.searchName].type && this.config.columns[this.searchName].type == 'select'){
          EventBus.$emit("columnStatusHasChanged", {searchName: this.searchName, searchParam: this.searchParam});
        }
      },
      dateAfter(){
          if(this.dateAfter){
            this.searchParam.after = this.dateAfter ? 
            `${new Date(this.dateAfter).getFullYear()}-${this.countMonth(this.dateAfter)}-${new Date(this.dateAfter).getDate()}` 
            :
            null;
  
            this.dataChange();
            this.requestForData();
          }
      },
      dateBefore(){
          if(this.dateBefore){
            this.searchParam.before = this.dateBefore ? 
            `${new Date(this.dateBefore).getFullYear()}-${this.countMonth(this.dateBefore)}-${new Date(this.dateBefore).getDate()}` 
            : 
            null;
    
            this.dataChange();
            this.requestForData();
          }
      }
    }
};
</script>



<style lang="scss" scoped>
@import '../../../css/variables';

.table-grid {
    &__search {
      width: 100%;
      max-width: 300px;
      color: $thead-txt-light-color;
      border-radius: 5px;
      border: 1px solid $form-input-border-col;
      padding-left: 10px; 
      margin-right:5px;
    }
    .search-reset, .search-search {
        // width:10px;
        // height:10px;
        // background-image: url("../static/images/icons/actions-general/cross-red.svg");
        // background-repeat: no-repeat;
        // background-position: center;
        // background-size: cover;
        // background-color: transparent;
        border: none;
        cursor: pointer;
        color:red;
        font-size: 17px;
    }
    .search-search {
      color: black;
      margin-right:10px;
      font-size:16px;
    }
    .isSearchFilled {
      border:1px solid #29cac4 !important;
      outline:none;
    }
}
</style>

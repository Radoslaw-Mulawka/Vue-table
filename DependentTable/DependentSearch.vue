<template>
    <div v-if='this.config.columns[searchName].type !== "select"' style="display:flex; align-items:center">
      <input  class='table-grid__search' 
              type='text'  
              v-model='searchParam' 
              @keyup='dataChange' 
              @keyup.enter='requestForData' 
              :class='{"isSearchFilled":searchParam !=""}'
        />

        <i v-if='searchParam !==""'@click.prevent='resetField(searchName); searchParam = ""' class='search-reset fe fe-x'/>
    </div>


    <div v-else style="display:flex; align-items:center">
      <select class='table-grid__search' v-model='searchParam' :class='{"isSearchFilled":searchParam !=""}' style="padding: 2px 0;">
        <template v-if='config.columns[searchName]'>
          <option v-for='option in config.columns[searchName].options'>{{option}}</option>
        </template>
      </select>

      <i v-if='searchParam !==""' @click.prevent='searchParam = ""' class='search-reset fe fe-x'/>
    </div>
</template>



<script>
  import {EventBus} from "../../../app.js";
import { sep } from 'path';

  export default {
    props: ["config", 
            "searchColumnName", 
            "allInputObjects", 
            "requestForData", 
            "resetField",
            "itemsPerPage"
            ],
    data: function() {
      return {
        searchName: "",
        searchParam: ""
      };
    },
    created: function() {
      EventBus.$on('clearAllSearchFields',()=>{
          this.searchParam = "";
          EventBus.$emit('resetSearchFieldWithoutFetching', this.searchName);
      })
      for (let key in this.config.columns) {
        if (this.config.columns[key].label == this.searchColumnName) {
          this.searchName = key;
        }
      }
      let filtersObject = {
          ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
      }
      if(filtersObject && filtersObject.search != undefined && filtersObject.search.find(item=>item.label==this.searchName)){
        this.searchParam = typeof (filtersObject.search.find(item=>item.label==this.searchName).param) != "object" ?
                           filtersObject.search.find(item=>item.label==this.searchName).param :
                           filtersObject.search.find(item=>item.label==this.searchName).param.paramForSessionStorage;
        EventBus.$emit("addSearchInput-dependentTable", {
          label: this.searchName,
          param: this.searchParam
        });
      }
      else {
        EventBus.$emit("addSearchInput-dependentTable", {
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
            EventBus.$emit("changeInitialPropObject-dependentTable", propCopy);
          }
        });
      }
    },
    watch:{
      searchParam(){
        if(this.config.columns[this.searchName].type && this.config.columns[this.searchName].type == 'select'){
          EventBus.$emit("columnStatusHasChanged-dependentTable", {searchName: this.searchName, searchParam: this.searchParam});
        }
      }
    }
};
</script>



<style lang="scss">
@import '../../../../css/variables';
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

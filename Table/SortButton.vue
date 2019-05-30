<template>
    <button @click='sortColumn(columnName)' class='table-grid__sort-button initial-arrows' :class='{
      "initial-arrows": initialArrows == true,
       "sort-button--asc" : sortDirection == "asc" && initialArrows == false,
       "sort-button--desc" : sortDirection == "desc" && initialArrows == false
    }' :disabled='isUpdated'></button>
    <!-- "initial-arrows": justMounted == true,
    && justMounted == false
    && justMounted == false -->
</template>


<script>
import { EventBus } from "../../app.js";
export default {
  props: [
    "config", 
    "columnName", 
    "itemsPerPage",
    "newSortDirection",
    "englishSortColumn"
  ],
  data: function() {
    return {
      sortDirection: "asc",
      initialArrows: true,
      isUpdated: false
    };
  },
  created(){
    EventBus.$on('startUpdating',()=>{
      this.isUpdated = true;
    })
    EventBus.$on('stopUpdating',()=>{
      this.isUpdated = false;
    })
    let englishKey;
    if(this.newSortDirection){
      for (let objectKey in this.config.columns) {
        if (this.config.columns[objectKey].label == this.columnName) {
          englishKey = objectKey;
        }
      }
      if(englishKey == this.englishSortColumn){
        this.sortDirection = this.newSortDirection;
        this.initialArrows = false;
      }
    }

    EventBus.$on('newSortDirectionHasCome', (newData)=>{
      let englishKey;
      for (let objectKey in this.config.columns) {
        if (this.config.columns[objectKey].label == this.columnName) {
          englishKey = objectKey;
        }
      }
      if(englishKey == this.englishSortColumn){
        this.sortDirection = newData;
        this.initialArrows = false;
      }
    })
    EventBus.$on('resetOtherSortButtons',(columnName)=>{
      if(columnName !== englishKey){
        this.initialArrows = true;
      }
      else {
        this.initialArrows = false;
      }
    })
  },
  methods: {
    sortColumn: function(key) {
      let englishKey;
      for (let objectKey in this.config.columns) {
        if (this.config.columns[objectKey].label == key) {
          englishKey = objectKey;
        }
      }

      EventBus.$emit('sortChanged', {
        columnName: englishKey,
        sortDirection: this.sortDirection == 'asc' ? 'desc' : 'asc'
      });
      EventBus.$emit('resetOtherSortButtons', englishKey)
    }
  },
  watch:{
    newSortDirection(){
      let englishKey;
      for (let objectKey in this.config.columns) {
        if (this.config.columns[objectKey].label == this.columnName) {
          englishKey = objectKey;
        }
      }
      if(englishKey == this.englishSortColumn){
        this.sortDirection = this.newSortDirection
        this.initialArrows = false;
      }
    }
  }
};
</script>


<style lang='scss'>
.table-grid {
  &__sort-button {
    background-color: transparent;
    width: 15px;
    height: 15px;
    cursor: pointer;
    
    background-repeat: no-repeat;
    background-size: contain;
    background-position: center left, center right;
    border: none;
  }

  .initial-arrows {
    background-image:  url("../static/images/icons/actions-general/arrow-up.png"), url("../static/images/icons/actions-general/arrow-down.png");
  }
  .sort-button--asc {
      background-image: url("../static/images/icons/actions-general/arrow-up.png"), url("../static/images/icons/actions-general/arrow-down-active.png");
  }
  .sort-button--desc {
      background-image: url("../static/images/icons/actions-general/arrow-up-active.png"), url("../static/images/icons/actions-general/arrow-down.png");
  }
}
</style>

<template>
    <div class='table-container table-responsive' style='display: grid'>
            <vue-progress-bar></vue-progress-bar>
            <div :class='{"tool-bar":true,  "tool-bar--sticky": itemsPerPage>50}'>
                <div class="table-pagination__show-select">
                    <select v-model='itemsPerPage'>
                        <option value="25">25 <span>na stronę</span></option>
                        <option value="50">50 <span>na stronę</span></option>
                        <option value="100">100 <span>na stronę</span></option>
                        <option value="250">250 <span>na stronę</span></option>
                        <option value="500">500 <span>na stronę</span></option>
                        <option value="1000">1000 <span>na stronę</span></option>
                    </select>
                </div>
                <Pagination v-if='pagination !=null && pagination["hydra:last"]' 
                            :config='config' 
                            :pagination='pagination' 
                            :top='true' 
                            :itemsPerPage='itemsPerPage'>
                </Pagination> 
                <span v-if='config.exportToExcel == true'></span>
                <div v-if='config.exportToExcel == true' class="export">
                    <a @click='exportToExcel'><i class="icon-xls"></i> Eksport do Excela</a>
                </div>

            </div>
            <table class='content__table content__table--white' style="height:1%">
                <Thead :config='config'
                        :fieldsWithSearch='fieldsWithSearch'
                        :fieldsWithSorting='fieldsWithSorting'
                        :pagination='pagination'
                        :itemsPerPage='itemsPerPage'
                        >
                </Thead>
                <Tbody :finalTableData='finalTableData' 
                        :config='config' 
                        :originalData ='originalData'>
                </Tbody>
            </table>

            <div :class='{"tool-bar":true, "tool-bar--bottom": true}' style='justify-content:center'>
                <div class="table-pagination__show-select">
                    <select v-model='itemsPerPage'>
                        <option value="25">25 <span>na stronę</span></option>
                        <option value="50">50 <span>na stronę</span></option>
                        <option value="100">100 <span>na stronę</span></option>
                        <option value="250">250 <span>na stronę</span></option>
                        <option value="500">500 <span>na stronę</span></option>
                        <option value="1000">1000 <span>na stronę</span></option>
                    </select>
                </div>
                <Pagination v-if='pagination !=null && pagination["hydra:last"]' 
                            :config='config' 
                            :pagination='pagination' 
                            :top='true' 
                            :itemsPerPage='itemsPerPage'>
                </Pagination> 
            </div>
    </div>
</template>


<script>
    import Thead from './Thead.vue';
    import Tbody from './Tbody.vue';
    import Pagination from "./Pagination.vue";
    import {EventBus} from "../../app.js";

    export default {
        props: ["conf"],
        components: {
        Thead,
        Tbody,
        Pagination
    },
    data: function() {
        return {
            config: null,
            tableData: [],
            finalTableData: [{'isLoading': true}],
            pagination: null,
            fieldsWithSearch:[],
            fieldsWithSorting:[],
            itemsPerPage:25,
            originalData:null
        };
    },
    created: function() {
        EventBus.$on('callGetDataOnCreation',(itemsPerPage)=>{
            this.itemsPerPage = itemsPerPage;
            this.getDataOnCreation();
        });
        EventBus.$on('setFinalTableDataToNull',()=>{
            this.finalTableData = null;
        });
        
        if (this.conf != null) {
          this.config = this.conf;
        } else {
          throw new Error({'GridConfig':"Can't find config file"})
        }

        EventBus.$on("paginationAction", passedData => {
            let filtersObject = JSON.parse(sessionStorage.getItem(this.conf.tableName+'-'+'filters'));
            if(filtersObject && filtersObject.selectedRow !== undefined){
                delete filtersObject.selectedRow;
                sessionStorage.setItem(this.conf.tableName+'-'+'filters', JSON.stringify(filtersObject));
            }
            EventBus.$emit('resetRowColouring');
            this.tableData = passedData["hydra:member"];
            this.pagination = passedData["hydra:view"];
        });

        if(this.config.stateManager.saveSorting == true){
            let filtersObject = JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters'));

            if(filtersObject && filtersObject.sort !== undefined){
                this.sortColumn = filtersObject.sort.sortColumn;
                this.sortDirection = filtersObject.sort.sortDirection;
            }
        }

        this.getDataOnCreation();
    },
    methods: {
        exportToExcel(){
            let jsonToSend = {
                columns: {},
                rows:{}
            };

            for(let columnName in this.config.columns){
                if(columnName !== 'actions' && this.config.columns[columnName]["areShown"] == true){
                    jsonToSend.columns[columnName] = this.config.columns[columnName].label
                }
            }
            for(let [index,rowObject] of this.finalTableData.entries()){
                jsonToSend.rows[index] = {};
                for(let prop in rowObject){
                    if(rowObject[prop]["areShown"] == true){
                        jsonToSend.rows[index] = {
                            ...jsonToSend.rows[index],
                            [prop]: rowObject[prop].value ? rowObject[prop].value : ''
                        }
                    }
                }
            }
            let fileName = '';
            let date = new Date();
            let year = date.getFullYear();
            let month = date.getMonth() + 1;
            let day = date.getDate();
            let hours = date.getHours();
            let minutes = date.getMinutes();
            let seconds = date.getSeconds();

            this.$Progress.start();
            fetch(`/api/generate/xlsx`,{
                method:'POST',
                body: JSON.stringify(jsonToSend),
                headers:{
                    'Accept': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
                }
            }).then(response=>{
                return response.blob();
            }).then(blob=>{
                this.$Progress.finish();
                var blob = new Blob([blob], { type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;' });
                var link = document.createElement('a');
                link.href = window.URL.createObjectURL(blob);
                link.download = year.toString() + month.toString() + day.toString() + hours.toString() + minutes.toString() + seconds.toString();

                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
            })
        },
        getDataOnCreation: function() {
            let filtersObject = JSON.parse(sessionStorage.getItem(this.conf.tableName+'-'+'filters'));
            if(this.config.stateManager.saveFilter == false && filtersObject && filtersObject.search){
                delete filtersObject.search;
                sessionStorage.setItem(this.conf.tableName+'-'+'filters', JSON.stringify(filtersObject));
            }
            let url = this.config.urlAPI;
                url+= `&itemsPerPage=${this.itemsPerPage}`;
            
            if(filtersObject !== null && (Object.entries(filtersObject).length>0) ){
                for(let key in filtersObject){
                    if(key == 'search'){
                        filtersObject[key].map(item=>{
                            if(typeof item.param != 'object'){
                                url += `&${item.label}=${item.param}`;
                            }
                            else {
                                url += `&${item.label}=${item.param.urlNumber}`;
                            }
                        })
                    }
                    else if(key == 'selectedRow') {
                        continue;
                    }
                    else if(key == 'sort'){
                        url+= `&order[${filtersObject[key].sortColumn}]=${filtersObject[key].sortDirection}`
                    }
                    else {
                        url += `&${filtersObject[key]}`
                    }

                }
            }

            this.$Progress.start();
            fetch(url)
                .then(response => {
                    return response.json();
                })
                .then((data) => {
                    this.$Progress.finish();
                    this.fieldsWithSearch = [];
                    this.fieldsWithSorting = [];
                    
                    if (this.config.arrayCollection && this.config.arrayCollection.length > 0) { // what is it?????????????
                        this.tableData = data[this.config.arrayCollection];
                    }

                    // part responsible for sorting and filtering depending on what has been got from API
                    if (data["hydra:search"]) {
                        let mapping = data["hydra:search"]["hydra:mapping"];

                        // declare what columns should be sortable / filterable
                        for (let valueObject of mapping) {
                            if (valueObject.property == valueObject.variable) {
                                this.fieldsWithSearch.push(valueObject.property)
                            }
                            if(valueObject.property.includes("Date")){
                                this.fieldsWithSearch.includes(valueObject.property) ? null : this.fieldsWithSearch.push(valueObject.property)
                            }
                            if (valueObject.variable.includes("order")) {
                                this.fieldsWithSorting.push(valueObject.property)
                            }
                        }
                    }


                    if(data["hydra:member"].length>0){
                        this.tableData = data["hydra:member"];
                        this.pagination = data["hydra:view"];
                    }
                    else {
                        this.finalTableData = [{'lackOfData': true}]
                    }  
                });

        }, // get data from api and put it to this.tableData
        // DRY CODE

        getNestedObject(key,myObj){
            const getNestedObject = (nestedObj, pathArr) => {
                return pathArr.reduce((obj, key) =>
                    (obj && obj[key] !== 'undefined') ? obj[key] : undefined, nestedObj);
            };

            let link = key.split('.'); // materialType.name = ['materialType', 'name']
            const nestedObject = getNestedObject(myObj, link);
            return nestedObject;
        },
        transformTableData(columns){
            let tableDataCopy = [];
            for (let key in columns) {  // key = actions, id, isSpecial (english names of columns)
                this.tableData.map(myObj => {

                    if(tableDataCopy.find((element, index)=>{
                        let elementId = element['@id'].split('/');
                        let myObjId = myObj['@id'].split('/');
                        return elementId.reverse()[0] == myObjId.reverse()[0];
                    }) == undefined){
                        tableDataCopy.push({...myObj})
                    }

                    if (key.includes('.') === true) {
                        myObj[key] = this.getNestedObject(key, myObj);
                    }
                    if (columns[key].fun) {
                        myObj[key] = columns[key].fun(myObj); // if column.name object has 'fun' inside it, than myObj[name] will be equal to result of 'fun', to which you passed current myObj.
                    }                                         // myObj[key] содержит результат выполнения функции с переданным ей объектом myObj.
                    myObj[key] = {
                        areShown: columns[key].areShown,
                        label: columns[key].label,
                        value: myObj[key]
                    };
                });
            }
            
            this.originalData = tableDataCopy;
        },
        fillFinalTableData(columns){
            this.finalTableData = [];
            if(this.tableData.length>0){
                for (let element of this.tableData) {
                    let newObj = {};
                    for (let column in columns) {
                        if (column !== 'actions') {
                            newObj[column] = {
                                areShown: element[column].areShown,
                                value: element[column].value
                            };
                        }
                    }
                    this.finalTableData.push(newObj);
                }
            }
            else {
                this.finalTableData = [{"lackOfData":true}]
            }
        },
        loadDataFromSessionStorage(){
            let filtersObject = JSON.parse(sessionStorage.getItem(this.conf.tableName+'-'+'filters'));
            if(this.config.stateManager.saveClickedRow == true){
                if(filtersObject !== null  && (Object.entries(filtersObject).length>0) ){
                    let indexInSessionStorage = filtersObject['selectedRow']; // 3
                    let object = this.finalTableData[indexInSessionStorage];
                    EventBus.$emit('fireShowRowDetails',{object,indexInSessionStorage});
                }
            }
        },
    },
    updated(){
        this.config.stateManager.saveFilter || 
        this.config.stateManager.saveSorting || 
        this.config.stateManager.saveClickedRow ? this.loadDataFromSessionStorage() : null;//6
    },
    watch: {
        tableData: function(newValue, oldVale) {
            
            let columns = this.config.columns; // { actions: {}, id: {}, name:{}, isActive:{}, isSpecial:{} }
            this.transformTableData(columns); //3
            this.fillFinalTableData(columns); //5
            this.config.stateManager.saveFilter || 
            this.config.stateManager.saveSorting || 
            this.config.stateManager.saveClickedRow ? this.loadDataFromSessionStorage() : null;//6
        },
        itemsPerPage(){
            this.getDataOnCreation();
        }
    }
};
</script>

<style lang="scss">
@import '../../../css/variables';
.status-span {
    color: $status-col;
}
#table-component {
    padding:0;
}
@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
.no-data {
    text-align: center;
    border:none;
    &:hover {
        background-color:transparent !important;
        
    }
    td,th{
        border:none !important;
        cursor: default;
    }
}
.loader {
  border: 5px solid #f3f3f3;
  border-top: 5px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 2s linear infinite;
  margin: 0 auto;
}
.table-container {
  background-color: white;
//   border-radius: 5px;
  width: 100%;
//   padding: 30px 0;
//   padding-bottom:20px;
//   overflow-x:auto;
}

.content{
    &__table {
        text-align: left;
        border-collapse: collapse;
        width: 100%;
        font-family: 'Quicksand medium';
        .table-grid {
            display:grid;
            grid-template-columns: 1fr;
            grid-template-rows: auto 25px;
        }
        .table-grid-one-row {
            grid-template-rows: auto;
        }
        thead {
            color: $thead-txt-dark-color;
            font-size: 13px;
            tr {
                vertical-align: top;
            }
        }
        
        tbody {
            color: $tbody-txt-col;
            font-size: 14px;
            @media screen and (max-width:1280px){
                font-size: 12px;
            }
        }
        tbody tr:nth-child(even){
            background-color: #f9f9f9;
            &:hover {
              background-color: darken(#f9f9f9,10%);
            }
        }
        
        tr:first-child {
            // usuwam - teraz tabele się łamią
            // white-space: nowrap;
        }

        tr {
            // height: 50px;
            padding-top: 25px;
            padding-bottom: 25px;

            &:hover {
                background-color: #ebebeb;
                cursor: pointer;
            }
        }

        td,
        th {
            border-bottom: 1px solid #d8d8d8;
            padding-left: 15px;
            padding-right: 15px;
            padding-top: 15px;
            padding-bottom: 15px;
            &:first-child{
                // width: 50px;
                // text-align: center;
                vertical-align: middle;
                position: relative;
                &>input[type='checkbox']{
                    position: absolute;
                    left: 50%;
                    top: 50%;
                    transform: translate(-50%,-50%);
                    height: 30px;
                    width: 30px;

                    -webkit-appearance: none;
                    -moz-appearance: none;
                    appearance: none;
                    outline: 0;
                    background: transparent;
                    border: 1px solid lightgray;
                    border-radius: 5px;
                }
                &>input[type="checkbox"]:checked {
                    background: black;
                  }
                  &>input[type="checkbox"]:after {
                    content: '';
                    position: relative;
                    left: 40%;
                    top: 50%;
                    width: 15%;
                    height: 40%;
                    border: solid #fff;
                    border-width: 0 2px 2px 0;
                    transform: translate(0,-50%) rotate(45deg);
                    display: none;
                  }
                  
                  &>input[type="checkbox"]:checked:after {
                    display: block;
                  }
                  
                  &>input[type="checkbox"]:disabled:after {
                    border-color: #7b7b7b;
                  }
            }
            &:first-child:not(td){
                vertical-align: top;
            }
        }
        .all-actions {
            position: static !important;
            transform: none !important;
        }
    }
}

.tool-bar {
    // display:flex;
    // justify-content: space-between;
    // align-items: center;
    border-bottom:1px solid #b7d1d4;
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
    align-items: center;
    padding: 20px;
    background-color: $grid-toolbar-bg;
    width:100%;
    -webkit-box-shadow: inset 0px -8px 5px -6px rgba(0,0,0,0.05);
    -moz-box-shadow: inset 0px -8px 5px -6px rgba(0,0,0,0.05);
    box-shadow: inset 0px -8px 5px -6px rgba(0,0,0,0.05);
    &--sticky {
        position: sticky;
        top: -1px;
        z-index: 10;
    }
    &--bottom {
        // border-top:1px solid #b4d6db;
        border-bottom: 0;
        -webkit-box-shadow: inset 0px 8px 5px -6px rgba(0,0,0,0.05);
        -moz-box-shadow: inset 0px 8px 5px -6px rgba(0,0,0,0.05);
        box-shadow: inset 0px 8px 5px -6px rgba(0,0,0,0.05);
    }
    select {
        background-color: transparent;
    }
}
.export {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    i {
        font-size:30px;
        margin-right:10px;
    }
    a {
        display: flex;
        align-items: center;
        color: $grid-toolbar-btn !important;
        cursor: pointer;
    }
}
</style>


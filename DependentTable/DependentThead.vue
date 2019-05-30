<template>
    <thead>
        <tr>
            <th  v-for="(value,key) in headerData" v-if="value.areShown === true" :key='key'>
                <div class='table-grid' :class='{"table-grid-one-row": value.search == false || value.search == undefined}'>
                    <div :class='{"th-action" : key=="actions" ? true: false}' style='marginBottom:10px; display: flex; align-items: center; white-space: nowrap;'>
                        <span style='marginRight:5px;'>{{value.label}}</span>
                        <DependentSortButton :config='config' 
                                    v-if='(config.columns[key].order == true) && (fieldsWithSorting.includes(key))' 
                                    :columnName='value.label' 
                                    :itemsPerPage='itemsPerPage'>
                        </DependentSortButton>
                    </div>
                    <DependentSearch  
                        :config='config'
                        v-if='config.columns[key].search == true && fieldsWithSearch.includes(key)'
                        :searchColumnName='value.label'
                        :allInputObjects='allInputObjects'
                        :requestForData='requestForData'
                        :resetField = resetField
                        :itemsPerPage='itemsPerPage'
                    >
                    </DependentSearch>
                </div>
            </th>
        </tr>
    </thead>
</template>

<script>
import DependentSearch from './DependentSearch';
import DependentSortButton from './DependentSortButton';
import { EventBus } from "../../../app.js";
import { parse } from 'querystring';
export default {
    components:{
        DependentSearch,
        DependentSortButton,
    },
    props:[
        'config',
        'fieldsWithSorting',
        'fieldsWithSearch',
        'pagination',
        'itemsPerPage',
        'clickedObject'
    ],
    data: function(){
        return {
            headerData: {},
            queryString: "",
            allInputObjects: [], 
            sortDirection: null,
            sortColumn: null
        }
    },
    created(){
        EventBus.$on("addSearchInput-dependentTable", inputObject => {
            this.allInputObjects.push(inputObject);
        });

        EventBus.$on("changeInitialPropObject-dependentTable", dataForReplacement => {
            this.allInputObjects = dataForReplacement.reverse();
        });
        EventBus.$on("columnStatusHasChanged-dependentTable", ({searchName, searchParam})=>{
            let paramCopy = searchParam;
            if(searchParam != ""){
                let optionsArray = ["aktywn", "zatwierdzon", "zatwierdzony automatycznie","opublikowan","specjal"];
                let transformSearchParam = searchParam.toLowerCase();
                for(let option of optionsArray){
                    if(transformSearchParam.startsWith(option)){
                        paramCopy = {urlNumber: 1, paramForSessionStorage: searchParam};
                        break;
                    }
                    else {
                        paramCopy = {urlNumber: 0, paramForSessionStorage: searchParam};
                    }
                }
            }

            let propCopy = [...this.allInputObjects];
            propCopy.forEach((item, index) => {
                if (item.label == searchName) {
                    propCopy.splice(index, 1);
                    propCopy.push({
                        label: searchName,
                        param: paramCopy
                    });
                }
            });
            
            
            this.queryString = "";
            propCopy.forEach((item, index) => {
                if (item.param !== "" && (typeof item.param) !== 'object' ) {
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param;
                    }
                }
                else if (item.param !== "" && (typeof item.param) == 'object'){
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param.urlNumber;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param.urlNumber;
                    }
                }
            });


            let indexInString = this.pagination ? this.pagination["@id"].lastIndexOf('page=') : null;
            let currentPageNumber = this.pagination ? this.pagination["@id"].slice(indexInString+1) : null;
            this.requestForData(currentPageNumber, ()=>{
                this.allInputObjects = propCopy;
            });

        });
        EventBus.$on("sortChanged-dependentTable", ({columnName, sortDirection})=>{
            let indexInString = this.pagination["@id"].lastIndexOf('page=');
            let currentPageNumber = this.pagination["@id"].slice(indexInString+1) ;
            
            this.sortDirection = sortDirection;
            this.sortColumn = columnName;

            this.requestForData(currentPageNumber);
        });
        EventBus.$on("resetSearchFieldWithoutFetching",(columnName)=>{
            let inputObject = this.allInputObjects.findIndex( (inputObject,index)=>{
                return inputObject.label == columnName;
            });
            let copyOfAllInputObjects = [...this.allInputObjects];

            copyOfAllInputObjects[inputObject].param = "";

            this.allInputObjects = [...copyOfAllInputObjects];
        })
        this.fillHeaderData();
    },
    methods: {
        fillHeaderData(){
            for(let columnName in this.config.columns){
                this.headerData[columnName] = {  // {id: {…}, name: {…}, shortcut: {…}, isSpecial: {…}, isActive: {…}, …}
                    ...this.config.columns[columnName]
                };
            }
        },
        requestForData: function(currentPageNumber, callback) {
            let url;
            let currentPage = currentPageNumber != false && parseInt(currentPageNumber) ? currentPageNumber : '';
            if (this.queryString == "") {
                if(this.sortColumn != null && this.sortDirection != null){
                    url =  `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                    url += `${this.config.urlAPI.includes('context=') ? "&" : "?"}`;
                    url += `order[${this.sortColumn}]=${this.sortDirection}`;
                    url += `&itemsPerPage=${this.itemsPerPage}`;
                    url += `${currentPage ? "&page="+currentPage : ''}`;        
                }
                else {
                    url = `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                    url +=`&itemsPerPage=${this.itemsPerPage}`;
                }
                this.$Progress.start();

                fetch(url)
                    .then(response => {
                        return response.json();
                    })
                    .then(data => {
                        this.$Progress.finish();
                        callback ? callback() : null;
                        let filtersObject = {
                            ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
                            search: this.allInputObjects.filter(item=>{
                                return item.param !='';
                            })
                        };

                        if(this.config.stateManager.saveFilter){
                            sessionStorage.setItem(this.config.tableName+'-'+'filters', JSON.stringify(filtersObject));
                        }
                        EventBus.$emit('paginationAction-dependentTable', data);
                        EventBus.$emit("resetOtherSortButtons-dependentTable", {columnName:this.sortColumn,sortDirection:this.sortDirection});
                        if(data["hydra:member"].length <= 0){ 
                            EventBus.$emit('setFinalTableDataToNull-dependentTable');
                        }
                    });

            } else {
                
                if (this.queryString[0] == "&") {
                    this.queryString = this.queryString.substr(1, this.queryString.length);
                }
                if(this.sortColumn != null && this.sortDirection != null){
                    url = `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                    url+= `${this.config.urlAPI.includes('context=') ? "&" : "?"}`;
                    url+= `${this.queryString}`;
                    url+= `&order[${this.sortColumn}]=${this.sortDirection}`;
                    url+= `${currentPage ? "&page="+currentPage : ''}`;
                    url+= `&itemsPerPage=${this.itemsPerPage}`;
                }
                else {
                    url = `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                    url+= `${this.config.urlAPI.includes('context=') ? "&" : "?"}`;
                    url+= `${this.queryString}`;
                    url+= `&itemsPerPage=${this.itemsPerPage}`;
                }
                this.$Progress.start();

                fetch(url)
                    .then(response => {
                        return response.json();
                    })
                    .then(data => {
                        this.$Progress.finish();
                        callback ? callback() : null;
                        let filtersObject = {
                            ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
                            search: this.allInputObjects.filter(item=>{
                                return item.param !=''
                            })
                        }
                        if(this.config.stateManager.saveFilter){
                            sessionStorage.setItem(this.config.tableName+'-'+'filters', JSON.stringify(filtersObject));
                        }
                        EventBus.$emit('paginationAction-dependentTable', data);
                        EventBus.$emit("resetOtherSortButtons-dependentTable", {columnName:this.sortColumn,sortDirection:this.sortDirection});
                        if(data["hydra:member"].length <= 0){
                            EventBus.$emit('setFinalTableDataToNull-dependentTable');
                        }
                    });
            }
        },
        resetField(columnName){
            let sortingParams;
            if(this.sortDirection !=null && this.sortColumn !=null){
                sortingParams = `order[${this.sortColumn}]=${this.sortDirection}`;
            }
            let currentColumnObject = this.allInputObjects.find(item=>{
                return item.label == columnName;
            })
            typeof currentColumnObject.param == "object" ? currentColumnObject.param.urlNumber = '' :  currentColumnObject.param = '';

            this.queryString = "";
            this.allInputObjects.forEach((item, index) => { 
                if (item.param != false && (typeof item.param) !== 'object' ) {
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param;
                    }
                }
                else if (item.param != false && (typeof item.param) == 'object'){
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param.urlNumber;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param.urlNumber;
                    }
                }
            });

            if (this.queryString == "") {
                let url;
                if (this.queryString[0] == "&") {
                    this.queryString = this.queryString.substr(1, this.queryString.length);
                }
                this.$Progress.start();

                url = `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                url +=`&itemsPerPage=${this.itemsPerPage}`;
                url +=`${sortingParams ? '&'+sortingParams : ''}`;
                fetch( url )
                    .then(response => {
                        return response.json();
                    })
                    .then(data => {
                        this.$Progress.finish();
                        let filtersObject = {
                            ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
                            search: this.allInputObjects.filter(item=>{
                                return item.param !='';
                            })
                        }
                        sessionStorage.setItem(this.config.tableName+'-'+'filters',JSON.stringify(filtersObject));
                        
                        EventBus.$emit('paginationAction-dependentTable', data);
                        if(data["hydra:member"].length <= 0){
                            EventBus.$emit('setFinalTableDataToNull-dependentTable');
                        }
                    });
            } 
            else {
                let url;
                if (this.queryString[0] == "&") {
                    this.queryString = this.queryString.substr(1,this.queryString.length);
                }
                this.$Progress.start();

                url= `${this.config.urlAPI + this.clickedObject.uuid.value}`;
                url+=`${this.config.urlAPI.includes('context=') ? "&" : "?"}`;
                url+=`${this.queryString}`;
                url+=`&itemsPerPage=${this.itemsPerPage}`;
                url+=`${sortingParams ? '&'+sortingParams : ''}`;
                fetch( url )
                    .then(response => {
                        return response.json();
                    })
                    .then(data => {
                        this.$Progress.finish();
                        let filtersObject = {
                            ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
                            search: this.allInputObjects.filter(item=>{
                                return item.param !='';
                            })
                        }
                        sessionStorage.setItem(this.config.tableName+'-'+'filters',JSON.stringify(filtersObject));
                        
                        EventBus.$emit('paginationAction-dependentTable', data);
                        if(data["hydra:member"].length <= 0){
                            EventBus.$emit('setFinalTableDataToNull-dependentTable');
                        }
                    });
            }
        },
    },
    watch:{
        allInputObjects: function() {
            this.queryString = "";
            this.allInputObjects.forEach((item, index) => {
                if (item.param != false && (typeof item.param) !== 'object' ) {
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param;
                    }
                }
                else if (item.param != false && (typeof item.param) == 'object'){
                    if (index == 0) {
                        this.queryString += item.label + "=" + item.param.urlNumber;
                    } else {
                        this.queryString += "&" + item.label + "=" + item.param.urlNumber;
                    }
                }
            });

        }
    }
}
</script>


<style lang="scss" scoped>
.th-action {
    justify-content: center;
    &>span {
        margin-right:0 !important;
    }
}
</style>

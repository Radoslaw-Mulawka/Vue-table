<template>
    <tr   :class='{ selectedRow: selectedRow && oneObject.isLoading == undefined && oneObject.lackOfData == undefined}'>
    
        <DependentDataCell v-if='Object.keys(oneObject).length>1 && value.areShown == true' 
                  v-for='(value,index2) in oneObject'
                  :oneObject='oneObject'
                  :index='index'
                  :key='index2'
                  :columnName='index2'
                  :config='config'
                  :value='value'
                  :showRowDetails='showRowDetails'>
        </DependentDataCell>

        <TableActions v-if='config.columns.actions.areShown == true && Object.keys(oneObject).length>1' 
                        :oneObject='oneObject' 
                        :config='config'
                        :originalData='originalData'
                        :is_granted_menu_edit='is_granted_menu_edit'
                        :is_granted_menu_show='is_granted_menu_show'
                        :is_granted_menu_pdf='is_granted_menu_pdf'
                        >
        </TableActions>

    </tr>
</template>

<script>
    import TableActions from "../TableActions.vue";
    import {EventBus} from "../../../app.js";
    import DependentDataCell from './DependentDataCell.vue';

    export default {
    props:[
        'oneObject',
        'index',
        'config',
        'originalData',
        'is_granted_menu_edit',
        'is_granted_menu_show',
        'is_granted_menu_pdf',
    ],
    components: {
        TableActions,
        DependentDataCell,
    },
    data: function(){
        return {
            selectedRow: false
        }
    },
    created(){
        EventBus.$on('resetRowColouring-dependentTable',()=>{
            this.selectedRow = false;
        });
        
        EventBus.$on('fireShowRowDetails-dependentTable',(data)=>{
            this.showRowDetails(data.object, data.indexInSessionStorage)
        });

        let filtersObject = {
            ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
        };
        if(this.config.stateManager.saveClickedRow == false){
            filtersObject != null && filtersObject['selectedRow'] != undefined ? delete filtersObject['selectedRow'] : null;
            sessionStorage.setItem(this.config.tableName+'-'+'filters',JSON.stringify(filtersObject));
        }
    },
    methods:{
        showRowDetails: function(currentObject,indexInSessionStorage, event) {
            if(this.config.stateManager.saveClickedRow){
                let filtersObject = {
                    ...JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters')),
                    ['selectedRow']: indexInSessionStorage
                };
                let sessionStorageRow = JSON.parse(sessionStorage.getItem(this.config.tableName+'-'+'filters'));

                if(event == undefined){
                    if(indexInSessionStorage == this.index){
                        this.selectedRow = true;
                        EventBus.$emit("getRowDetails", currentObject);
                    }
                }
                else {
                    if(sessionStorageRow !== null && sessionStorageRow['selectedRow'] == this.index){
                        EventBus.$emit('resetRowColouring-dependentTable');
                        EventBus.$emit("getRowDetails", null);
                        delete sessionStorageRow['selectedRow'];
                        sessionStorage.setItem(this.config.tableName+'-'+'filters',JSON.stringify(sessionStorageRow));
                    }
                    else {
                        EventBus.$emit('resetRowColouring-dependentTable');
                        this.selectedRow = true;
                        EventBus.$emit("getRowDetails", currentObject);
                        sessionStorage.setItem(this.config.tableName+'-'+'filters',JSON.stringify(filtersObject));
                    }
                }
            }
            else {
                EventBus.$emit("getRowDetails", currentObject);
            }
        }
    }
}
</script>


<style lang="scss">
@import '../../../../css/variables';
.selectedRow {
        background-color: $nav-bg-color !important;
        color:white;
        .status-span {
            color: #00e09a !important;
        }
}
</style>

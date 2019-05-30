<template>
    <tbody>

        <DependentTrow v-if='oneObject.isLoading == undefined && oneObject.lackOfData == undefined && clickedObject != null' 
            v-for='(oneObject, index) in finalTableData' 
            :key='index' 
            :oneObject='oneObject' 
            :index='index' 
            :config='config'
            :originalData='originalData'
            :is_granted_menu_edit='is_granted_menu_edit'
            :is_granted_menu_show='is_granted_menu_show'
            :is_granted_menu_pdf='is_granted_menu_pdf'
        >
        </DependentTrow>
        <tr v-else>
            <LoadingCell v-if="finalTableData[0].isLoading" :config='config'></LoadingCell>
            <EmptyCell v-if='finalTableData[0].lackOfData' :config='config'></EmptyCell>
        </tr>
    </tbody>
</template>

<script>
import DependentTrow from './DependentTrow.vue';
import LoadingCell from '../LoadingCell.vue';
import EmptyCell from '../EmptyCell.vue';
export default {
    props:[
        'finalTableData',
        'config',
        'originalData',
        'clickedObject',
        'is_granted_menu_edit',
        'is_granted_menu_show',
        'is_granted_menu_pdf',
    ],
    components: {
        DependentTrow,
        LoadingCell, 
        EmptyCell
    }
}
</script>



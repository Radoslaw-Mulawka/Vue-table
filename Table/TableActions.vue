<template>
    <td class='action-border-none'>
        <div class="dropdown">
            <a style="position:relative;" role="button" :id='oneObject.uuid.value' data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
                <div class="content__action-dots">
                    <span class='action-dot'></span>
                    <span class='action-dot'></span>
                    <span class='action-dot'></span>
                </div>
            </a>
            <div class="dropdown-menu dropdown-menu-right dropdown-menu--triangle" :aria-labelledby='oneObject.uuid.value'>
                <template v-for="(action,index) in actions">
                    <a  
                        v-if="!action.fun || (action.fun && (action.fun(objectComparison(oneObject)) == true ? true : checkFunction() )   )"
                        :key='index'
                        class="dropdown-item"
                        :href='changeActionUrl(action.url, oneObject)'
                        @click='isTypeDelete(action.type);'
                        :class="{'nowrap-action': action.type=='show', ['hidden-actions__'+action.type] : true}"
                    >
                        {{action.text}}
                    </a>
                </template>
            </div>
        </div>

        <div>
            <div
                class="modal fade"
                id="confirmDocumentModal"
                tabindex="-1"
                role="dialog"
                aria-labelledby="confirmDocumentModalTitle"
                aria-hidden="true">

                <div class="modal-dialog modal-dialog-centered" role="document">
                    <div class="modal-content text-left" style='color:#11080e'>
                        <div class="modal-header">
                            <h4 class="card-title font-weight-bold pt-1">
                                <i class="fe fe-check-square p-1 mr-3 h5 text-primary bg-light rounded-circle border border-secondary"></i>
                                Usuwanie elementu
                            </h4>
                            <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                                <span aria-hidden="true">&times;</span>
                            </button>
                        </div>
                        <div class="modal-body" style='display: flex; align-items: center;'>
                            <input type="hidden" name="hiddenValue" id="hiddenValue" value="" />
                            <strong id='deleteTextElement'>
                                {{ deleteText }}
                            </strong>
                            <br>
                            <br>
                        </div>
                        <div class="modal-footer">
                            <button type="button" class="btn btn-outline-dark px-3" data-dismiss="modal">
                                <i class="fe fe-x-circle mr-2"></i>Anuluj
                            </button>
                            <button class="btn btn-outline-success px-3" @click="confirmDocument">
                                <i class="fe fe-check-circle mr-2"></i>Usuń
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </td>
</template>



<script>
import {EventBus} from '../../app.js';
export default {
    props:[
        'config',
        'oneObject',
        'type', 
        'is_granted_menu_edit', 
        'is_granted_menu_show', 
        'is_granted_menu_pdf', 
        'originalData'],
    data: function(){
        return {
            isShown: false
        }
    },
    computed: {
        showOrNot: function(){
            return this.isShown==true ? 'block' : 'none'
        },
        actions(){
            let actions = [];
            for(let value of this.config.actions){
                if(this.type == 'dependent-table'){
                    if(value.type == "show" && this.is_granted_menu_show =="1"){
                        actions.push(value);
                    }
                    if(value.type == "file" && this.is_granted_menu_pdf =="1"){
                        actions.push(value);
                    }
                    else if(value.type == "edit" && this.is_granted_menu_edit =="1"){
                        actions.push(value);
                    }
                    else if(value.type == "clone" && this.is_granted_menu_edit =="1"){
                        actions.push(value);
                    }
                    else {
                        continue;
                    }
                }
                else {
                    actions.push(value);
                }
            }
            return actions;
        },
        deleteText(){
            let actionIndex = this.config.actions.findIndex(item=> item.type =='delete');
            if(actionIndex != -1){
                if(this.config.actions[actionIndex].deleteMessage !== undefined){
                  return  this.config.actions[actionIndex].deleteMessage
                 }
                 else {
                     return 'Czy chcesz usunąć ten element?';
                 }
            }
        }
    },
    methods: {
        closeActionsWindow: function(){
            this.isShown = false;
        },
        changeActionUrl: function(url, oneObject) {
            let newUrl = "";

            newUrl = url;
            for (let key in oneObject) {
                newUrl = newUrl.replace("{"+key+"}", oneObject[key].value);
            }

            return newUrl;

        },
        objectComparison(modifiedObject){
            return this.originalData.find(originalObject=>{
                return originalObject.uuid == modifiedObject.uuid.value
            })
        },
        checkFunction(){
            return false;
        },
        isTypeDelete(type, $event){
            event.preventDefault();
            if(type == 'delete'){
                $("#confirmDocumentModal #hiddenValue").val(event.target.href);
                $("#confirmDocumentModal #deleteTextElement").text(this.deleteText);
                $("#confirmDocumentModal").modal("show");               
            }
            else {
                window.location = event.target.href;
            }
        },
        confirmDocument: function() {
             window.location = $("#confirmDocumentModal #hiddenValue").val();
            $("#confirmDocumentModal").modal("hide");
        },
    }
}
</script>

<style lang="scss">
@import '../../../css/variables';
.nowrap-action {
    white-space: nowrap;
}
.action-border-none{
    text-align: center;
}
.content {
    &__table-actions {
        position: relative;
    }
    &__action-dots {
        display: inline-block;
        cursor: pointer;
        position:relative;
        &:hover .action-dot {
            background-color: $action-color;
        }
    }
    &__hidden-actions {
        display: none;
        position: absolute;
        border: 1px solid #ebebeb;
        background-color: white;
        border-radius:5px;
        padding: 10px;
        z-index: 1;
        left: -105px;
        top: 32px;
        width: 150px;
        font-size: 14px;
        box-shadow: 0px 4px 9px -2px rgba(0, 0, 0, 0.59);
        .triangle {
            position: absolute;
            width: 20px;
            height: 20px;
            background-color: #ffffff;
            border-top: 1px solid #ebebeb;
            border-left: 1px solid #ebebeb;
            top: -12px;
            transform: rotate(45deg);
            right: 20px;
        }
    }
}
.hidden-actions {
    &__option {
        display: block;
        border-bottom: 1px solid #e6e6e6;
        // padding-left: 30px;
        margin: 0 5px 10px;
        padding-bottom: 10px;

        &:last-child {
            border: none;
            margin-bottom: 0;
            padding-bottom: 0;
        }
    }

    &__show,
    &__edit,
    &__clone,
    &__change-status,
    &__delete,
    &__client,
    &__assign-group,
    &__assign-kitchen,
    &__assign-diet,
    &__contract,
    &__file,
    &__department {
        &:before {
            font-family: 'feather' !important;
            margin-right: 10px;
        }
    }


    &__show:before { content: "\e97a"; }
    &__edit:before { content: "\e957"; }
    &__clone:before { content: "\e943"; }
    &__change-status:before { content: "\e966"; }
    &__delete:before { content: "\e9e2"; }
    &__client:before { content: "\e9ef"; }
    &__assign-group:before { content: "\e981"; }
    &__assign-kitchen:before { content: "\e9a0"; }
    &__assign-diet:before { content: "\e922"; }
    &__contract:before { content: "\e9ca"; }
    &__file:before { content: "\e963"; }
    &__department:before { content: "\e96a"; }

}
.action-dot {
    display: inline-block;
    width: 6px;
    height: 6px;
    border-radius: 30px;
    background-color: darken($light-txt-col,25%);
    cursor: pointer;
}
</style>

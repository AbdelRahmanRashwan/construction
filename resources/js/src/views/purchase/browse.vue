<template>
    <div>
        <div class="vx-col w-full mb-base" v-if="can('browse-purchase')">
            <vx-card ref="apartment" title="Purchase List" collapse-action refreshContentAction @refresh="getPurchasesData">
                <vs-table :data="purchases">
                    <template slot="header">
                        <vs-button size="small" to="/dashboard/purchase/create" icon-pack="feather" icon="icon-plus" type="filled">Create Purchase</vs-button>
                    </template>
                    <template slot="thead">
                        <vs-th sort-key="id">ID</vs-th>
                        <vs-th sort-key="display_name" v-if="!customer">Customer Name</vs-th>
                        <vs-th sort-key="display_name" v-if="!apartment">Apartment</vs-th>
                        <vs-th sort-key="display_name">End Date</vs-th>
                        <vs-th sort-key="display_name">Finished</vs-th>
                        <vs-th>Action</vs-th>
                    </template>
                    <template slot-scope="{data}">
                        <vs-tr :key="index" v-for="(purchase, index) in data">
                            <vs-td :data="purchase.id">
                                {{ index+1 }}
                            </vs-td>

                            <vs-td :data="purchase.customer" v-if="!customer">
                                <span v-if="purchase.customer">
                                    {{ purchase.customer.name}}
                                </span>
                            </vs-td>

                            <vs-td :data="purchase.apartment" v-if="!apartment">
                                <span v-if="purchase.apartment">
                                    {{ purchase.apartment.project.name}} {{purchase.apartment.floor}} - {{purchase.apartment.area}}
                                </span>
                            </vs-td>

                            <vs-td :data="purchase.end_date">
                                {{ purchase.end_date}}
                            </vs-td>

                            <vs-td :data="purchase.finished">
                                {{ purchase.finished? "yes":"no" }}
                            </vs-td>

                            <vs-td>
                                <vs-row>
                                    <div class="flex mb-4">
                                        <div class="w-1/3">
                                            <vs-button :to="`/dashboard/purchase/${purchase.id}`" radius color="primary"
                                                       type="border" icon-pack="feather" icon="icon-eye"/>
                                        </div>
                                        <div class="w-1/3 ml-5" >
                                            <vs-button :to="`/dashboard/purchase/${purchase.id}/edit`" radius
                                                       color="warning" type="border" icon-pack="feather"
                                                       icon="icon-edit"/>
                                        </div>
                                        <div class="w-1/3 ml-5">
                                            <vs-button :id="`btn-delete-${purchase.id}`"
                                                       class="vs-con-loading__container" radius color="danger"
                                                       type="border" icon-pack="feather" icon="icon-trash"
                                                       @click="is_requesting?$store.dispatch('viewWaitMessage', $vs):confirmDeletePurchase(purchase)"/>
                                        </div>
                                    </div>
                                </vs-row>
                            </vs-td>
                        </vs-tr>
                    </template>
                </vs-table>
            </vx-card>
        </div>
    </div>
</template>

<script>
    import "@fortawesome/fontawesome-free/css/all.css"
    import "@fortawesome/fontawesome-free/js/all.js"
    export default {
        name: "purchase",
        mounted() {
            this.getPurchasesData();
        },
        props: {
            'apartment': {
                required : false
            },
            'customer': {
                required : false
            },
            'filter': {
                required : false
            },
        },
        data: function (){
            return {
                searchText: "",
                resultTime: 0,
                purchases: [],
                is_requesting: false
            }
        },
        methods: {
            getPurchasesData(){
                this.$vs.loading({container: this.$refs.browse, scale: 0.5});

                this.$store.dispatch('purchase/getData', this.filter?this.filter:'')
                    .then(response => {
                        this.$vs.loading.close(this.$refs.browse);
                        this.purchases = response.data.data.data;
                    })
                    .catch(error => {
                        console.log(error);
                        this.$vs.loading.close(this.$refs.browse);
                        this.$vs.notify({
                            title: 'Error',
                            text: error.response.data.error,
                            iconPack: 'feather',
                            icon: 'icon-alert-circle',
                            color: 'danger'
                        });
                    });
            },

            confirmDeletePurchase(type)
            {
                this.$vs.dialog({
                    type: 'confirm',
                    color: 'danger',
                    title: `Are you sure!`,
                    text: 'This data can not be retrieved again.',
                    accept: this.deletePurchase,
                    parameters: [type]
                });
            },

            deletePurchase(params)
            {
                this.is_requesting=true;
                this.$vs.loading({container: `#btn-delete-${params[0].id}`, color: 'danger', scale: 0.45});
                this.$store.dispatch('purchase/delete', params[0].id)
                    .then(response => {
                        this.is_requesting = false;
                        this.$vs.loading.close(`#btn-delete-${params[0].id} > .con-vs-loading`);
                        this.purchases = this.purchases.filter(type => {return type.id !== params[0].id});
                        this.$vs.notify({
                            title: 'Success',
                            text: response.data.message,
                            iconPack: 'feather',
                            icon: 'icon-check',
                            color: 'success'
                        });
                    })
                    .catch(error => {
                        console.log(error);
                        this.is_requesting=false;
                        this.$vs.loading.close(`#btn-delete-${params[0].id} > .con-vs-loading`);
                        this.$vs.notify({
                            title: 'Error',
                            text: error.response.data.error,
                            iconPack: 'feather',
                            icon: 'icon-alert-circle',
                            color: 'danger'
                        });
                    });
            },

            copyToClipboard(text) {
                if (window.clipboardData && window.clipboardData.setData) {
                    // IE specific code path to prevent textarea being shown while dialog is visible.
                    this.onCopy();
                    return clipboardData.setData("Text", text);

                } else if (document.queryCommandSupported && document.queryCommandSupported("copy")) {
                    var textarea = document.createElement("textarea");
                    textarea.textContent = text;
                    textarea.style.position = "fixed";  // Prevent scrolling to bottom of page in MS Edge.
                        document.body.appendChild(textarea);
                    textarea.select();
                    try {
                        this.onCopy();
                        return document.execCommand("copy");  // Security exception may be thrown by some browsers.
                    } catch (ex) {
                        this.onError();
                        return false;
                    } finally {
                        document.body.removeChild(textarea);
                    }
                }
            },
            onCopy() {
                this.$vs.notify({
                    title: 'Success!',
                    text: 'Text copied successfully.',
                    color: 'success',
                    iconPack: 'feather',
                    position: 'bottom-right',
                    icon: 'icon-check-circle'
                });
            },
            onError() {
                this.$vs.notify({
                    title: 'Failed!',
                    text: 'Error in copying text.',
                    color: 'danger',
                    iconPack: 'feather',
                    position: 'bottom-right',
                    icon: 'icon-alert-circle'
                })
            },
        }
    }
</script>

<style>
    .txt-hover:hover{
        cursor: pointer;
        color: black !important;
    }
</style>

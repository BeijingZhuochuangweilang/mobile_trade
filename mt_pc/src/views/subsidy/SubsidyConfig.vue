<template>
<div class="subsidy_config_show">
    <page-content enable ref="subsidy_list" body_key="subsidy_params" req_url="/cash/get_subsidy_params">
        <template v-slot:default="slotProps">
            <div style="height: 80vh">
                <el-table :data="slotProps.content" style="width: 100%" stripe height="100%">
                    <el-table-column prop="stuff.name" label="物料"></el-table-column>
                    <el-table-column prop="gate" label="门槛"></el-table-column>
                    <el-table-column label="优惠类型">
                        <template slot-scope="scope">
                        <el-tag type="success" v-if="scope.row.discount">折扣</el-tag>
                        <el-tag type="warning" v-else-if="scope.row.amount">优惠金额</el-tag>
                        </template>
                    </el-table-column>

                    <!-- 数值列 -->
                    <el-table-column label="优惠值">
                        <template slot-scope="scope">
                        <span v-if="scope.row.discount">{{ scope.row.discount }}折</span>
                        <span v-else-if="scope.row.amount">￥{{ scope.row.amount }}</span>
                        </template>
                    </el-table-column>
                    <el-table-column>
                        <template slot="header">
                            <el-button size="mini" type="success" @click="add_subsidy_diag = true">新增</el-button>
                        </template>
                        <template slot-scope="scope">
                            <el-button size="mini" type="danger" @click="del_subsidy(scope.row)">删除</el-button>
                        </template>
                    </el-table-column>
                </el-table>
            </div>
        </template>
    </page-content>
    <el-dialog title="新增补贴规则" :visible.sync="add_subsidy_diag" width="50%">
        <el-form :model="new_subsidy" ref="new_subsidy" :rules="new_subsidy_rules">
            <el-form-item label="物料" prop="stuff_id">
                <select-search body_key="stuff" get_url="/stuff/get_all" item_label="name" item_value="id" v-model="new_subsidy.stuff_id" :permission_array="['stuff']"></select-search>
            </el-form-item>
            <el-form-item label="门槛" prop="gate">
                <el-input v-model="new_subsidy.gate"></el-input>
            </el-form-item>
            <el-form-item label="优惠类型">
            <el-radio-group v-model="new_subsidy.selectedType">
                <el-radio label="discount">折扣</el-radio>
                <el-radio label="amount">优惠金额</el-radio>
            </el-radio-group>
            </el-form-item>
            <el-form-item label="折扣" prop="discount" v-if="new_subsidy.selectedType === 'discount'">
                <el-tooltip effect="dark" content="输入8即八折，9.5即九五折" placement="right">
                    <el-input v-model="new_subsidy.discount"></el-input>
                </el-tooltip>
            </el-form-item>
            <el-form-item label="优惠金额" prop="amount" v-if="new_subsidy.selectedType === 'amount'">
                <el-input v-model.number="new_subsidy.amount"></el-input>
            </el-form-item>
        </el-form>
        <span slot="footer">
            <el-button @click="add_subsidy_diag = false">取消</el-button>
            <el-button type="primary" @click="add_subsidy">确定</el-button>
        </span>
    </el-dialog>
</div>
</template>

<script>
import PageContent from '../../components/PageContent.vue'
import SelectSearch from '../../components/SelectSearch.vue'
export default {
    name: 'SubsidyConfig',
    components: {
        "page-content": PageContent,
        "select-search": SelectSearch
    },
    data() {
        return {
            add_subsidy_diag: false,
            new_subsidy: {
                stuff_id: 0,
                gate: 0,
                discount: 10,
                amount: null,
                selectedType: 'discount'
            },
            new_subsidy_rules: {
                stuff_id: [{
                    required: true,
                    message: '物料不能为空',
                    trigger: 'blur'
                }],
                gate: [{
                        required: true,
                        message: '门槛不能为空',
                        trigger: 'blur'
                    },
                    {
                        pattern: /^[0-9]+(.[0-9]{1,2})?$/,
                        message: '门槛必须是数字',
                        trigger: 'blur'
                    }
                ],
                discount: [{
                        required: true,
                        message: '折扣不能为空',
                        trigger: 'blur'
                    },
                    {
                        pattern: /^(10|[0-9](\.[0-9]+)?)$/,
                        message: '折扣必须在0-10之间',
                        trigger: 'blur'
                    }
                ],
                amount: [{
                    required: true,
                    message: '请输入优惠金额',
                    trigger: 'blur',
                    validator: (rule, value, callback) => {
                    if (this.new_subsidy.selectedType === 'amount' && (!value || value <= 0)) {
                        callback(new Error('优惠金额必须大于0'));
                    } else {
                        callback();
                    }
                    }
                }]
            }
        }
    },
    methods: {
        refresh_subsidy_list() {
            this.$refs.subsidy_list.refresh();
        },
        add_subsidy: async function () {
            this.$refs.new_subsidy.validate(async (valid) => {
                if (valid) {
                    this.new_subsidy.gate = parseFloat(this.new_subsidy.gate);
                    this.new_subsidy.discount = parseFloat(this.new_subsidy.discount);
                    this.new_subsidy.amount = parseFloat(this.new_subsidy.amount);
                    if (this.new_subsidy.selectedType === 'discount') {
                        this.new_subsidy.amount = null;
                    } else if (this.new_subsidy.selectedType === 'amount') {
                        this.new_subsidy.discount = null;
                    }
                    await this.$send_req('/cash/add_subsidy_params', this.new_subsidy);
                    this.refresh_subsidy_list();
                    this.new_subsidy = {
                        stuff_id: 0,
                        gate: 0,
                        discount: 10,
                        amount: null,
                        selectedType: 'discount'
                    };
                    this.add_subsidy_diag = false;
                }
            });
        },
        del_subsidy: async function (row) {
            try {
                await this.$confirm('是否删除该补贴规则?', '提示', {
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    type: 'warning'
                });
                await this.$send_req('/cash/del_subsidy_params', {
                    id: row.id
                });
                this.refresh_subsidy_list();
            } catch (error) {
                console.log(error);
                return;
            }
        }
    },

}
</script>

<style>

</style>

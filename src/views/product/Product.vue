<template>
    <section>
        <!--工具条-->
        <el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
            <el-form :inline="true" :model="filters">
                <el-form-item>
                    <el-input v-model="filters.keyword" placeholder="关键字"></el-input>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" v-on:click="getProducts">查询</el-button>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" @click="handleAdd">新增</el-button>
                </el-form-item>
            </el-form>
        </el-col>

        <!--列表-->
        <el-table :data="products" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
            <el-table-column type="selection" width="55">
                </el-table-column>
                <el-table-column type="index" width="60">
                </el-table-column>
                <el-table-column prop="name" :show-overflow-tooltip="true" label="标题" width="186" sortable>
                </el-table-column>
                <el-table-column prop="subName" :show-overflow-tooltip="true" label="副标题" width="186" sortable>
                </el-table-column>
                <el-table-column prop="onSaleTime" label="上架时间" width="180" sortable>
                </el-table-column>
                <el-table-column prop="offSaleTimeStr" label="下架时间" width="180" sortable>
                </el-table-column>
                <el-table-column prop="state" label="状态" width="80" :formatter="formatState" sortable>
                    <!--<template slot-scope="scope">
                        <el-popover placement="top">


                            <div v-if="scope.row.state ==1" slot="reference" class="name-wrapper">
                                <el-tag size="medium" class="stateStyle">上架</el-tag>
                            </div>
                            <div v-else-if="scope.row.state ==0" slot="reference" class="name-wrapper">
                                <el-tag size="medium">下架</el-tag>
                            </div>
                        </el-popover>
                    </template>-->
                </el-table-column>
            <el-table-column label="操作" width="150">
                <!--自定义列显示的模板-->
                <template scope="scope">
                    <el-button size="small" @click="handleEdit(scope.$index, scope.row)">编辑</el-button>
                    <!--  scope.$index 行索引      scope.row  该行数据  -->
                    <el-button type="danger" size="small" @click="handleDel(scope.$index, scope.row)">删除</el-button>
                </template>
            </el-table-column>
        </el-table>

        <!--工具条-->
        <el-col :span="24" class="toolbar">
            <el-button type="danger" @click="batchRemove" :disabled="this.sels.length===0">批量删除</el-button>
            <el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="pageSize" :total="total" style="float:right;">
            </el-pagination>
        </el-col>

        <!--新增/编辑界面-->
        <el-dialog title="编辑" v-model="formVisible" :close-on-click-modal="false">
            <el-form :model="form" label-width="80px" :rules="formRules" ref="form">
                <el-form-item label="标题" prop="name">
                    <el-input v-model="form.name" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item label="副标题" prop="subName">
                    <el-input v-model="form.subName" auto-complete="off"></el-input>
                </el-form-item>

                <el-form-item label="商品分类" prop="productTypeId">
                    <select-tree width="225" v-model="form.productTypeId" :options="productTypes"/>
                </el-form-item>

                <el-form-item label="品牌" prop="brandId">
                    <el-select v-model="form.brandId" clearable placeholder="请选择">
                        <el-option
                                v-for="item in brands"
                                :key="item.id"
                                :label="item.name"
                                :value="item.id">
                        </el-option>
                    </el-select>
                </el-form-item>

                <el-form-item label="简介" prop="productExt.description">
                <el-input v-model="form.productExt.description" auto-complete="off"></el-input>
                </el-form-item>
              <el-form-item label="详情" prop="productExt.richContent">
                     <div class="edit_container">
                         <quill-editor v-model="form.productExt.richContent" ref="myQuillEditor" class="editer"
                                       :options="editorOption" @ready="onEditorReady($event)"></quill-editor>
                     </div>
                  <el-input v-model="form.productExt.richContent" auto-complete="off"></el-input>
                 </el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer">
                <el-button @click.native="formVisible = false">取消</el-button>
                <el-button type="primary" @click.native="editSubmit" :loading="editLoading">提交</el-button>
            </div>
        </el-dialog>
    </section>
</template>

<script>
    import util from '../../common/js/util'
    //import NProgress from 'nprogress'
    import { getUserListPage, removeUser, batchRemoveUser, editUser, addUser } from '../../api/api';
    import SelectTree from '@/components/SelectTree.vue';
    import { quillEditor } from "vue-quill-editor"; //调用编辑器
    import 'quill/dist/quill.core.css'
    import 'quill/dist/quill.snow.css'
    import 'quill/dist/quill.bubble.css'
    export default {
        name: 'about',
        components: {
            SelectTree,quillEditor
        },
        data() {
            return {
                editorOption: {},
                // 默认选中值
                selected: 'A',
                // 数据默认字段
                defaultProps: {
                    parent: 'pid',   // 父级唯一标识
                    value: 'id',          // 唯一标识
                    label: 'name',       // 标签显示
                    children: 'children', // 子级
                },
                productTypes:[],

                brands:[],
                logoList:[],

                props:{
                    label:"name",
                    value:"id"
                },
                filters: {
                    keyword: ''
                },
                pageSize:10,
                products: [],
                total: 0,
                page: 1,
                listLoading: false,
                sels: [],//列表选中列

                formVisible: false,//编辑界面是否显示
                editLoading: false,
                formRules: {
                    name: [
                        { required: true, message: '请输入姓名', trigger: 'blur' }
                    ]
                },
                //编辑界面数据:定义模型
                form: {
                    name: '',
                    subName: '',
                    brandId: '',
                    productTypeId: '',
                    productExt: {"description": "", "richContent": ""}
                },

                addFormVisible: false,//新增界面是否显示
                addLoading: false,
                addFormRules: {
                    name: [
                        { required: true, message: '请输入姓名', trigger: 'blur' }
                    ]
                },
                //新增界面数据
                addForm: {
                    name: '',
                    subName: '',
                    brandId: '',
                    productTypeId: '',
                    productExt: {"description": "", "richContent": ""}
                }

            }
        },
        computed: {
            editor() {
                return this.$refs.myQuillEditor.quill
            }
        },
        methods: {

            onEditorReady(editor) {
            },
            //上传之前
            handleBeforeUpload(){
                console.debug(this.logoList.length)
                if(this.logoList.length>0){
                    this.$message({
                        message: '只能上传一个文件',
                        type: 'error'
                    });
                    return false;
                }
            },
            //logo上传成功的钩子函数
            handleSuccess(response, file, fileList){
                this.addForm.logo = response.restObj;
                this.logoList = fileList;
            },
            //删除图片
            handleRemoveLogo(file, fileList){
                console.debug("file",file)
                console.debug("fileList",fileList)
                this.logoList = fileList;
            },
            //加载类型数据
            loadProductTypes(){
                this.$http.get("/product/productType/list")
                    .then(res=>{
                        this.productTypes = res.data;
                    })
            },
            formatState: function (row, column) {
                return row.state === 1 ? '上架' : '下架'
            },
            handleCurrentChange(val) {
                this.page = val;
                this.getProducts();
            },
            //获取品牌列表
            getProducts() {
                this.listLoading = true;
                //NProgress.start();
                this.$http.post("/product/product/json",{
                    pageNum:this.page,
                    pageSize:this.pageSize,
                    keyword:this.filters.keyword
                }).then(res=>{
                    this.listLoading = false;
                    let pageList = res.data;
                    this.products = pageList.rows;
                    this.total = pageList.total;
                })
            },
            //获取品牌列表
            getBrands() {
                this.$http.get("/product/brand/list").then(res=>{
                    this.brands = res.data;
                })
            },
            //删除
            handleDel: function (index, row) {
                this.$confirm('确认删除该记录吗?', '提示', {
                    type: 'warning'
                }).then(() => {
                    //确定删除
                    this.listLoading = true;
                    //NProgress.start();
                    this.$http.delete("/product/product/delete/"+row.id)
                        .then((res)=>{
                            this.listLoading = false;
                            if(res.data.success){
                                this.$message({
                                    message: '删除成功',
                                    type: 'success'
                                });
                                this.getProducts();
                            }else{
                                //失败的提示
                                this.$message({
                                    message: res.data.message,
                                    type: 'error'
                                });
                            }
                        })
                })
            },
            //显示编辑界面
            handleEdit: function (index, row) {
                this.formVisible = true;
                this.form = Object.assign({}, row);
            },

            //显示新增界面
            handleAdd: function () {
                this.formVisible = true;
                this.form = {
                    name: '',
                    subName: '',
                    brandId: '',
                    productTypeId: '',
                    productExt: {"description": "", "richContent": ""}
                };
            },
            //编辑
            editSubmit: function () {
                this.$refs.form.validate((valid) => {
                    if (valid) {
                        this.$confirm('确认提交吗？', '提示', {}).then(() => {
                            this.editLoading = true;
                            //NProgress.start();
                            let para = Object.assign({}, this.form);
                            para.birth = (!para.birth || para.birth == '') ? '' : util.formatDate.format(new Date(para.birth), 'yyyy-MM-dd');
                            editUser(para).then((res) => {
                                this.editLoading = false;
                                //NProgress.done();
                                this.$message({
                                    message: '提交成功',
                                    type: 'success'
                                });
                                this.$refs['form'].resetFields();
                                this.formVisible = false;
                                this.getProducts();
                            });
                        });
                    }
                });
            },
            //新增
            addSubmit: function () {
                this.$refs.addForm.validate((valid) => {
                    if (valid) {
                        this.$confirm('确认提交吗？', '提示', {}).then(() => {
                            this.addLoading = true;
                            //NProgress.start();
                            let para = Object.assign({}, this.addForm);//对象的复制
                            //级联选择器的结果是一个数组
                            para.productTypeId = para.productTypeId[para.productTypeId.length-1];

                            this.$http.post("/product/product/add",para)
                                .then(res=>{
                                    this.addLoading = false;
                                    let data = res.data;
                                    if(data.success){
                                        this.$message({
                                            message: '提交成功',
                                            type: 'success'
                                        });
                                        this.$refs['addForm'].resetFields();//清空表单
                                        this.addFormVisible = false;//关闭模态框
                                        this.getProducts();//重新加载表格数据
                                    }else{
                                        this.$message({
                                            message: data.message,
                                            type: 'error'
                                        });
                                    }
                                })

                            /*addUser(para).then((res) => {
                                this.addLoading = false;
                                //NProgress.done();
                                this.$message({
                                    message: '提交成功',
                                    type: 'success'
                                });
                                this.$refs['addForm'].resetFields();
                                this.addFormVisible = false;
                                this.getProducts();
                            });*/
                        });
                    }
                });
            },
            selsChange: function (sels) {
                this.sels = sels;
            },
            //批量删除
            batchRemove: function () {
                var ids = this.sels.map(item => item.id).toString();
                this.$confirm('确认删除选中记录吗？', '提示', {
                    type: 'warning'
                }).then(() => {
                    this.listLoading = true;
                    //NProgress.start();
                    let para = { ids: ids };
                    batchRemoveUser(para).then((res) => {
                        this.listLoading = false;
                        //NProgress.done();
                        this.$message({
                            message: '删除成功',
                            type: 'success'
                        });
                        this.getProducts();
                    });
                }).catch(() => {

                });
            }
        },
        //相当于jquery的$(function(){})
        mounted() {
            this.getProducts();
            this.getBrands();
            this.loadProductTypes();
        }
    }

</script>

<style scoped>

</style>
<template>
	<section>
		<el-row>
			<el-col class="left-div" :span="8">
				<el-tree :data="productTypes" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
			</el-col>
			<el-col class="reght-div" :span="16"></el-col>
		</el-row>
	</section>
</template>

<script>
    export default {
        data() {
            return {
                productTypes: [],
                defaultProps: {
                    children: 'children',
                    label: 'name'
                }
            };
        },
        methods: {
            handleNodeClick(data) {
                console.log(data);
            },
			loadProductType(){
				this.$http.get("/product/productType/list").then(result=>{
				    let data = result.data;
				    this.prodctTypes =data;
				});
			}

        },
		mounted(){
			this.loadProductType();
		}
    };

</script>

<style scoped>
	.left-div{
		height: 500px;
		border: 2px solid #9999
	}
	.reght-div{
		height: 500px;
		border: 2px solid #9999;
		border-left: none;
	}
</style>
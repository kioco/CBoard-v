<!-- 数据集管理页面 -->
<template>
    <div class="content">
      <div class="row">
        
        <!-- 目录面板 -->
  			<div class="col-md-3">
  			    <div class="box box-solid">
  			        <div class="box-header with-border">
  			            <i class="fa fa-dashboard"></i><h3 class="box-title">Dataset</h3>
  			            <div class="box-tools pull-right">
                        <i class="fa fa-plus toolbar-icon" @click="add"></i>
                    </div>
  			        </div>
  			        <div class="panel-body">
  			            <el-tree 
  			              :data="mainTreeData" 
  			              :props="mainTreeProps"
  			              @node-click="handleNodeClick"></el-tree>
  			        </div>
  			    </div>
  			</div>

        <!-- 配置面板 -->
  			<div class="col-md-9" v-if="datasetConfigVisible">
  				<div class="box">
              <div class="box-header with-border">
                <h3 class="box-title" style="font-weight: bold">{{ currentDataset.name }}</h3>
                <i class="pull-right el-icon-delete" @click="del"></i>
              </div>

              <div class="box-body">
                <!-- Name -->
                <div class="el-form-item">
                  <label class="el-form-item__label">Name:</label>
                  <div class="el-form-item__content">
                    <el-input v-model="currentDataset.name" placeholder="Dataset Name"></el-input>
                  </div>
                </div>

                <!-- Category Name -->
                <div class="el-form-item">
                  <label class="el-form-item__label">Category Name:</label>
                  <div class="el-form-item__content">
                    <el-input v-model="currentDataset.categoryName" placeholder="Category Name"></el-input>
                  </div>
                </div>

                <!-- Datasource -->
                <div class="el-form-item">
                  <label class="el-form-item__label">Datasource:</label>
                  <div class="el-form-item__content">
                    <el-select v-model="currentDataset.data.datasource" placeholder="请选择">
                      <el-option
                        v-for="datasource in datasourceList"
                        :key="datasource.id"
                        :label="datasource.name + ' (' + datasource.type + ')'"
                        :value="datasource.id">
                      </el-option>
                    </el-select>
                  </div>
                </div>

                <!-- SQL Text -->
                <div class="el-form-item">
                  <label class="el-form-item__label">SQL Text:</label>
                  <div class="el-form-item__content">
                    <el-input v-model="currentDataset.data.query.sql" type="textarea" :rows="8" placeholder="input SQL Text"></el-input>
                    <el-button type="primary" size="small" @click="loadData">Load Data</el-button>
                  </div>
                </div>

                <div class="row">
                  <div class="col-md-6">
                    <div class="dashed-box dataset-collection">
                      <span v-for="item in datasetCollection"
                       :key="item.name" 
                       :class="{selected: item.selected}"
                       @click="selectDataset(item)">{{ item.name }}</span>
                    </div>
                  </div>
                  <div class="col-md-6">
                    <div class="dashed-box dataset-tree">
                      <dimension-tree 
                        v-model="currentDimension"
                        :treeData="currentDataset.data.schema.dimension"
                        :options="dimensionOptions"
                        :edit="true"></dimension-tree>
                      <measure-tree 
                        v-model="currentMeasure"
                        :treeData="currentDataset.data.schema.measure"
                        :options="measureOptions"></measure-tree>
                    </div>
                  </div>
                </div>

                <div class="row">
                  <div class="col-md-12" style="margin-top: 20px">
                    <el-button type="primary" size="small" class="pull-right" @click="save">Save</el-button>
                  </div>
                </div>

              </div>
          </div>
  			</div>

      </div>
    </div>
</template>

<script>
import UUID from '@/utils/uuid.js';

export default {
  name: 'DatasetConfig',
  components: {
    DimensionTree: () => import('@/components/widgetConfig/DimensionTree.vue'),
    MeasureTree: () => import('@/components/widgetConfig/MeasureTree.vue')
  },
  created() {
  	this.$store.dispatch('config/getDatasetList');
    this.$store.dispatch('config/getDatasourceList');
  },
  data() {
  	return {
  		mainTreeProps: {
	    	label: 'name'
	    },
      currentDimension: [],
      currentMeasure: [],
      dimensionOptions: {
        animation: 0,
        group: 'valueGroup',
      },
      measureOptions: {
        animation: 0,
        group: 'valueGroup',
      },
      datasetConfigVisible: false, //配置面板的隐藏和显示
      currentDataset: null, //当前的 dataset，点击左侧目录node获得
      allDataset: [], //通过请求返回的所有的 dataset 字段
      flag: 'update' // save 按钮基于 flag 的值判断是更新还是新增操作。update-更新，add-新增
  	}
  },
  computed: {
  	datasetList() {
      return this.$store.state.config.datasetList;
    },
    datasourceList() {
      return this.$store.state.config.datasourceList;
    },
    /*右侧主目录数据*/
    mainTreeData() {
    	let treeData = [];
    	this.datasetList.forEach(dataset => {
    		let item = categoryInArray(dataset.categoryName, treeData);
    		if(item === null) {
    			item = {
	    			name: dataset.categoryName,
	    			children: []
	    		}
	    		item.children.push(dataset)
	    		treeData.push(item);
    		}else {
    			item.children.push(dataset)
    		}

    	})

    	function categoryInArray(categoryName, array) {
    		let result = null;
    		array.forEach(item => {
    			if(item.name === categoryName) result = item;
    		})
    		return result;
    	}

    	return treeData;
    },
    datasetSelected() {
      let schema = this.currentDataset.data.schema;
      let selects = [];
      selects = selects.concat(schema.measure);
      schema.dimension.forEach(e => {
        if (e.type == 'level') {
            e.columns.forEach(c => {
              selects.push(c);
            })
        } else {
            selects.push(e);
        }
      })
      return selects;
    },
    datasetCollection() { //点击 Load Data 加载后，虚线框内获得的字段
      let collection = [];
      let datasetSelected = this.datasetSelected;
      this.allDataset.forEach(item => {
        let obj = { name: item };
        if( existInSelected(item, datasetSelected) ) {
          obj.selected = true;
        }else {
          obj.selected = false;
        }
        collection.push(obj);
      })

      function existInSelected(name, datasetSelected) {
        for(let i=0; i<datasetSelected.length; i++) {
          if(name === datasetSelected[i].column) {
            return true;
          }
        }
        return false;
      }

      return collection;
    },
  },
  methods: {
  	handleNodeClick(node) {
      if(node.children) {
        return;
      }
      this.flag = 'update';
      this.datasetConfigVisible = true;
      this.currentDataset = node;
      this.allDataset = [];
  		console.log(node)
  	},
    add() {
      this.flag = 'add';
      let datasourceId = this.datasetList[0] ? this.datasourceList[0].id : -1;
      this.currentDataset = {
        categoryName: 'Default Category',
        name: '',
        data: {
          expressions: [],
          filters: [],
          schema: {
            dimension: [],
            measure: []
          },
          query: {},
          datasource: datasourceId
        }
      }
      this.datasetConfigVisible = true;
    },
    del() {
      this.$confirm('是否删除该数据集?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
        closeOnClickModal: false,
        customClass: 'preview-config-modal'
      }).then(() => {
         this.$req.post(this.$api.deleteDataset, {id: this.currentDataset.id})
          .then(res => {
            console.log(res)
            this.$message({
                type: 'success',
                message: '删除成功'
            });
            this.datasetConfigVisible = false;
            this.$store.dispatch('config/getDatasetList');
          }).catch(err => {
            this.$message({
                type: 'error',
                message: '删除失败'
            });
          })
      }).catch(err => {})
    },
    loadData() {
     /* const temp_sql = `SELECT    
       b.the_year + 5 AS the_year, b.month_of_year, b.day_of_month,
       date_add(b.the_date, interval 5 year) AS the_date,
       r.SALES_DISTRICT, r.SALES_REGION, r.SALES_COUNTRY,
       d.yearly_income, d.total_children, d.member_card, d.num_cars_owned, d.gender,
       a.store_sales, a.store_cost, a.unit_sales
  FROM foodmart2.sales_fact_sample a
  JOIN foodmart2.time_by_day b ON a.time_id = b.time_id
  JOIN foodmart2.store c ON a.store_id = c.store_id
  JOIN foodmart2.region r ON c.REGION_ID = r.REGION_ID
  JOIN foodmart2.customer d ON a.CUSTOMER_ID = d.CUSTOMER_ID
 WHERE SALES_COUNTRY IS NOT NULL` // 线上预览，写死该语句。*/
      let params = {
        datasourceId: this.currentDataset.data.datasource,
        query: JSON.stringify( {sql: this.currentDataset.data.query.sql} )
        //query: JSON.stringify( {sql: temp_sql} ) // 线上预览，写死该语句。
      };
      this.$store.dispatch('config/getColumns', params)
      .then(data => {
        this.allDataset = data;
      })
    },
    selectDataset(dataset) {
      if(dataset.selected) {
        return;
      }else {
        let id = UUID.generate();
        let item = {
          column: dataset.name,
          type: 'column',
          id: id
        }
        this.currentDataset.data.schema.dimension.push(item);
        dataset.selected = true;
      }
    },
    save() {
      this.currentDataset.data.schema.dimension = this.currentDimension;
      this.currentDataset.data.schema.measure = this.currentMeasure;

      let url = '';
      let message = {};
      if(this.flag === 'update') {
        url = this.$api.updateDataset;
        message = { success: '保存成功', error: '保存失败' };
      }else if(this.flag === 'add') {
        url = this.$api.addDataset;
        message = { success: '新增成功', error: '新增失败' };
      }

      let params = {
        json: JSON.stringify(this.currentDataset)
      };
      this.$req.post(url, params)
        .then(res => {
          if(res.status === 200) {
            this.$message({
                type: 'success',
                message: message.success
            });
            this.$store.dispatch('config/getDatasetList');
          }else {
            this.$message({
                type: 'error',
                message: message.error
            });
          }
        })
        .catch(err => {
          console.log('err', err)
          this.$message({
              type: 'error',
              message: message.success
          });
        })

    }
  }
}
</script>

<style scoped>
.panel-body {
  padding: 10px 20px;
}
.box-tools {
  top: 11px!important;
}
.box-tools i {
  font-size: 16px;
  cursor: pointer;
}
.box-header .el-icon-delete {
  font-size: 20px;
  font-weight: bold;
  cursor: pointer;
}
/*表单样式*/
.el-form-item__label {
  font-weight: bold;
  width: 130px;
}
.el-form-item__content {
  overflow: hidden;
}
.dataset-collection span {
  display: inline-block;
  font-size: 12px;
  vertical-align: middle;
  margin: 3px 3px;
  padding: 5px 10px;
  background-color: #fbfcfd;
  color: #333;
  border: 1px solid #d9e3ec;
  border-radius: 3px;
  cursor: pointer;
  white-space: nowrap;
  user-select: none;
}
.dataset-collection span.selected {
  background-color: #3497db;
  color: #fff;
}
</style>

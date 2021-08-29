<!-- 商品管理---分类管理模块-->
<template>
  <div>
    <el-switch
      v-model="display"
      active-text="开启拖动"
      inactive-text="关闭拖动"
    >
    </el-switch>
    <el-button v-if="display" type="primary" @click="batchSave"
      >拖动数据批量保存</el-button
    >
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <!-- :data表示关联v-bind vue中的对象data  
    show-checkbox 表示是否显示选择框，
    expand-on-click-node表示是否在选中条目时自动展开，设置为false后只有点击箭头才会自动展开 
    node-key唯一标识 
    ref表示vue模板中关联的标签的标识，可以通过this.$ref.tree来进行访问
    -->
    <el-tree
      :data="data"
      :props="defaultProps"
      @node-click="handleNodeClick"
      show-checkbox
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedKeys"
      :draggable="display"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="tree"
    >
      <!--在数据项后展示删除添加按钮 -->
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level < 3"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <!--  -->
          <el-button type="text" size="mini" @click="() => modify(data)">
            Modify
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="tittle"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form ref="form" :model="category" label-width="80px">
        <el-form-item label="分类名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
        <!--提交时候根据类型选择，修改或者新增提交 -->
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      maxLevel: 0,
      data: [],
      tittle: "添加商品分类",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null,
      },
      demoCategory: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null,
      },
      expandedKeys: [],
      dialogVisible: false, //显示对话框，默认false，点击添加后改为true
      display: false,
      defaultProps: {
        children: "categories", // children表示子树的节点的属性名称，在后台返回的数据中list的属性，后台entity中子节点的属性
        label: "name", // label表示要显示的标签的属性名称
      },
    };
  },
  methods: {
    // 是否允许拖动到指定位置，限制最大层级是三层，根据层级判断是否可以拖动
    allowDrop: function (draggingNode, dropNode, type) {
      console.log("draggingNode", draggingNode, dropNode);

      this.maxLevel = 0;
      let currentLevel = this.caculateCurrentNodeLevel(draggingNode);
      if (type == "inner") {
        return dropNode.level + currentLevel <= 3;
      } else {
        return dropNode.parent.level + currentLevel <= 3;
      }
    },
    // 计算需要拖动的节点的层级
    caculateCurrentNodeLevel: function (node) {
      // console.log("currentlevel",node);
      if (this.maxLevel < node.level) {
        this.maxLevel = node.level;
      }
      for (let index = 0; index < node.childNodes.length; index++) {
        let element = node.childNodes[index];
        if (this.maxLevel < element.level) {
          this.maxLevel = element.level;
        }
        // 递归计算层级，递归结束的条件是node没有childNodes了
        this.caculateCurrentNodeLevel(element);
      }
      return this.maxLevel - node.level + 1;
    },
    // 拖动完成后需要更新数据库内容
    handleDrop() {
      // 1. 当前节点最新父节点的id
      // 2. 当前拖动节点的最新顺序
      // 3. 当前拖动节点的最新层级
      console.log("拖动了数据");
    },
    // 批量保存拖动后的数据，防止每次拖动完都保存一次
    batchSave() {
      console.log("批量保存");
    },
    // 批量删除功能，根据checkbox是否被选中来判断是不是需要删除数据
    batchDelete() {
      let keys = this.$refs.tree.getCheckedKeys();
      console.log("被选中的元素:", keys);
       // 添加确认和取消按钮
      this.$confirm(`此操作将永久删除[${keys}]选项, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(keys, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "删除成功",
                type: "success",
              });
              this.queryCatetories();
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    queryCatetories: function () {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
        // 大括号表示解构
      }).then(({ data }) => {
        if (data && data.code == 0) {
          this.data = data.categoryEntities;
        } else {
          this.data = [];
        }
      });
    },
    handleNodeClick(data) {
      console.log(data);
    },
    // 复用了dialog，所以需要根据字段判断是添加还是修改
    submitData() {
      if (this.tittle == "添加商品分类") {
        this.addCategory();
      } else {
        this.modifyCategory();
      }
    },
    append(data) {
      console.log(data);
      console.log("默认值被改变了", this.demoCategory);
      this.tittle = "添加商品分类";
      let {
        catId,
        name,
        showStatus,
        sort,
        productUnit,
        icon,
      } = this.demoCategory; // category重置为初始空状态，防止修改等操作改了category的内容
      this.category.catId = catId;
      this.category.name = name;
      this.category.showStatus = showStatus;
      this.category.sort = sort;
      this.category.productUnit = productUnit;
      this.category.icon = icon;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel + 1;
      this.dialogVisible = true;
    },
    addCategory() {
      console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "添加成功",
            type: "success",
          });
          this.dialogVisible = false;

          this.queryCatetories();
          this.expandedKeys = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
          this.dialogVisible = false;
        }
      });
      this.dialogVisible = false;
    },
    modify(data) {
      console.log(data);
      this.dialogVisible = true;
      this.tittle = "修改商品分类";
      // 再去数据库查一遍，防止长时间未更新别人修改了
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        let tmpCategory = data.category;
        this.category.name = tmpCategory.name;
        this.category.productUnit = tmpCategory.productUnit;
        this.category.icon = tmpCategory.icon;
        this.category.catId = tmpCategory.catId;
        this.category.sort = tmpCategory.sort;
        this.category.parentCid = tmpCategory.parentCid;
        this.category.catLevel = tmpCategory.catLevel;
      });
    },
    modifyCategory() {
      var { catId, name, icon, productUnit } = this.category; // 解构数据，只拿出来需要修改的字段
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "修改成功",
            type: "success",
          });
          this.dialogVisible = false;
          this.queryCatetories();
          this.expandedKeys = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
          this.dialogVisible = false;
        }
      });
    },
    remove(node, data) {
      // 添加确认和取消按钮
      this.$confirm(`此操作将永久删除[${data.name}]选项, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          let ids = [data.catId];
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "删除成功",
                type: "success",
              });
              this.queryCatetories();
              this.expandedKeys = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
  },
  activated() {
    this.queryCatetories();
  },
  components: {},
};
</script>

<style scoped lang="scss">
</style>

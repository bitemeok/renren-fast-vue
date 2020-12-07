<template>
  <div>
    <el-switch
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
      v-model="draggable"
    ></el-switch>
    <el-button v-if="draggable" @click="batchSave"></el-button>
    <el-button @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="categoryTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            @click="() => append(data)"
            v-if="node.level < 3"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => remove(node, data)"
            v-if="node.childNodes.length == 0"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxChildLevel: 1,
      title: "",
      dialogType: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        catId: null,
      },
      menus: [],
      expandedKeys: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      dialogVisible: false,
    };
  },
  activated() {},
  methods: {
    batchDelete() {
      let checkedNodes = this.$refs.categoryTree.getCheckedNodes();
      console.log("被选中的节点：", checkedNodes);
      let catIds = [];
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`确定删除【${catIds}】吗？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              message: "菜单批量删除成功",
              type: "success",
            });
            this.getMenus();
          });
        })
        .catch(() => {});
    },

    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "批量拖拽成功",
          type: "success",
        });
        this.expandedKeys = this.pCid;
        this.getMenus();
      });
    },

    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", draggingNode, dropNode, dropType);

      var pCid = 0;
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
      } else {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
      }
      this.pCid.push(pCid);

      var siblings = null;
      if (dropType == "inner") {
        siblings = dropNode.childNodes;
      } else {
        siblings = dropNode.parent.childNodes;
      }
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          var level = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            level = siblings[i].level;
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: level,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log("updateNodes:", this.updateNodes);
    },

    updateChildNodeLevel(node) {
      var childNodes = node.childNodes;
      if (childNodes.length > 0) {
        for (let i = 0; i < childNodes.length; i++) {
          this.updateNodes.push({
            catId: childNodes[i].data.catId,
            catLevel: childNodes[i].level,
          });
          this.updateChildNodeLevel(childNodes[i]);
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      console.log("allowDrag: ", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode);
      var depth = Math.abs(this.maxChildLevel - draggingNode.level) + 1;
      // console.log("当前节点最大深度：", depth);
      if (type == "inner") {
        return depth + dropNode.level < 4;
      } else {
        return depth + dropNode.parent.level < 4;
      }
    },

    countNodeLevel(draggingNode) {
      if (
        draggingNode.childNodes != null &&
        draggingNode.childNodes.length != 0
      ) {
        for (let index = 0; index < draggingNode.childNodes.length; index++) {
          if (this.maxChildLevel < draggingNode.childNodes[index].level) {
            this.maxChildLevel = draggingNode.childNodes[index].level;
          }
          this.countNodeLevel(draggingNode.childNodes[index]);
        }
      } else {
        this.maxChildLevel = draggingNode.level;
      }
    },

    addCategory() {
      console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: `添加分类【${data.name}】成功`,
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKeys = [this.category.parentCid];
      });
    },
    handleNodeClick(data) {
      console.log(data);
    },
    getMenus() {
      this.$http({
        method: "get",
        url: this.$http.adornUrl("/product/category/list/tree"),
      }).then(({ data }) => {
        console.log("成功获取商品分类数据", data.listWithTree);
        this.menus = data.listWithTree;
        this.maxChildLevel = 0;
        this.updateNodes = [];
        this.pCid = 0;
      });
    },
    append(data) {
      console.log("append", data);
      this.title = "添加分类";
      this.dialogType = "add";
      this.dialogVisible = true;
      this.category.name = "";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },

    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },

    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: `编辑分类【${this.category.name}】成功`,
          type: "success",
        });
        this.dialogVisible = false;
        this.expandedKeys = [this.category.parentCid];
        this.getMenus();
      });
    },

    edit(data) {
      console.log("要修改的数据: ", data);
      this.title = "编辑分类";
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("回显的数据：", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】分类?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: `删除分类【${data.name}】成功`,
              type: "success",
            });
            this.expandedKeys = [node.parent.data.catId];
            this.getMenus();
          });
          console.log("remove", node, data);
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
  },
  created() {
    this.getMenus();
  },
};
</script>

<style>
</style>
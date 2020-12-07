<template>
  <el-tree
    :data="menus"
    :props="defaultProps"
    :expand-on-click-node="false"
    node-key="catId"
    ref="categoryTree"
    @node-click="nodeclick"
  >
  </el-tree>
</template>

<script>
export default {
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    nodeclick(data, node, nodeComponent) {
      console.log("子组件节点被点击", data, node, nodeComponent);
      this.$emit("tree-node-click", data, node, nodeComponent);
    },

    getMenus() {
      this.$http({
        method: "get",
        url: this.$http.adornUrl("/product/category/list/tree"),
      }).then(({ data }) => {
        this.menus = data.listWithTree;
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
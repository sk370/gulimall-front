<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存修改</el-button>
    <el-button type="danger" @click="batchDelete">批量删除修改</el-button>
    <el-tree
      show-checkbox
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedkey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            v-if="node.level <= 2"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length == 0"
            @click="() => remove(data)"
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
      </el-form>
      <el-form :model="category">
        <el-form-item label="图标">
          <el-input v-model="category.item" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <el-form :model="category">
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      draggable: false,
      updateNodes: [],
      pCid:[],
      maxLevel: 0,
      menus: [],
      expandedkey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      dialogType: "",
      title: "",
      dialogVisible: false,
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
    };
  },
  // 监听属性 类似于data概念
  computed: {},
  // 监控data中的数据变化
  watch: {},
  // 方法集合
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        // console.log(data.data);
        this.menus = data.data;
      });
    },
    /**
     * 批量删除选中
     */
    batchDelete(){
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for(let i = 0; i < checkedNodes.length; i++){
        console.log(checkedNodes[i])
        console.log(checkedNodes[i].catId)
        catIds.push(checkedNodes[i].catId)
        console.log(catIds)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success",
            });
            this.getMenus(); //删除后刷新页面
          });
        })
        .catch(() => {
          // 取消删除操作。
        });
    },
    /**
     * 是否可拖动
     */
    allowDrop(draggingNode, dropNode, type) {
      // 规则：分类的最大深度为3
      // 1: type = inner 当前节点的深度 + 被拖动到目标节点的层次 <= 3
      // 2: type !=inner 当前节点的深度 + 被拖动到目标节点的父节点层次 <= 3
      // console.log(draggingNode);
      this.countLNodeLevel(draggingNode); // 被拖动节点的最大深度
      // console.log(this.maxLevel);
      let deep = this.maxLevel - draggingNode.level + 1; //计算拖动节点的层级数
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    /**
     * 找到所有子节点，求出最大层数[递归方法]
     */
    countLNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countLNodeLevel(node.childNodes[i]);
        }
      }
      else {
        this.maxLevel = node.level; //应该要写，当只有一层时的深度，但老师为什么没写
      }
    },
    /**
     * 节点拖拽成功后触发的事件
     */
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // 1. 最新父节点id
      let pCid = 0
      let siblings = null
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      this.pCid.push(pCid)
      // console.log(pCid);
      // console.log(siblings);
      // 2. 拖拽节点的最新顺序(统计)
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != catLevel) {
            // 拖拽节点的层级发生了变化
            // 2.1 拖拽节点的新层级
            catLevel = siblings[i].level;
            // 2.2 拖拽节点子节点的新层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log(this.updateNodes);
    },
    /**
     * 保存拖拽后的改动
     */
    batchSave() {
      // 3. 更新数据库
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序等修改成功",
          type: "success",
        });
        this.getMenus() // 修改后刷新页面
        this.expandedkey = this.pCid // 展开原先的分类层级
        this.updateNodes = []
        this.maxLevel = 0
      });
    },
    /**
     * 遍历节点，修改层级
     */
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    /**
     * 添加分类事件
     */
    append(data) {
      this.dialogVisible = true; //设置显示对话框
      this.dialogType = "add";
      this.title = "添加分类";
      // 获取要添加的分类数据
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //*1的作用是转换为数字
      //解决先点击修改按钮，category的其他数据发生变化的问题
      this.category.name = "";
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.catId = null;
      console.log("要添加的分类节点信息", data);
    },
    /**
     * 修改分类事件
     */
    edit(data) {
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.title = "修改分类";
      // 发送请求获取节点的最新数据
      // console.log("要修改的节点信息", data);
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        // console.log("要回显的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    /**
     * 点击确定按钮判断请求类型
     */
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    /**
     * 添加分类请求
     */
    addCategory() {
      console.log("要添加的分类信息" + this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "添加成功",
          type: "success",
        });
        this.dialogVisible = false; // 关闭对话框
        this.getMenus(); // 添加后刷新页面
        this.expandedkey = [this.category.parentCid]; // 展开原先的分类层级
      });
    },
    /**
     * 修改分类请求
     */
    editCategory() {
      //封装要修改的数据
      var { catId, name, icon, productUnit } = this.category; // 解构赋值
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功",
          type: "success",
        });
        this.dialogVisible = false; // 关闭对话框
        this.getMenus(); //修改后刷新页面
        this.expandedkey = [this.category.parentCid]; //展开原先的分类层级
      });
    },
    /**
     * 删除分类请求
     */
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
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
              message: "删除成功",
              type: "success",
            });
            this.getMenus(); //删除后刷新页面
            this.expandedkey = [node.parent.data.catId]; //展开原先的分类层级
          });
        })
        .catch(() => {
          // 取消删除操作。
        });
      console.log("remove", node, data);
    },
  },
  // 生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  // 生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, // 生命周期 - 创建之前
  beforeMount() {}, // 生命周期 - 挂载之前
  beforeUpdate() {}, // 生命周期 - 更新之前
  updated() {}, // 生命周期 - 更新之后
  beforeDestroy() {}, // 生命周期 - 销毁之前
  destroyed() {}, // 生命周期 - 销毁完成
  activated() {}, // 如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>
<template>
  <div class="v-cascader-wrapper">
    <el-popover
      v-model="showPopover"
      placement="bottom-start"
      trigger="manual"
      popper-class="v-cascader-popper">
      <div slot="reference">
        <el-select
          v-model="selectedLabels"
          multiple
          :placeholder="placeholder"
          style="width: 100%;"
          popper-class="v-cascader-popper-hide"
          :disabled="disabled"
          :size="size"
          @change="selectChange"
          @focus="handleFocus" />
      </div>
      <div
        v-click-outside="hidePopover"
        class="v-cascader-menu-wrapper">
        <ul
          v-for="(cas, index) in casTree"
          :key="index"
          class="el-cascader-menu v-cascader-menu">
          <li
            v-for="(item, itemIdx) in cas"
            :key="itemIdx"
            :class="{
              'el-cascader-menu__item': true,
              'el-cascader-menu__item--extensible': item.children && item.children.length > 0,
              'has-checked-child': item.hasCheckedChild,
              'is-active': item.hasCheckedChild,
            }"
            @click="spreadNext(item.children, index)">
            <el-checkbox
              v-if="item.checkbox !== false"
              v-model="item.checked"
              :disabled="item.disabled"
              @change="checked => { checkedChange(item, index, checked) }">
              {{ item.label }}
            </el-checkbox>
            <span v-else>
              {{ item.label }}
            </span>
          </li>
        </ul>
      </div>
    </el-popover>
  </div>
</template>

<script>
import ClickOutside from 'vue-click-outside'

function deepClone(source) {
  if (!source && typeof source !== "object") {
    throw new Error("error arguments", "shallowClone");
  }
  const targetObj = source.constructor === Array ? [] : {};
  Object.keys(source).forEach(keys => {
    if (source[keys] && typeof source[keys] === "object") {
      targetObj[keys] = source[keys].constructor === Array ? [] : {};
      targetObj[keys] = deepClone(source[keys]);
    } else {
      targetObj[keys] = source[keys];
    }
  });
  return targetObj;
}

export default {
  name: 'VMultiCascader',
  directives: { ClickOutside },
  props: {
    value: {
      type: Array,
      default() {
        return [];
      }
    },
    placeholder: {
      type: String,
      default: "请选择"
    },
    disabled: {
      type: Boolean,
      default: false
    },
    options: {
      type: Array,
      default() {
        return [];
      }
    },
    size: {
      type: String,
      default: ""
    },
    multipleLimit: {
      type: Number,
      default: 0
    }
  },
  data() {
    return {
      elWidth: "",
      popperWidth: "",
      showPopover: false,
      clonedOpts: [],
      casTree: [],
      selectedItems: [],
      selectedLabels: [],
      selectedValues: [],
      hasInitModel: false
    };
  },
  watch: {
    options: {
      deep: true,
      handler() {
        this.initOpts();
      }
    },
    value: {
      deep: true,
      handler() {
        if (!this.hasInitModel) {
          this.initDatas();
          this.hasInitModel = true;
        }
      }
    }
  },
  created() {
    this.initOpts();
    this.initDatas();
  },
  mounted() {
    // 设置弹出层宽度
    this.elWidth = this.$el.offsetWidth;
    this.setPopperWidth();
  },
  destroyed() {
    this.clonedOpts = null;
    this.casTree = null;
    this.selectedItems = null;
    this.selectedLabels = null;
    this.selectedvalues = null;
  },
  methods: {
    initOpts() {
      this.clonedOpts = deepClone(this.options);
      this.casTree = [this.clonedOpts];
    },
    // 初始化
    initDatas() {
      this.selectedValues = this.value;
      this.selectedItems = [];
      this.selectedLabels = [];
      this.recursiveOpt(this.clonedOpts);
    },
    // 递归option数据
    recursiveOpt(children) {
      const vm = this;
      children.forEach(child => {
        child.checked = false;
        if (child.checkbox !== false) {
          if (this.selectedValues.some(val => val + "" == child.value + "")) {
            this.selectedItems.push(child);
            this.selectedLabels.push(child.label);
            child.checked = true;
            this.describeCheckedMap(child);
          } else {
            child.checked = false;
          }
        }

        if (child.children && child.children.length > 0) {
          vm.recursiveOpt(child.children);
        }
      });
    },
    // 描述已选中路径
    describeCheckedMap(targetChild) {
      function findParents(parents = [], children) {
        children.forEach(child => {
          if (targetChild.value === child.value) {
            if (child.checked) {
              parents.forEach(per => {
                per.hasCheckedChild = true;
              });
            } else {
              // 倒序遍历
              for (let i = parents.length - 1; i >= 0; i--) {
                let per = parents[i];
                let hasCheckedChild = false;
                if (per.children && per.children.length > 0) {
                  hasCheckedChild = per.children.some(
                    v => v.checked || v.hasCheckedChild
                  );
                }
                per.hasCheckedChild = hasCheckedChild;
              }
            }
          } else {
            if (Array.isArray(child.children)) {
              findParents([...parents, child], child.children);
            }
          }
        });
      }
      findParents([], this.clonedOpts);
    },
    // 展开下一级
    spreadNext(children, index) {
      if (index || index === 0) {
        if (children && children.length > 0) {
          this.casTree.splice(index + 1, this.casTree.length - 1, children);
        } else {
          this.casTree.splice(index + 1, this.casTree.length - 1);
        }
        this.setPopperWidth();
      }
    },
    syncData() {
      // 同步数据到上层
      this.$emit("input", this.selectedValues);
      this.$emit("change", this.selectedValues, this.selectedItems);
    },
    // 选择框点击删除
    selectChange(val) {
      this.selectedLabels = val;
      let deletedItem = this.selectedItems.filter(
        item => !val.includes(item.label)
      )[0];
      if (deletedItem) {
        deletedItem.checked = false;
        this.describeCheckedMap(deletedItem);
      }
      this.selectedItems = this.selectedItems.filter(v =>
        this.selectedLabels.includes(v.label)
      );
      this.selectedValues = this.selectedItems.map(v => v.value);
      this.syncData();
      this.refresPopover();
    },
    // 菜单选中变化
    checkedChange(item, index, checked) {
      if (checked) {
        if (this.multipleLimit <= 0 || this.selectedItems.length < this.multipleLimit) {
          this.selectedItems.push(item);
        } else {
          item.checked = false
        }
      } else {
        this.selectedItems = this.selectedItems.filter(
          selectedItem => selectedItem.label != item.label
        );
      }
      this.selectedLabels = [];
      this.selectedValues = [];
      this.selectedItems.forEach(item => {
        this.selectedLabels.push(item.label);
        this.selectedValues.push(item.value);
      });
      this.syncData();
      this.describeCheckedMap(item);
      this.refresPopover();
    },
    // 改变菜单宽度
    setPopperWidth() {
      let width = (160 + 1) * this.casTree.length;
      document.getElementsByClassName("v-cascader-popper")[0].style.width =
        width + "px";
    },
    handleFocus(evt) {
      if (this.disabled) return;
      this.showPopover = true;
      this.$emit("focus", evt);
    },
    hidePopover(evt) {
      this.showPopover = false;
      this.$emit("blur", evt);
    },
    refresPopover() {
      // 触发resize，让poppover跟随选框，不兼容IE
      let resize = new Event("resize");
      setTimeout(() => {
        window.dispatchEvent(resize);
      }, 66);
    }
  }
};
</script>

<style>
.v-cascader-popper-hide {
  display: none;
}
.v-cascader-popper {
  width: 100%;
  padding: 0px !important;
}
.v-cascader-menu {
  width: 160px !important;
}
.el-cascader-menu__item.has-checked-child {
  background-color: #f5f7fa !important;
}
</style>

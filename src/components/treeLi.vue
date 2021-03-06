<template>
  <li
    v-if="itemVisible"
    :class='liClass'
  >
    <div class="tree-node-el" :draggable="draggable" @dragstart="drag(item, $event)">
      <span @click="expandNode(item)" v-if="showExpand" class='tree-expand' 
        :class="item.expanded ? 'tree-open' : 'tree-close'"
      >
      </span>
      <span v-if='multiple && !item.nocheck' :class="[item.checked ? (item.halfcheck ? 'box-halfchecked' : 'box-checked') : 'box-unchecked', 'inputCheck']">
        <input :disabled="item.chkDisabled"
          :class="['check', item.chkDisabled ? 'chkDisabled' : '']"
          v-if='multiple' type="checkbox" @change="changeNodeCheckStatus(item, $event)" :checked="item.checked"/>
      </span>
      <loading v-if="item.loading && item.expanded"/>
      <Render :node="item" :parent='parent' :index='index' :tpl ='tpl' :nodeMouseOver='nodeMouseOver'/>
    </div>
    <template v-if="showNextUl">
      <collapse-transition>
        <TreeUl
          v-show="item.expanded"
          :dragAfterExpanded="dragAfterExpanded"
          :draggable="draggable"
          :tpl ="tpl"
          :data="item.children"
          :halfcheck='halfcheck'
          :scoped ='scoped'
          :parent ='item'
          :canDeleteRoot ='canDeleteRoot'
          :multiple="multiple"
          :level='level'
          :maxLevel='maxLevel'
        />
      </collapse-transition>
    </template>
  </li>
</template>
<script>
import mixins from './mixins'
import Render from './render'
import Loading from './loading'
import CollapseTransition from './collapse-transition'
export default {
  name: 'TreeLi',
  mixins: [mixins],
  components: {Render, Loading, CollapseTransition, 
    // TreeUl: () => import('./treeUl.vue') // 解决循环引用的问题
  },
  beforeCreate() {
    this.$options.components.TreeUl = require('./treeUl.vue').default
  },
  props: {
    item: {
      type: Object,
      default: () => {}
    },
    index: Number,
    dataLength: {
      type: Number,
      default: 0
    },
    parent: {
      type: Object,
      default: () => null
    },
    multiple: {
      type: Boolean,
      default: false
    },
    draggable: {
      type: Boolean,
      default: false
    },
    dragAfterExpanded: {
      type: Boolean,
      default: true
    },
    halfcheck: {
      type: Boolean,
      default: false
    },
    scoped: {
      type: Boolean,
      default: false
    },
    canDeleteRoot: {
      type: Boolean,
      default: false
    },
    tpl: Function,
    maxLevel: Number,
    level: Number
  },
  inject:['isLeaf', 'childChecked', 'parentChecked', 'nodeSelected', 'emitEventToTree', 'setAttr'],
  computed: {
    itemVisible() {
      let {visible = true} = this.item
      // visible = visible === false ? false : true
      return visible
    },
    hasExpended() { // 已经展开过
      let {hasExpended = false, expanded = false} = this.item
      return this.itemVisible && (expanded || hasExpended)
    },
    liClass (){
      const index = this.index
      let res
      if (this.parent) {
        res = {
          leaf: this.isLeaf(this.item),
        }
      } else { // top node
        res = {
          'first-node': index === 0,
          'only-node': this.dataLength === 1,
          'second-node': index === 1
        }
      }
      return res
    },
    hasChildren () {
      const item = this.item
      return item.children && item.children.length > 0
    },
    showExpand () {
      const item = this.item
      return !this.parent || this.hasChildren || item.async
    },
    showNextUl () {
      return !this.isLeaf(this.item) && this.maxLevel > this.level && this.hasExpended
    }
  },
  watch: {
    'item.checked': {
      handler () {
        this.checkedChange()
      },
      immediate: true
    },
    'item.halfcheck': {
      handler () {
        this.checkedChange()
      },
      immediate: true
    }
  },
  methods: {
    /* @method drag node
     * @param node draged node
     * @param ev  $event
    */
    drag (node, ev) {
      let guid = this.guid()
      this.setDragNode(guid, node, this.parent)
      ev.dataTransfer.setData('guid', guid)
    },
    /* @method expand or close node
     * @param node current node
    */
    expandNode (node) {
      const expended = !node.expanded
      this.setAttr(node, 'expanded', expended)
      this.setAttr(node, 'hasExpended', true)
      if(node.children || node.async) {
        if (node.async && !node.children) {
          this.emitEventToTree('async-load-nodes', node)
        }
      }
    },
    /* @event passing the node-check event to the parent component
     * @param node clicked node
     */
    nodeCheck (node, checked) {
      this.$set(node, 'checked', checked)
      const halfcheck = this.halfcheck
      if (halfcheck) {
        this.$set(node, 'halfcheck', false)
      }
      if (!this.scoped) {
        this.childChecked(node, checked, halfcheck)
        // this.theParentChecked(checked, halfcheck)
      }
    },
    /* @event passing the node-mouse-over event to the parent component
     * @param node overed node
     */
    nodeMouseOver (node, index, parent) {
      this.emitEventToTree('node-mouse-over', node, index, parent)
      // this.$emit('node-mouse-over', node)
    },
    /*
     *@method change the check box status method
     *@param node current node
     *@param $event event object
     */
    changeNodeCheckStatus (node, $event) {
      const checked = $event.target.checked
      this.nodeCheck(node, checked)
      this.emitEventToTree('node-check', node, checked)
    },
    theParentChecked(checked, halfcheck){
      const parentNode = this.parent
      this.parentChecked(parentNode, checked, halfcheck)
    },
    checkedChange () {
      const {checked = false} = this.item
      this.theParentChecked(checked, this.halfcheck)
      // if (!checked) {
      //   this.$set(this.item, 'selected', checked)
      // }
    },
  }
}
</script>
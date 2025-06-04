<template>
  <div
    ref="tableWrapper"
    :class="[
      {
        [ns.m('fit')]: fit,
        [ns.m('striped')]: stripe,
        [ns.m('border')]: border || isGroup,
        [ns.m('hidden')]: isHidden,
        [ns.m('group')]: isGroup,
        [ns.m('fluid-height')]: maxHeight,
        [ns.m('scrollable-x')]: layout.scrollX.value,
        [ns.m('scrollable-y')]: layout.scrollY.value,
        [ns.m('enable-row-hover')]: !store.states.isComplex.value,
        [ns.m('enable-row-transition')]:
          (store.states.data.value || []).length !== 0 &&
          (store.states.data.value || []).length < 100,
        'has-footer': showSummary,
      },
      ns.m(tableSize),
      className,
      ns.b(),
      ns.m(`layout-${tableLayout}`),
    ]"
    :style="style"
    :data-prefix="ns.namespace.value"
    @mouseleave="handleMouseLeave"
  >
    <div :class="ns.e('inner-wrapper')" :style="tableInnerStyle">
      <div ref="hiddenColumns" class="hidden-columns">
        <slot />
      </div>
      <div
        v-if="showHeader && tableLayout === 'fixed'"
        ref="headerWrapper"
        v-mousewheel="handleHeaderFooterMousewheel"
        :class="ns.e('header-wrapper')"
      >
        <table
          ref="tableHeader"
          :class="ns.e('header')"
          :style="tableBodyStyles"
          border="0"
          cellpadding="0"
          cellspacing="0"
        >
          <hColgroup
            :columns="store.states.columns.value"
            :table-layout="tableLayout"
          />
          <table-header
            ref="tableHeaderRef"
            :border="border"
            :default-sort="defaultSort"
            :store="store"
            @set-drag-visible="setDragVisible"
          />
        </table>
      </div>
      <div ref="bodyWrapper" :class="ns.e('body-wrapper')">
        <el-scrollbar
          ref="scrollBarRef"
          :view-style="scrollbarViewStyle"
          :wrap-style="scrollbarStyle"
          :always="scrollbarAlwaysOn"
        >
          <table
            ref="tableBody"
            :class="ns.e('body')"
            cellspacing="0"
            cellpadding="0"
            border="0"
            :style="{
              width: bodyWidth,
              tableLayout,
            }"
          >
            <hColgroup
              :columns="store.states.columns.value"
              :table-layout="tableLayout"
            />
            <table-header
              v-if="showHeader && tableLayout === 'auto'"
              ref="tableHeaderRef"
              :class="ns.e('body-header')"
              :border="border"
              :default-sort="defaultSort"
              :store="store"
              @set-drag-visible="setDragVisible"
            />
            <table-body
              :context="context"
              :highlight="highlightCurrentRow"
              :row-class-name="rowClassName"
              :tooltip-effect="tooltipEffect"
              :tooltip-options="tooltipOptions"
              :row-style="rowStyle"
              :store="store"
              :stripe="stripe"
            />
            <table-footer
              v-if="showSummary && tableLayout === 'auto'"
              :class="ns.e('body-footer')"
              :border="border"
              :default-sort="defaultSort"
              :store="store"
              :sum-text="computedSumText"
              :summary-method="summaryMethod"
            />
          </table>
          <div
            v-if="isEmpty"
            ref="emptyBlock"
            :style="emptyBlockStyle"
            :class="ns.e('empty-block')"
          >
            <span :class="ns.e('empty-text')">
              <slot name="empty">{{ computedEmptyText }}</slot>
            </span>
          </div>
          <div
            v-if="$slots.append"
            ref="appendWrapper"
            :class="ns.e('append-wrapper')"
          >
            <slot name="append" />
          </div>
        </el-scrollbar>
      </div>
      <div
        v-if="showSummary && tableLayout === 'fixed'"
        v-show="!isEmpty"
        ref="footerWrapper"
        v-mousewheel="handleHeaderFooterMousewheel"
        :class="ns.e('footer-wrapper')"
      >
        <table
          :class="ns.e('footer')"
          cellspacing="0"
          cellpadding="0"
          border="0"
          :style="tableBodyStyles"
        >
          <hColgroup
            :columns="store.states.columns.value"
            :table-layout="tableLayout"
          />
          <table-footer
            :border="border"
            :default-sort="defaultSort"
            :store="store"
            :sum-text="computedSumText"
            :summary-method="summaryMethod"
          />
        </table>
      </div>
      <div v-if="border || isGroup" :class="ns.e('border-left-patch')" />
    </div>
    <div
      v-show="resizeProxyVisible"
      ref="resizeProxy"
      :class="ns.e('column-resize-proxy')"
    />
  </div>
</template>

<script lang="ts">
// @ts-nocheck
import {
  computed,
  defineComponent,
  getCurrentInstance,
  onMounted,
  provide,
  ref,
} from 'vue'
import { debounce } from 'lodash-unified'
import { Mousewheel } from '@element-plus/directives'
import { useLocale, useNamespace } from '@element-plus/hooks'
import ElScrollbar from '@element-plus/components/scrollbar'
import { createStore } from './store/helper'
import TableLayout from './table-layout'
import TableHeader from './table-header'
import TableBody from './table-body'
import TableFooter from './table-footer'
import useUtils from './table/utils-helper'
import useStyle from './table/style-helper'
import useKeyRender from './table/key-render-helper'
import defaultProps from './table/defaults'
import { TABLE_INJECTION_KEY } from './tokens'
import { hColgroup } from './h-helper'
import { useScrollbar } from './composables/use-scrollbar'

import type { Table } from './table/defaults'

let tableIdSeed = 1
export default defineComponent({
  name: 'ElTable',
  directives: {
    Mousewheel,
  },
  components: {
    TableHeader,
    TableBody,
    TableFooter,
    ElScrollbar,
    hColgroup,
  },
  props: defaultProps,
  emits: [
    'select',
    'select-all',
    'selection-change',
    'cell-mouse-enter',
    'cell-mouse-leave',
    'cell-contextmenu',
    'cell-click',
    'cell-dblclick',
    'row-click',
    'row-contextmenu',
    'row-dblclick',
    'header-click',
    'header-contextmenu',
    'sort-change',
    'filter-change',
    'current-change',
    'header-dragend',
    'expand-change',
  ],
  setup(props) {
    type Row = typeof props.data[number]
    const { t } = useLocale()
    const ns = useNamespace('table')
    const table = getCurrentInstance() as Table<Row>
    provide(TABLE_INJECTION_KEY, table)
    const store = createStore<Row>(table, props)
    table.store = store
    const layout = new TableLayout<Row>({
      store: table.store,
      table,
      fit: props.fit,
      showHeader: props.showHeader,
    })
    table.layout = layout

    const isEmpty = computed(() => (store.states.data.value || []).length === 0)

    /**
     * open functions
     */
    const {
      setCurrentRow,
      getSelectionRows,
      toggleRowSelection,
      clearSelection,
      clearFilter,
      toggleAllSelection,
      toggleRowExpansion,
      clearSort,
      sort,
    } = useUtils<Row>(store)
    const {
      isHidden,
      renderExpanded,
      setDragVisible,
      isGroup,
      handleMouseLeave,
      handleHeaderFooterMousewheel,
      tableSize,
      emptyBlockStyle,
      handleFixedMousewheel,
      resizeProxyVisible,
      bodyWidth,
      resizeState,
      doLayout,
      tableBodyStyles,
      tableLayout,
      scrollbarViewStyle,
      tableInnerStyle,
      scrollbarStyle,
    } = useStyle<Row>(props, layout, store, table)

    const { scrollBarRef, scrollTo, setScrollLeft, setScrollTop } =
      useScrollbar()

    const debouncedUpdateLayout = debounce(doLayout, 50)

    const tableId = `${ns.namespace.value}-table_${tableIdSeed++}`
    table.tableId = tableId
    table.state = {
      isGroup,
      resizeState,
      doLayout,
      debouncedUpdateLayout,
    }
    const computedSumText = computed(
      () => props.sumText || t('el.table.sumText')
    )

    const computedEmptyText = computed(() => {
      return props.emptyText || t('el.table.emptyText')
    })

    const extractDisplayText = (vnode: any, depth = 0): string => {
      if (!vnode) return ''

      // 处理字符串
      if (typeof vnode === 'string') {
        return vnode.trim()
      }

      // 处理文本节点
      if (vnode.type === Symbol.for('v-txt')) {
        const text = vnode.children || ''
        return typeof text === 'string' ? text.trim() : ''
      }

      // 处理数组
      if (Array.isArray(vnode)) {
        return vnode
          .map((item) => extractDisplayText(item, depth + 1))
          .filter(Boolean)
          .join('')
      }

      // 处理函数
      if (typeof vnode === 'function') {
        try {
          return extractDisplayText(vnode(), depth + 1)
        } catch {
          return ''
        }
      }

      // 处理对象（包括 VNode）
      if (typeof vnode === 'object' && vnode !== null) {
        // 处理 children 为插槽对象且有 default 函数的情况
        if (
          vnode.children &&
          typeof vnode.children === 'object' &&
          typeof vnode.children.default === 'function'
        ) {
          try {
            return extractDisplayText(vnode.children.default(), depth + 1)
          } catch {
            return ''
          }
        }
        // 处理子节点
        if (vnode.children) {
          if (typeof vnode.children === 'function') {
            try {
              return extractDisplayText(vnode.children(), depth + 1)
            } catch {
              return ''
            }
          }
          return extractDisplayText(vnode.children, depth + 1)
        }
        // 处理默认插槽
        if (vnode.props?.default) {
          return extractDisplayText(vnode.props.default, depth + 1)
        }
        // 处理其他属性
        if ('value' in vnode) return String(vnode.value)
        if ('label' in vnode) return String(vnode.label)
        if ('text' in vnode) return String(vnode.text)
        if ('content' in vnode) return String(vnode.content)
      }
      return ''
    }

    const extractValue = (vnode: any) => {
      if (!vnode) return ''

      console.log('提取值 - 详细结构:', {
        type: vnode.type,
        props: vnode.props,
        children: vnode.children,
        isVNode: vnode.__v_isVNode,
        shapeFlag: vnode.shapeFlag,
        patchFlag: vnode.patchFlag,
        component: vnode.component,
        el: vnode.el,
      })

      // 处理字符串
      if (typeof vnode === 'string') {
        return vnode.trim()
      }

      // 处理函数
      if (typeof vnode === 'function') {
        try {
          return extractDisplayText(vnode())
        } catch (error) {
          console.error('执行函数失败:', error)
          return ''
        }
      }

      // 处理数组
      if (Array.isArray(vnode)) {
        return vnode
          .map((item) => extractDisplayText(item))
          .filter(Boolean)
          .join('')
      }

      // 处理对象（包括 VNode）
      if (typeof vnode === 'object' && vnode !== null) {
        // 处理子节点
        if (vnode.children) {
          console.log('处理子节点:', vnode.children)
          if (typeof vnode.children === 'function') {
            try {
              return extractDisplayText(vnode.children())
            } catch (error) {
              console.error('执行子节点函数失败:', error)
              return ''
            }
          }
          return extractDisplayText(vnode.children)
        }

        // 处理默认插槽
        if (vnode.props?.default) {
          console.log('处理默认插槽:', vnode.props.default)
          return extractDisplayText(vnode.props.default)
        }

        // 处理其他属性
        if ('value' in vnode) return String(vnode.value)
        if ('label' in vnode) return String(vnode.label)
        if ('text' in vnode) return String(vnode.text)
        if ('content' in vnode) return String(vnode.content)
      }

      return ''
    }

    /**
     * 虚拟渲染数据提取方法
     * @param data 需要转换的原始数据数组
     * @returns { data: 渲染后纯文本数据数组, restore: 恢复函数（此实现已无副作用，可为空） }
     * 说明：本方法不会修改表格的真实数据，也不会触发真实DOM渲染，仅做数据转换，适合大数据量导出。
     */
    const getVirtualRenderData = (data: any[]) => {
      const instance = table
      const results = new Map()
      const rowKey = props.rowKey || 'id'

      // 获取行标识
      const getRowKey = (row: any) => {
        if (typeof rowKey === 'function') {
          return rowKey(row)
        }
        return row[rowKey]
      }

      // 保存原始的 renderCell 方法（本实现不再修改 store.states.data.value，不会影响真实渲染）
      const originalRenderCells = new Map()
      store.states.columns.value.forEach((column) => {
        originalRenderCells.set(column.id, column.renderCell)
        // 虚拟渲染：仅用于提取文本，不做真实渲染
        column.renderCell = (scope) => {
          // 获取原始值
          const rawValue = scope.row[column.property]

          // 获取格式化后的值
          let formattedValue = rawValue
          if (column.formatter) {
            formattedValue = column.formatter(
              scope.row,
              column,
              rawValue,
              scope.$index
            )
          }

          // 获取渲染后的值
          let renderedValue

          // 获取当前列的插槽
          const slots = instance.slots
          if (slots.default) {
            const slotContent = slots.default()
            if (Array.isArray(slotContent)) {
              // 查找与当前列匹配的插槽
              const columnSlot = slotContent.find((slot: any) => {
                const slotColumn = slot.props?.column
                return (
                  slotColumn?.id === column.id ||
                  slotColumn?.property === column.property ||
                  (slotColumn?.type === 'default' &&
                    slotColumn?.prop === column.property) ||
                  slot.props?.prop === column.property ||
                  slot.props?.column?.property === column.property
                )
              })

              if (columnSlot?.props?.default) {
                // 使用当前列的插槽渲染，传递作用域参数
                const slotScope = {
                  row: scope.row,
                  column: scope.column,
                  $index: scope.$index,
                  store: instance.store,
                  _self: instance,
                }
                try {
                  const slotResult = columnSlot.props.default(slotScope)
                  renderedValue = Array.isArray(slotResult)
                    ? slotResult[0]
                    : slotResult
                } catch {
                  // 插槽渲染异常，忽略
                }
              }
            }
          }

          // 如果没有插槽渲染结果，使用原始的 renderCell
          if (!renderedValue) {
            const originalRender = originalRenderCells.get(column.id)
            if (originalRender) {
              try {
                renderedValue = originalRender(scope)
              } catch {
                // 原始渲染异常，忽略
              }
            } else {
              // 如果没有原始的 renderCell，使用格式化后的值
              renderedValue = formattedValue
            }
          }

          // 提取最终值（纯文本）
          let finalValue = ''
          if (renderedValue) {
            if (typeof renderedValue === 'object') {
              // 新增图片链接提取逻辑：如果是图片元素，直接获取src属性
              if (renderedValue.type === 'img' && renderedValue.props) {
                finalValue = renderedValue.props.src || ''
              } else {
                // 处理 VNode 或普通对象
                if (renderedValue.__v_isVNode) {
                  finalValue = extractValue(renderedValue)
                } else {
                  if ('value' in renderedValue) {
                    finalValue = renderedValue.value
                  } else if ('label' in renderedValue) {
                    finalValue = renderedValue.label
                  } else if ('text' in renderedValue) {
                    finalValue = renderedValue.text
                  } else if ('content' in renderedValue) {
                    finalValue = renderedValue.content
                  } else {
                    finalValue = extractValue(renderedValue)
                  }
                }
              }
            } else {
              finalValue = String(renderedValue)
            }
          }

          // ===== 新增逻辑 =====
          // 如果提取值为空或空字符串，则使用原始值
          if (finalValue === '' || finalValue == null) {
            // 首先尝试使用格式化后的值（如果有）
            if (formattedValue != null && formattedValue !== '') {
              finalValue = formattedValue
            }
            // 如果格式化值也无效，使用原始数据值
            else if (rawValue != null && rawValue !== '') {
              finalValue = rawValue
            }
          }
          // ====================

          const rowKey = getRowKey(scope.row)
          results.set(`${rowKey}_${column.id}`, finalValue)

          return null // 阻断真实渲染
        }
      })

      // 手动调用所有列的 renderCell 来收集数据（仅做虚拟转换，不影响页面）
      store.states.columns.value.forEach((column) => {
        data.forEach((row) => {
          const scope = {
            row,
            column,
            $index: data.indexOf(row),
            store: instance.store,
            _self: instance,
          }
          column.renderCell(scope)
        })
      })

      // 恢复原始的 renderCell 方法，防止后续表格渲染异常
      store.states.columns.value.forEach((column) => {
        column.renderCell = originalRenderCells.get(column.id)
      })

      // 处理并返回最终的纯文本数据
      const processedData = processData(results, data)
      return {
        data: processedData,
        restore: () => {
          // 已经恢复了原始的 renderCell 方法，这里不需要额外操作
        },
      }
    }

    /**
     * 处理虚拟渲染结果，组装为最终导出用的纯文本数据数组
     * @param results Map<string, any> 虚拟渲染收集到的单元格数据
     * @param data 原始数据数组
     * @returns 组装后的纯文本数据数组
     */
    const processData = (results: Map<string, any>, data: any[]) => {
      const rowKey = props.rowKey || 'id'
      const getRowKey = (row: any) => {
        if (typeof rowKey === 'function') {
          return rowKey(row)
        }
        return row[rowKey]
      }

      // 遍历原始数据，组装每一行的最终导出数据
      return data.map((row) => {
        const rendered = { ...row }
        store.states.columns.value.forEach((col) => {
          const key = `${getRowKey(row)}_${col.id}`
          if (results.has(key)) {
            rendered[col.property] = results.get(key)
          }
        })
        return rendered
      })
    }

    useKeyRender(table)

    return {
      getVirtualRenderData,
      processData,
      extractValue,
      ns,
      layout,
      store,
      handleHeaderFooterMousewheel,
      handleMouseLeave,
      tableId,
      tableSize,
      isHidden,
      isEmpty,
      renderExpanded,
      resizeProxyVisible,
      resizeState,
      isGroup,
      bodyWidth,
      tableBodyStyles,
      emptyBlockStyle,
      debouncedUpdateLayout,
      handleFixedMousewheel,
      /**
       * @description used in single selection Table, set a certain row selected. If called without any parameter, it will clear selection
       */
      setCurrentRow,
      /**
       * @description returns the currently selected rows
       */
      getSelectionRows,
      /**
       * @description used in multiple selection Table, toggle if a certain row is selected. With the second parameter, you can directly set if this row is selected
       */
      toggleRowSelection,
      /**
       * @description used in multiple selection Table, clear user selection
       */
      clearSelection,
      /**
       * @description clear filters of the columns whose `columnKey` are passed in. If no params, clear all filters
       */
      clearFilter,
      /**
       * @description used in multiple selection Table, toggle select all and deselect all
       */
      toggleAllSelection,
      /**
       * @description used in expandable Table or tree Table, toggle if a certain row is expanded. With the second parameter, you can directly set if this row is expanded or collapsed
       */
      toggleRowExpansion,
      /**
       * @description clear sorting, restore data to the original order
       */
      clearSort,
      /**
       * @description refresh the layout of Table. When the visibility of Table changes, you may need to call this method to get a correct layout
       */
      doLayout,
      /**
       * @description sort Table manually. Property `prop` is used to set sort column, property `order` is used to set sort order
       */
      sort,
      t,
      setDragVisible,
      context: table,
      computedSumText,
      computedEmptyText,
      tableLayout,
      scrollbarViewStyle,
      tableInnerStyle,
      scrollbarStyle,
      scrollBarRef,
      /**
       * @description scrolls to a particular set of coordinates
       */
      scrollTo,
      /**
       * @description set horizontal scroll position
       */
      setScrollLeft,
      /**
       * @description set vertical scroll position
       */
      setScrollTop,
    }
  },
})
</script>

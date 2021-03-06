<template>
  <el-dialog
    :title="title"
    class="history-dialog"
    :visible.sync="visibility"
    :width="style.width"
    @closed="reset"
  >
    <div class="top-bar">
      <el-input placeholder="搜索" v-model="keyword" clearable>
        <i slot="prefix" class="el-input__icon el-icon-search"></i>
      </el-input>

      <div v-if="filteredHistory.length" class="button-bar">
        <div>
          <el-button v-if="!checking" type="text" @click="checking = true"
            >选择</el-button
          >
          <el-button
            v-else-if="indeterminateChecked"
            type="text"
            @click="checkAll"
            >全选</el-button
          >
          <el-button v-else type="text" @click="clearCheck">清除</el-button>
          <el-button
            type="text"
            @click="remove"
            :disabled="!checkedItems.length"
            >删除</el-button
          >
        </div>
        <div>
          <el-button type="text" @click="removeAll">清空历史</el-button>
        </div>
      </div>
    </div>

    <lazy-list
      v-if="filteredHistory.length"
      class="history-list"
      :style="{ height: style.height }"
      :class="{ checking }"
      key-prop="key"
      :data="filteredHistory"
    >
      <div class="history-item" slot-scope="{ item: history }">
        <div class="time">{{ history.time }}</div>
        <div
          class="content"
          v-for="item of history.items"
          :key="item.timestamp"
        >
          <el-checkbox
            v-if="checking"
            class="checkbox"
            v-model="item.checked"
          />
          <div class="content-main">
            <code-panel
              v-if="item.type === 'json'"
              class="code-panel"
              :code="item.content"
              :collapse="4"
              :highlight="false"
              collapsed
            />
            <div v-else class="text">
              <span class="button-copy">
                <clipboard-button :text="item.content" />
              </span>
              <span>{{ item.content }}</span>
            </div>
          </div>
        </div>
      </div>
    </lazy-list>

    <div v-else class="empty-background"></div>
  </el-dialog>
</template>

<script>
import {
  mapGetters as globalMapGetters,
  mapMutations as globalMapMutations,
} from 'vuex'

import { format as timeAgo } from 'timeago.js'
import uuid from 'uuid/v4'
import CodePanel from '../../components/CodePanel'
import ClipboardButton from '../../components/ClipboardButton'
import LazyList from '../../components/LazyList'

export default {
  name: 'HistoryDialog',
  components: { LazyList, ClipboardButton, CodePanel },
  props: {
    visible: {
      type: Boolean,
      default: false,
    },

    title: {
      type: String,
      default: '历史记录',
    },

    clearConfirmTips: {
      type: String,
      default: '确定要清空吗？',
    },

    keyMap: {
      type: Object,
      default: () => ({}),
    },
  },

  data() {
    return {
      checking: false,
      visibility: this.$props.visible,
      style: this.getStyle(),
      keyword: '',
      history: [],
    }
  },

  computed: {
    ...globalMapGetters(['getHistory']),

    filteredHistory() {
      const { keyword, history } = this
      let list = null
      if (keyword) {
        const reg = new RegExp(
          keyword.replace(/[*.?+$^[\](){}|\\]/g, '\\$&'),
          'i'
        )
        list = history.filter(({ content }) => reg.test(content))
      } else {
        list = history
      }
      return this.getHistoryGroups(list)
    },

    checkedItems() {
      const checked = []
      for (const { items } of this.filteredHistory) {
        for (const item of items) {
          if (item.checked) {
            checked.push(item)
          }
        }
      }
      return checked
    },

    indeterminateChecked() {
      for (const { items } of this.filteredHistory) {
        for (const item of items) {
          if (!item.checked) {
            return true
          }
        }
      }
      return false
    },
  },
  watch: {
    visible(cur) {
      this.visibility = !!cur
    },
    visibility(visible) {
      this.$emit('update:visible', !!visible)
      if (visible) {
        this.style = this.getStyle()
        this.refresh()
      }
    },
  },

  methods: {
    ...globalMapMutations(['clearHistory', 'clearAllHistory']),

    getHistoryGroups(list) {
      const groups = {}
      for (const item of list) {
        const time = this.formatTime(item.timestamp)
        const group = groups[time] || []
        item.checked = false
        group.push(item)
        groups[time] = group
      }
      return Object.keys(groups).map((time) => ({
        time,
        key: uuid(),
        items: groups[time],
      }))
    },

    getStyle() {
      return {
        width: Math.min(Math.max(window.innerWidth - 200, 400), 600) + 'px',
        height: Math.min(Math.max(window.innerHeight - 350, 300), 400) + 'px',
      }
    },

    formatTime(timestamp) {
      return timeAgo(timestamp, 'zh_CN')
    },

    reset() {
      this.keyword = ''
      this.checking = false
    },

    checkAll() {
      for (const { items } of this.filteredHistory) {
        for (const item of items) {
          item.checked = true
        }
      }
    },

    clearCheck() {
      for (const { items } of this.filteredHistory) {
        for (const item of items) {
          item.checked = false
        }
      }
    },

    remove() {
      const items = [...this.checkedItems]
      this.removeHistory(items)
      this.$emit('remove', items)
    },

    removeAll() {
      this.$confirm(this.clearConfirmTips, '清空', {
        type: 'warning',
      }).then(
        () => {
          this.removeAllHistory()
          this.$emit('clear')
        },
        () => {}
      )
    },

    removeHistory(data) {
      const { keyMap } = this
      const { path, type } = Object.assign({}, keyMap)
      this.clearHistory({
        path,
        type,
        data,
      })
      this.refresh()
    },

    removeAllHistory() {
      const { keyMap } = this
      const { path, type } = Object.assign({}, keyMap)
      this.clearAllHistory({ path, type })
      this.refresh()
    },

    refresh() {
      const { keyMap } = this
      const { path, type } = Object.assign({}, keyMap)
      const history = this.getHistory(path, type)
      for (const item of history) {
        item.checked = false
      }
      this.history = history
    },
  },
}
</script>

<style lang="less">
.history-dialog {
  .el-dialog__header {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    padding-right: 50px;
  }
  .el-dialog__body {
    padding-top: 8px;
    .code-panel {
      .code-box {
        border-radius: 4px;
        .code-wrap {
          background-color: #f7f7f7;
          &:after {
            background-color: #f7f7f7;
          }
        }
      }
      .button-bar {
        background-color: #f7f7f7;
      }
    }
  }
}
</style>

<style lang="less" scoped>
.top-bar {
  .button-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: 12px;
    padding: 4px 0;
    border-bottom: 1px solid #e4e7ed;
    margin-top: 16px;

    .el-button {
      font-size: 12px;
      padding: 4px 0;
    }
  }
}

.history-list {
  color: #262626;

  &.checking {
    .content-main {
      margin-left: 22px;
    }
  }
}

.history-item {
  &:after {
    content: '';
    display: block;
    width: 0;
    height: 0;
    clear: both;
  }
  .time {
    font-size: 12px;
    line-height: 1em;
    color: #909399;
    margin: 32px 8px 8px 0;
    text-align: center;
  }
  .content {
    overflow: visible;
    margin-bottom: 8px;
    position: relative;

    .checkbox {
      position: absolute;
      left: 0;
      top: 50%;
      transform: translateY(-50%);
      z-index: 10;
      line-height: 1em;
    }

    .text {
      position: relative;
      padding: 8px;
      overflow: visible;
      white-space: normal;
      word-break: break-all;
      background-color: #f7f7f7;
      border-radius: 4px;

      .button-copy {
        position: absolute;
        right: 0;
        top: 8px;
        padding: 1px 20px 8px 16px;
        background-color: #f7f7f7;
        color: #8c8c8c;
        z-index: 10;
        line-height: normal;

        &:hover {
          color: #595959;
        }
      }
    }
  }
}

.empty-background {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 300px;

  &:after {
    content: '';
    background: url('../../assets/img/empty.svg') center center no-repeat;
    background-size: cover;
    width: 211px;
    height: 170px;
  }
}
</style>

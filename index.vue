<template>
  <div class="search-form-setting">
    <el-icon @click="handlerConfig">
      <Tools/>
    </el-icon>

    <el-dialog
      v-model="isShowConfig"
      :title="getLangValue('自定义搜索项')"
      :width="730"
      :close-on-click-modal="false"
      @close="cancelDialog"
      size="small"
      draggable
      destroy-on-close
    >
      <div class="tip-text-desc">Tip: {{ getLangValue('上下拖拽以排序') }}</div>
      <div class="head-text">
        <div class="f-l">{{ getLangValue('搜索项') }}</div>
        <div class="f-r">{{ getLangValue('是否展示') }}</div>
      </div>
      <ul class="fields-list">
        <li class="fields-list-li" v-for="(item,i) in newList" :data-prop="item.prop">
          <div class="left">
            <span class="move-wrapper"></span>
            <span class="fields-title">{{ item.label }}</span>
          </div>
          <div class="right">
            <el-switch v-model="checkList[item.prop]"/>
          </div>
        </li>
      </ul>

      <template #footer>
        <div class="dialog-footer">
          <el-button @click="submitConfig" type="primary">{{ getLangValue('确认') }}</el-button>
          <el-button @click="cancelDialog">{{ getLangValue('关闭') }}</el-button>
        </div>
      </template>

    </el-dialog>
  </div>
</template>

<script>
/**
 * 自定义搜索项组件
 * */
let sortableObject = null
import zh from './lang/zh.js'
import en from './lang/en.js'

export default {
  props: {
    //搜索项json数组
    searchFormList: {
      type: Array,
      default: () => []
    },

    //存储数据的索引，默认根据当前页面的路由path
    indexKey: {
      type: String
    },

    //axios请求工具
    request: {
      type: Function,
      required: true
    },

    //拖拽工具
    Sortable: {
      type: [Object, Function],
      required: true
    },

    //portal接口前缀
    apiPortalPath: {
      type: String,
      default: ''
    },

    //cookie工具
    jsCookie: {
      type: Object,
      required: true
    },
  },

  data() {
    return {
      isShowConfig: false,
      checkList: {},
      newList: [],
    }
  },

  computed: {
    //计算后的索引
    indexKeyComputed() {
      return this.indexKey || this.$route.path
    },
  },

  watch: {
    //实时同步列表的数据状态
    searchFormList: {
      handler(val, old) {
        val.forEach(item => {
          this.checkList[item.prop] = Boolean(item.isShow)
        })
        this.newList = val
      },
      deep: true,
      immediate: true,
    }
  },

  mounted() {
    this.fetchResultFromServer()
  },

  methods: {
    //查询用户自定义配置
    userCustomItemSelectByUrl(params) {
      return this.request({
        baseURL: this.apiPortalPath,
        url: '/userCustomItem/selectByUrl',
        method: 'GET',
        params
      })
    },

    ////用户自定义条件配置
    userCustomItemSave(data) {
      return this.request({
        baseURL: this.apiPortalPath,
        url: '/userCustomItem/save',
        method: 'POST',
        data
      })
    },

    //获取语言包值
    getLangValue(langKey) {
      const lang = this.jsCookie.get('language') || 'zh'
      let langObj = {}
      if (lang === 'zh') {
        langObj = zh
      } else {
        langObj = en
      }
      return langObj['searchFormSetting'][langKey]
    },

    //从服务同步最新的状态
    fetchResultFromServer() {
      this.userCustomItemSelectByUrl({
        menuUrl: this.indexKeyComputed
      }).then(res => {
        // 如果数据有记录，同步数据库的结果到传过来的结果中
        if (/^\[.*\]$/.test(res.data?.conditionItems || '')) {
          const tempList = JSON.parse(res.data?.conditionItems)
          const newList = JSON.parse(JSON.stringify(this.newList))
          newList.forEach(item => {
            const tempItem = tempList.find(item2 => item2.prop === item.prop)
            item.isShow = Boolean(tempItem.isShow)
            item.sortNum = tempItem.sortNum //同步排序结果
          })
          newList.sort((a, b) => a.sortNum - b.sortNum) //排序一下
          this.$emit('submit', newList) //提交更新到列表
        }
      })
    },

    // 快捷入口拖拽排序
    handlerDrop() {
      sortableObject = this.Sortable.create(this.$el.querySelector(".fields-list"), {
        handle: '.move-wrapper',
        filter: '.el-icon',
        animation: 300,
        onEnd: ({newIndex, oldIndex}) => {
          console.log('newIndex, oldIndex:', newIndex, oldIndex)
          // 通过index更改数据，获取拖拽排序之后的数据

        }
      });
    },

    cancelDialog() {
      this.isShowConfig = false
    },

    handlerConfig() {
      this.isShowConfig = true
      this.$nextTick(() => {
        this.handlerDrop();
      })
    },

    submitConfig() {
      //如果只剩下一个了
      if (Object.values(this.checkList).filter(item => !!item).length < 1) {
        this.$message.error('不能全部隐藏')
        return false
      }

      let tempList = JSON.parse(JSON.stringify(this.newList))
      const domList = this.$el.querySelector('.fields-list').querySelectorAll('.fields-list-li')
      for (let i = 0; i < domList.length; i++) {
        const tempItem = tempList.find(item => item.prop === domList[i].getAttribute('data-prop'))
        if (tempItem) {
          tempItem.sortNum = i
        }
      }

      tempList = tempList.sort((a, b) => a.sortNum - b.sortNum)
      tempList.forEach(item => {
        item.isShow = this.checkList[item.prop]
      })
      console.log('xxx:', JSON.parse(JSON.stringify(tempList)))

      //调接口存起来
      this.userCustomItemSave({
        menuUrl: this.indexKeyComputed,
        conditionItems: JSON.stringify(tempList)
      })

      this.$emit('submit', tempList)
      this.isShowConfig = false
      if (sortableObject && typeof sortableObject.destroy === 'function') {
        sortableObject.destroy();
      }
    },
  },
}
</script>

<style lang="scss" scoped>
.search-form-setting {
  display: inline-flex;
  vertical-align: middle;
  margin-left: 10px;

  .tip-text-desc {
    color: red;
    margin-bottom: 20px;
  }

  .head-text {
    display: flex;
    justify-content: space-between;
    background: #F8F8F9;
    color: #666666;
    height: 36px;
    line-height: 36px;
    padding: 0 15px;
  }

  .fields-list {
    .fields-list-li {
      display: flex;
      justify-content: space-between;
      border-bottom: 1px solid #DFE6EC;
      padding: 5px 15px;
      align-items: center;

      &:last-child {
        border-bottom: none;
      }

      .left {
        display: flex;
        justify-content: left;
        align-items: center;
      }

      .move-wrapper {
        width: 6px;
        height: 16px;
        background: url("./img/move-bg.png");
        background-size: 100% 100%;
        margin-right: 10px;
        cursor: move;
      }
    }
  }

  .el-icon {
    cursor: pointer;
    color: var(--yt-color-primary);
  }
}
</style>
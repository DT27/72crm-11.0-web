<template>
  <div
    v-loading="loading"
    class="approval-flow">
    <flexbox
      class="approval-flow__hd"
      justify="space-between">

      <div class="approval-flow__hd--left">
        <span
          class="flow-title">
          <i class="wk wk-approve" />审批流信息
        </span>

        <el-popover
          v-model="showFlowPopover"
          width="300"
          trigger="click">
          <check-flow
            ref="checkFlow"
            :show="showFlowPopover"
            :id="id"
            :examine-type="examineType"
            @close="showFlowPopover=false"/>
          <el-button
            slot="reference"
            type="text">查看历史审批流程</el-button>
        </el-popover>
      </div>


      <div class="approval-flow__hd--right">
        <el-button
          v-if="examineInfo.is_check == 1"
          class="xr-btn--green"
          icon="wk wk-success"
          @click="examineHandle(1)">通过</el-button>
        <el-button
          v-if="examineInfo.is_check == 1"
          class="xr-btn--red"
          icon="wk wk-close"
          @click="examineHandle(2)">拒绝</el-button>
        <el-button
          v-if="examineInfo.is_recheck == 1"
          class="xr-btn--primary"
          icon="wk wk-reset"
          @click="examineHandle(4)">撤回</el-button>
      </div>
    </flexbox>

    <!-- 授权 -->
    <flexbox
      v-if="examineInfo.config == 0"
      class="check-items">
      <flexbox
        v-for="(item, index) in examineInfo.stepList"
        :key="index"
        class="check-item">
        <img
          :src="getStatusImageIcon(item.type)"
          class="check-item-img">
        <div class="check-item-name">{{ item.userInfo.realname }}</div>
        <div class="check-item-status">{{ getStatusName(item.type) }}</div>
        <i
          v-if="examineInfo.stepList.length -1 != index"
          class="el-icon-arrow-right check-item-arrow"/>
      </flexbox>
    </flexbox>

    <!-- 固定 -->
    <flexbox
      v-else-if="examineInfo.config == 1"
      class="check-items"
      wrap="wrap">
      <el-popover
        v-for="(item, index) in examineInfo.stepList"
        :key="index"
        :disabled="!item.user_id_info || item.user_id_info.length==0"
        placement="bottom"
        trigger="hover">
        <div class="popover-detail">
          <flexbox
            v-for="(subItem, subIndex) in item.user_id_info"
            :key="subIndex"
            align="stretch"
            class="popover-detail-item">
            <img
              :src="getStatusImageIcon(subItem.check_type)"
              class="popover-detail-item-img">
            <div>
              <div class="popover-detail-item-time">{{ subItem.check_time }}</div>
              <flexbox class="popover-detail-item-examine">
                <div class="examine-name">{{ subItem.realname }}</div>
                <div class="examine-info">{{ getStatusName(subItem.check_type) }}此申请</div>
              </flexbox>
            </div>
          </flexbox>
        </div>
        <flexbox
          slot="reference"
          class="check-item">
          <img
            :src="getStatusImageIcon(item.type)"
            class="check-item-img">
          <div class="check-item-name">{{ item|detailName }}</div>
          <div class="check-item-status">{{ getStatusName(item.type) }}</div>
          <i
            v-if="examineInfo.stepList.length -1 != index"
            class="el-icon-arrow-right check-item-arrow"/>
        </flexbox>
      </el-popover>
    </flexbox>

    <examine-handle
      :show="showExamineHandle"
      :id="id"
      :record-id="recordId"
      :examine-type="examineType"
      :detail="examineInfo"
      :status="examineHandleInfo.status"
      @close="showExamineHandle = false"
      @save="examineHandleClick"/>
  </div>
</template>
<script type="text/javascript">
import { crmExamineRecordRecordListAPI } from '@/api/examine' // 审批步骤
import { oaExamineFlowStepListAPI } from '@/api/oa/examine'

import Nzhcn from 'nzh/cn'
import ExamineHandle from './ExamineHandle' // 审批操作理由
import CheckFlow from './CheckFlow' // 审批流程
import CheckStatusMixin from '@/mixins/CheckStatusMixin'

// 审核信息 config 1 固定 0 自选
export default {
  name: 'ExamineInfo', // 合同审核操作
  components: {
    ExamineHandle,
    CheckFlow
  },
  filters: {
    detailName: function(data) {
      if (data.status == 2) {
        return data.user_id_info.length + '人或签'
      } else if (data.status == 3) {
        return data.user_id_info.length + '人会签'
      } else if (data.status == 1) {
        return '负责人主管'
      } else if (data.status == 4) {
        return '上一级审批人主管'
      } else if (data.status == 5) {
        return data.user_id_info && data.user_id_info.length ? data.user_id_info[0].realname : ''
      }
    },
    stepName: function(index) {
      return '第' + Nzhcn.encodeS(index) + '级'
    }
  },
  mixins: [CheckStatusMixin],
  props: {
    examineType: {
      type: String,
      default: ''
    },
    // 详情信息id
    id: [String, Number],
    // 审批流id
    recordId: [String, Number],
    ownerUserId: [String, Number]
  },
  data() {
    return {
      loading: false,
      examineInfo: {}, // 审核信息
      showFlowPopover: false,
      examineHandleInfo: { status: 1 }, // 1 审核通过 2 审核拒绝 4 已撤回
      showExamineHandle: false // 审核操作
    }
  },
  computed: {},
  watch: {
    recordId: {
      handler(val) {
        if (val) {
          this.examineInfo = {}
          this.getFlowStepList()
          if (this.$refs.checkFlow) {
            this.$refs.checkFlow.getDetail()
          }
        }
      },
      deep: true,
      immediate: true
    }
  },
  mounted() {},
  methods: {
    getFlowStepList() {
      if (!this.recordId || !this.id) {
        return
      }
      this.loading = true
      const request = {
        crm_contract: crmExamineRecordRecordListAPI,
        crm_receivables: crmExamineRecordRecordListAPI,
        crm_invoice: crmExamineRecordRecordListAPI,
        oa_examine: oaExamineFlowStepListAPI
      }[this.examineType]

      request({
        flow_id: this.recordId,
        types: this.examineType,
        // ownerUserId: this.ownerUserId
        types_id: this.id,
        action: 'view'
      })
        .then(res => {
          this.loading = false
          this.examineInfo = res.data
          this.$emit('value-change', {
            config: res.data.examineType, // 审批类型
            value: [] // 审批信息
          })
        })
        .catch(() => {
          this.loading = false
        })
    },
    // 撤回审核 通过 拒绝
    examineHandle(status) {
      this.examineHandleInfo.status = status
      this.showExamineHandle = true
    },
    getContentFilters(array) {
      var content = ''
      for (let index = 0; index < array.length; index++) {
        const item = array[index]
        if (index == array.length - 1) {
          content =
            content + item.realname + '：' + this.getStatusName(item.checkType)
        } else {
          content =
            content +
            item.realname +
            '：' +
            this.getStatusName(item.checkType) +
            '、'
        }
      }
      return content
    },
    // 审批操作点击
    examineHandleClick(data) {
      this.getFlowStepList()
      if (this.$refs.checkFlow) {
        this.$refs.checkFlow.getDetail()
      }
      this.$emit('on-handle', data)
    }
  }
}
</script>
<style lang="scss" scoped>
.approval-flow{
  position: relative;
  background-color: white;
  padding: 15px;

  &__hd {
    position:relative;

    &--right {

      .xr-btn--green,
      .xr-btn--red {
        color: white;
      }
    }
  }
}

// 头部标示
.flow-title {
  font-weight: 600;
  color: #333333;
  margin-right: 20px;
  i {
    color: white;
    margin-right: 5px;
    padding: 3px;
    font-size: 12px;
    border-radius: $xr-border-radius-base;
    background-color: #1CBAF5;
  }
}

/** 审核流程 */
.check-items {
  overflow-x: auto;
  padding: 10px 0;
  margin-top: 10px;
}

.check-item {
  width: auto;
  flex-shrink: 0;
  .check-item-img {
    display: block;
    width: 18px;
    height: 18px;
    border-radius: 9px;
    margin-right: 8px;
  }
  .check-item-name {
    color: #333;
    font-size: 14px;
    margin-right: 8px;
  }
  .check-item-status {
    color: #666;
    font-size: 12px;
  }

  .check-item-arrow {
    color: #666;
    margin: 0 15px;
    font-size: 13px;
  }
}

// 固定审批流详情
.popover-detail {
  padding: 0 5px;
}
.popover-detail-item {
  padding: 5px 0;
  &-img {
    display: block;
    width: 16px;
    height: 16px;
    border-radius: 8px;
    margin-right: 10px;
  }
  &-time {
    color: #bababa;
    font-size: 12px;
  }
  &-examine {
    .examine-name {
      color: #333;
      margin-right: 10px;
    }
    .examine-info {
      color: #666;
    }
  }
}
</style>

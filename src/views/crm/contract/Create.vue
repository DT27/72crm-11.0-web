<template>
  <xr-create
    :loading="loading"
    :title="title"
    :confirm-button-text="sureBtnTitle"
    @close="close"
    @save="saveClick">
    <create-sections title="基本信息">
      <wk-form
        ref="crmForm"
        :model="fieldForm"
        :rules="fieldRules"
        :field-from="fieldForm"
        :field-list="fieldList"
        label-position="top"
        @change="formChange"
      >
        <template slot-scope="{ data }">
          <crm-relative-cell
            v-if="data && data.formType == 'customer'"
            :value="fieldForm[data.field]"
            :disabled="data.disabled"
            relative-type="customer"
            @value-change="otherChange($event, data)"
          />
          <crm-relative-cell
            v-if="data && (data.formType == 'business' || data.formType == 'contacts')"
            :value="fieldForm[data.field]"
            :disabled="data.disabled"
            :relation="data.relation"
            :relative-type="data.formType"
            @value-change="otherChange($event, data)"
          />
          <xh-product
            v-if="data && data.formType == 'product'"
            :value="fieldForm[data.field]"
            @value-change="otherChange($event, data)"
          />
        </template>
      </wk-form>
    </create-sections>
    <create-sections
      v-if="isOpenExamine"
      title="审核信息">
      <template slot="header">
        <div
          v-if="examineInfo.config===1 || examineInfo.config===0"
          class="examine-type">{{ examineInfo.config===1 ? '固定审批流' : '授权审批人' }}</div>
        <el-tooltip
          v-if="examineInfo && examineInfo.remark"
          :content="examineInfo.remark"
          effect="dark"
          placement="top">
          <i class="wk wk-help wk-help-tips" style="height: 13px;"/>
        </el-tooltip>
      </template>
      <create-examine-info
        ref="examineInfo"
        :types-id="action.id"
        types="crm_contract"
        @value-change="examineValueChange" />
    </create-sections>

    <el-button
      slot="footer"
      type="primary"
      plain
      @click.native="debouncedSaveField(true)">保存草稿</el-button>
  </xr-create>
</template>

<script>
import { filedGetFieldAPI } from '@/api/crm/common'
import { crmContractSaveAPI } from '@/api/crm/contract'
import { crmBusinessProductAPI } from '@/api/crm/business'

import XrCreate from '@/components/XrCreate'
import CreateSections from '@/components/CreateSections'
import WkForm from '@/components/NewCom/WkForm'
import {
  XhProduct,
  CrmRelativeCell
} from '@/components/CreateCom'
import CreateExamineInfo from '@/components/Examine/CreateExamineInfo'

import CustomFieldsMixin from '@/mixins/CustomFields'
import { debounce } from 'throttle-debounce'
import { objDeepCopy } from '@/utils'
// import { isEmpty } from '@/utils/types'

export default {
  // 新建编辑
  name: 'ContractCreate',

  components: {
    XrCreate,
    CreateSections,
    CrmRelativeCell,
    XhProduct,
    CreateExamineInfo,
    WkForm
  },

  mixins: [CustomFieldsMixin],

  props: {
    action: {
      type: Object,
      default: () => {
        return {
          type: 'save',
          id: '',
          data: {}
        }
      }
    }
  },

  data() {
    return {
      loading: false,
      baseFields: [],
      fieldList: [],
      fieldForm: {},
      fieldRules: {},

      // 审批信息
      examineInfo: {}
    }
  },

  computed: {
    title() {
      if (this.action.title) {
        return this.action.title
      }
      return this.action.type === 'update' ? '编辑合同' : '新建合同'
    },

    // 合同 回款 下展示审批人信息
    isOpenExamine() {
      if (this.examineInfo) {
        // 初始状态是空对象默认展示，请求之后，根据status判断
        return Object.keys(this.examineInfo).length > 0 ? this.examineInfo.examineStatus : true
      }
      return false
    },

    // 确认名称
    sureBtnTitle() {
      if (this.isOpenExamine) {
        return '提交审核'
      }
      return '保存'
    }
  },

  watch: {},

  created() {
    this.debouncedSaveField = debounce(300, this.saveClick)

    this.getField()
  },

  mounted() {},

  beforeDestroy() {},

  methods: {
    /**
     * 获取数据
     */
    getField() {
      this.loading = true
      const params = {
        // label: crmTypeModel.contract
        types: 'crm_contract',
        module: 'crm',
        controller: 'contract',
        action: this.action.type
      }

      if (this.action.type == 'update') {
        params.action_id = this.action.id
      }

      filedGetFieldAPI(params)
        .then(res => {
          const list = res.data || []

          const fieldList = []
          const fieldRules = {}
          const fieldForm = {}
          list.forEach(item => {
            const temp = {}
            temp.field = item.field
            temp.formType = item.form_type
            temp.fieldId = item.fieldId
            temp.inputTips = item.input_tips
            temp.name = item.name
            temp.setting = item.setting
            const canEdit = this.getItemIsCanEdit(item, this.action.type)
            // 是否能编辑权限
            if (canEdit) {
              // 自动生成编号
              if (item.autoGeneNumber == 1) {
                temp.placeholder = '根据编号规则自动生成，支持手动输入'
                temp.value = item.value ? item.value : 'ZN-HT-' + (new Date().getFullYear()) + '-'
                //item.value = 'ZN-HT-' + (new Date().getFullYear())
                const copyItem = objDeepCopy(item)
                copyItem.isNull = 0
                fieldRules[temp.field] = this.getRules(copyItem)
              } else {
                fieldRules[temp.field] = this.getRules(item)
              }
            }

            // 是否可编辑
            temp.disabled = !canEdit

            // 禁止某些业务组件选择
            if (temp.formType == 'contacts' ||
                temp.formType == 'customer' ||
                temp.formType == 'business'
            ) {
              if (this.action.type == 'relative') {
                const relativeDisInfos = {
                  contacts: { customer: true },
                  customer: { customer: true },
                  business: { customer: true, business: true }
                }

                // 在哪个类型下添加
                const relativeTypeDisInfos = relativeDisInfos[this.action.crmType]
                if (relativeTypeDisInfos) {
                  // 包含的字段值
                  temp.disabled = relativeTypeDisInfos[item.form_type] || false
                }
              } else if (this.action.type != 'update') {
                temp.disabled = item.form_type === 'business' || item.form_type === 'contacts'
              }
            }

            // 处理关联
            if ((this.action.type == 'relative' || this.action.type == 'update') && (item.form_type == 'business' || item.form_type == 'contacts' || item.form_type == 'contract'
            )) {
              const customerItem = this.getItemRelatveInfo(list, 'customer')
              if (customerItem) {
                if (item.form_type == 'business' || item.form_type == 'contacts') {
                  customerItem['moduleType'] = 'customer'
                  temp['relation'] = customerItem
                } else if (item.formType == 'contract') {
                  customerItem['moduleType'] = 'customer'
                  customerItem['params'] = { checkStatus: 2 }
                  temp['relation'] = customerItem
                }
              }
            }
            // 特殊字段允许多选
            this.getItemRadio(item, temp)

            // 获取默认值
            // 非编辑情况下 填充默认值
            if (this.action.type != 'update' && item.field === 'order_date') {
              fieldForm[temp.field] = this.$moment().format('YYYY-MM-DD')
            } else {
              fieldForm[temp.field] = this.getItemValue(item, this.action.data, this.action.type)
            }
            fieldList.push(temp)
          })


          // 填充商机关联的产品信息
          if (this.action.type == 'relative') {
            const businessData = this.action.data.business
            if (businessData) {
              this.getBusinessProduct(businessData.business_id).then(resData => {
                const businessData = resData || {}
                fieldForm.product = {
                  product: businessData.list,
                  total_price: businessData.total_price,
                  discount_rate: businessData.discount_rate
                }
                fieldForm.money = businessData.total_price || ''
              }).catch(() => {})
            }
          }

          this.baseFields = list
          this.fieldList = fieldList
          this.fieldForm = fieldForm
          this.fieldRules = fieldRules


          this.loading = false
        })
        .catch(() => {
          this.loading = false
        })
    },

    /**
     * 保存
     */
    saveClick(isDraft = false) {
      this.loading = true
      const crmForm = this.$refs.crmForm.instance
      crmForm.validate(valid => {
        if (valid) {
          if (this.isOpenExamine) {
            /** 验证审批数据 */
            if (isDraft) {
              // 不验证数据 验证审批人
              this.$refs.examineInfo.validateField((result) => {
                if (result) {
                  const params = this.getSubmiteParams(this.baseFields, this.fieldForm)
                  if (
                    this.examineInfo.config === 0 &&
                    this.examineInfo.hasOwnProperty('value') &&
                    this.examineInfo.value.length
                  ) {
                    params['check_user_id'] = this.examineInfo.value[0].id
                  }
                  params['examineStatus'] = this.examineInfo.examineStatus
                  params.is_draft = 1
                  this.submiteParams(params)
                } else {
                  this.loading = false
                }
              })
            } else {
              this.$refs.examineInfo.validateField((result) => {
                if (result) {
                  const params = this.getSubmiteParams(this.baseFields, this.fieldForm)
                  if (this.examineInfo.config === 0) {
                    params['check_user_id'] = this.examineInfo.value[0].id
                  }
                  params['examineStatus'] = this.examineInfo.examineStatus
                  this.submiteParams(params)
                } else {
                  this.loading = false
                }
              })
            }
          } else {
            const params = this.getSubmiteParams(this.baseFields, this.fieldForm)
            if (isDraft) {
              params.is_draft = 1
            }
            this.submiteParams(params)
          }
        } else {
          this.loading = false
          // 提示第一个error
          this.getFormErrorMessage(crmForm)
          return false
        }
      })
    },

    /**
     * 提交上传
     */
    submiteParams(params) {
      if (this.action.type == 'update' && !this.action.isCopy) {
        params.id = this.action.id
        params.batchId = this.action.batchId
      }

      // 相关添加时候的多余提交信息
      if (
        this.action.relativeData &&
        Object.keys(this.action.relativeData).length
      ) {
        params = { ...params, ...this.action.relativeData }
      }

      crmContractSaveAPI(params)
        .then(res => {
          this.loading = false
          if (this.action.isCopy) {
            this.$message.success('操作成功')
          } else {
            this.$message.success(
              this.action.type == 'update' ? '编辑成功' : '添加成功'
            )
          }


          this.close()

          // 保存成功
          this.$emit('save-success', {
            type: 'contract'
          })
        })
        .catch(() => {
          this.loading = false
        })
    },

    /**
     * 验证唯一
     */
    UniquePromise({ field, value }) {
      return this.getUniquePromise(field, value, this.action)
    },

    /**
     * change
     */
    formChange(field, index, value, valueList) {
    },

    /**
     * 地址change
     */
    otherChange(data, field) {
      if (field.formType === 'customer') {
        let contractForCount = 0
        for (let index = 0; index < this.fieldList.length; index++) {
          const element = this.fieldList[index]
          // 需要处理 需关联客户信息或客户下信息
          const handleFields = [
            'business_id',
            'contacts_id',
            'order_user_id'
          ]

          // 添加请求关联
          const addRelation = ['business_id', 'contacts_id']

          // 需要disabled
          const addDisabled = ['business_id', 'contacts_id']

          // 复制
          const getValueObj = {
            contacts_id: data => {
              if (!data.contacts_id) {
                return []
              }
              return [
                {
                  name: data.contactsName || '',
                  contacts_id: data.contacts_id
                }
              ]
            },
            order_user_id: data => {
              if (!data.ownerUserId) {
                return []
              }
              return [
                {
                  realname: data.owner_user_name || '',
                  id: data.owner_user_id
                }
              ]
            }
          }

          if (handleFields.includes(element.field)) {
            if (data.value.length > 0) {
              element.disabled = false

              // 增加关联信息
              const customerItem = data.value[0]
              if (addRelation.includes(element.field)) {
                customerItem['moduleType'] = 'customer'
                element['relation'] = customerItem
              }

              // 填充值
              if (getValueObj[element.field]) {
                this.fieldForm[element.field] = getValueObj[element.field](customerItem)
              } else {
                this.fieldForm[element.field] = []
              }
            } else {
              // 禁用
              element.disabled = !!addDisabled.includes(element.field)

              if (addRelation.includes(element.field)) {
                element['relation'] = {}
              }

              this.fieldForm[element.field] = []
            }

            contractForCount++
            if (contractForCount == handleFields.length) {
              break
            }
          }
        }

        // 重置产品信息
        this.fieldForm.product = {
          product: [],
          total_price: '',
          discount_rate: ''
        }
        this.fieldForm.money = ''
      } else if (field.formType === 'business') {
        if (data.value.length > 0) {
          this.getBusinessProduct(data.value[0].business_id).then(resData => {
            const businessData = resData || {}
            this.fieldForm.product = {
              product: businessData.list,
              total_price: businessData.total_price,
              discount_rate: businessData.discount_rate
            }
            this.fieldForm.money = businessData.total_price || ''
          }).catch(() => {})
        }
      } else if (field.formType === 'product') {
        this.fieldForm.money = data.value.total_price || ''
      }
      this.$set(this.fieldForm, field.field, data.value)
      this.$refs.crmForm.instance.validateField(field.field)
    },

    /**
     * 获取商机的产品信息进行填充
     */
    getBusinessProduct(business_id) {
      return new Promise((resolve, reject) => {
        crmBusinessProductAPI({
          business_id: business_id,
          pageType: 0
        })
          .then(res => {
            resolve(res.data)
          })
          .catch(err => {
            reject(err)
          })
      })
    },

    /**
     * 审批信息值更新
     */
    examineValueChange(data) {
      this.examineInfo = data
    },

    /**
     * 关闭
     */
    close() {
      this.$emit('close')
    }
  }
}
</script>

<style lang="scss" scoped>
.wk-form {
  /deep/ .el-form-item.is-product {
    flex: 0 0 100%;
  }
}
</style>

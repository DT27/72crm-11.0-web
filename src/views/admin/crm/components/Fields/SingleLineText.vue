<template>
  <div class="box-wrapper">
    <div class="title">
      <span>{{ attr.is_null ? '*' : '' }}</span>{{ attr.name }}<span v-if="attr.input_tips">{{ '（'+attr.input_tips+'）' }}</span>
    </div>
    <div class="box">
      <span class="default-val">{{ typeof attr.default_value == 'string' ? attr.default_value : '' }}</span>
    </div>
    <span
      v-if="isShow"
      class="el-icon-delete control"
      @click="handleDelete"/>
  </div>
</template>

<script>
/**
 * 单行文本
 */
import mixins from './mixin'

export default {
  name: 'SingleLineText',
  mixins: [mixins]
}
</script>

<style scoped lang="scss">
@import '@/styles/mixin.scss';
@import 'form.scss';

.box {
  width: 310px;
  height: 32px;
  padding: 0 10px;
  border: 1px solid #dcdfe6;
  border-radius: $xr-border-radius-base;
  background: white;
  font-size: 14px;
  @include left;
  .placeholder-val {
    color: #ccc;
  }
  .default-val {
    color: #333;
  }
}
</style>

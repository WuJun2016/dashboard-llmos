<script>
import KeyValue from '@shell/components/form/KeyValue';
import Question from './Question';

export default {
  name:       'QuestionMap',
  components: { KeyValue },
  mixins:     [Question],

  methods: {
    update(val) {
      this.$emit('input', val);
    }
  }
};
</script>

<template>
  <div>
    <div
      v-if="showDescription"
      class="row mt-10"
    >
      <div class="col span-12">
        {{ question.description }}
      </div>
    </div>
    <div class="row">
      <div class="col span-12 mt-10">
        <KeyValue
          v-model:value="value"
          :title="question.label"
          :mode="mode"
          :protip="false"
          :disabled="disabled"
          :title-protip="displayTooltip"
          @update:value="update"
        />
      </div>
    </div>
  </div>
</template>

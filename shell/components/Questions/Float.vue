<script>
import { LabeledInput } from '@shell/components/form/LabeledInput';
import Question from './Question';

export default {
  components: { LabeledInput },
  mixins:     [Question]
};
</script>

<template>
  <div
    :data-testid="`float-row-${question.variable}`"
    class="row"
  >
    <div class="col span-6">
      <LabeledInput
        type="text"
        :mode="mode"
        :label="displayLabel"
        :placeholder="question.default"
        :required="question.required"
        :value="value"
        :disabled="disabled"
        :tooltip="displayTooltip"
        :rules="rules"
        :data-testid="`float-input-${question.variable}`"
        @input="val = parseFloat($event); if ( !isNaN(val) ) { $emit('input', val) }"
      />
    </div>
    <div
      v-if="showDescription"
      :data-testid="`float-description-${question.variable}`"
      class="col span-6 mt-10"
    >
      {{ question.description }}
    </div>
  </div>
</template>

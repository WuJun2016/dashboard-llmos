<script>
import CruResource from '@shell/components/CruResource';
import { LabeledInput } from '@shell/components/form/LabeledInput';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { Banner } from '@shell/components/Banner';
import CreateEditView from '@shell/mixins/create-edit-view';
import { TextAreaAutoGrow } from '@shell/components/form/TextArea';
import formRulesGenerator from '@shell/utils/validators/formRules/index';

import { ALLOWED_SETTINGS, SETTING } from '@shell/config/settings';
import { RadioGroup } from '@shell/components/form/Radio';
import FormValidation from '@shell/mixins/form-validation';
import { setBrand } from '@shell/config/private-label';
import { keyBy, mapValues } from 'lodash';
import { isLocalhost, isServerUrl } from '@shell/utils/validators/setting';
import CodeMirror from '@shell/components/CodeMirror.vue';

export default {
  components: {
    CruResource,
    LabeledInput,
    LabeledSelect,
    RadioGroup,
    TextAreaAutoGrow,
    Banner,
    CodeMirror,
  },

  mixins: [CreateEditView, FormValidation],

  data() {
    const t = this.$store.getters['i18n/t'];

    const cmOptions = {
      mode: {
        name: 'javascript',
        json: true,
      },
    };

    return {
      setting:        ALLOWED_SETTINGS[this.value.id],
      description:    t(`advancedSettings.descriptions.${ this.value.id }`),
      editHelp:       t(`advancedSettings.editHelp.${ this.value.id }`),
      enumOptions:    [],
      canReset:       false,
      errors:         [],
      fvFormRuleSets: [],
      cmOptions,
    };
  },

  created() {
    if (this.value.value === undefined) {
      this.value['value'] = this.value.default;
    }
    this.value.value = this.value.value || this.value.default;
    if (this.setting?.kind === 'json') {
      const jsonVal = JSON.parse(this.value.value);

      this.value.value = JSON.stringify(jsonVal, null, 2);
    }

    this.enumOptions = this.setting?.kind === 'enum' ? this.setting.options.map((id) => ({
      // i18n-uses advancedSettings.enum.*
      label: `advancedSettings.enum.${ this.value.id }.${ id }`,
      value: id,
    })) : [];
    this.canReset = this.setting?.canReset || !!this.value.default;
    this.fvFormRuleSets = this.setting?.ruleSet ? [
      {
        path:  'value',
        rules: this.setting.ruleSet.map(({ name }) => name),
      },
    ] : [];

    // Don't allow the user to reset the server URL if there is no default
    // helps to ensure that a value is always set
    if (isServerUrl(this.value.id) && !this.value.default) {
      this.canReset = false;
    }
  },

  computed: {
    fvExtraRules() {
      const t = this.$store.getters['i18n/t'];

      // We map the setting rulesets to use values to define validation from factory
      return this.setting?.ruleSet ? mapValues(
        keyBy(this.setting.ruleSet, 'name'),
        // The validation is curried and may require the factory argument for the ValidatorFactory
        ({ key, name, factoryArg }) => {
          const rule = formRulesGenerator(t, key ? { key } : {})[name];

          return factoryArg ? rule(factoryArg) : rule;
        }
      ) : {};
    },

    showLocalhostWarning() {
      return isServerUrl(this.value.id) && isLocalhost(this.value.value);
    },

    showWarningBanner() {
      return this.setting?.warning;
    },

    validationPassed() {
      return this.fvFormIsValid && this.fvGetPathErrors(['value']).length === 0;
    },
  },

  methods: {
    convertToString(event) {
      this.value.value = `${ event.target.value }`;
    },

    saveSettings(done) {
      const t = this.$store.getters['i18n/t'];

      // Validate the JSON if the setting is a json value
      if (this.setting.kind === 'json') {
        try {
          JSON.parse(this.value.value);
          this.errors = [];
        } catch (e) {
          this.errors = [t('advancedSettings.edit.invalidJSON')];

          return done(false);
        }
      }

      if (this.value?.id === SETTING.BRAND) {
        setBrand(this.value.value);
      }

      this.save(done);
    },

    useDefault(ev) {
      // Lose the focus on the button after click
      if (ev && ev.srcElement) {
        ev.srcElement.blur();
      }

      if (isServerUrl(this.value.id) && !this.value.default) {
        return;
      }

      this.value.value = this.value.default;
    },

    onInput(value) {
      this.value.value = value;
    },
  },
};
</script>

<template>
  <CruResource
    class="route"
    :done-route="'c-cluster-product-resource'"
    :errors="fvUnreportedValidationErrors"
    :mode="mode"
    :resource="value"
    :subtypes="[]"
    :can-yaml="false"
    :validation-passed="validationPassed"
    @error="(e) => (errors = e)"
    @finish="saveSettings"
    @cancel="done"
  >
    <Banner
      v-if="showWarningBanner"
      color="warning"
      :label="t(`advancedSettings.warnings.${setting.warning}`)"
      data-testid="advanced_settings_warning_banner"
    />

    <h4>{{ description }}</h4>

    <h5
      v-if="editHelp"
      v-clean-html="editHelp"
      class="edit-help"
    />

    <div class="edit-change mt-20">
      <h5 v-t="'advancedSettings.edit.changeSetting'" />
      <a-button
        data-testid="advanced_settings_use_default"
        :disabled="!canReset"
        type="primary"
        @click="useDefault"
      >
        {{ t('advancedSettings.edit.useDefault') }}
      </a-button>
    </div>

    <Banner
      v-if="showLocalhostWarning"
      color="warning"
      :label="t('validation.setting.serverUrl.localhost')"
      data-testid="setting-serverurl-localhost-warning"
    />

    <Banner
      v-for="(err, i) in fvGetPathErrors(['value'])"
      :key="i"
      color="error"
      :label="err"
      data-testid="setting-error-banner"
    />

    <div class="mt-20">
      <div v-if="setting.kind === 'enum'">
        <LabeledSelect
          v-model:value="value.value"
          data-testid="input-setting-enum"
          :label="t('advancedSettings.edit.value')"
          :rules="fvGetAndReportPathRules('value')"
          :localized-label="true"
          :mode="mode"
          :required="true"
          :options="enumOptions"
        />
      </div>
      <div v-else-if="setting.kind === 'boolean'">
        <RadioGroup
          v-model:value="value.value"
          data-testid="input-setting-boolean"
          name="settings_value"
          :rules="fvGetAndReportPathRules('value')"
          :labels="[
            t('advancedSettings.edit.trueOption'),
            t('advancedSettings.edit.falseOption'),
          ]"
          :options="['true', 'false']"
        />
      </div>
      <div v-else-if="setting.kind === 'multiline'">
        <TextAreaAutoGrow
          v-model:value="value.value"
          data-testid="input-setting-json"
          :required="true"
          :rules="fvGetAndReportPathRules('value')"
          :min-height="254"
        />
      </div>
      <div v-else-if="setting.kind === 'json'">
        <CodeMirror
          ref="cm"
          :value="value.value"
          :options="cmOptions"
          @onInput="onInput"
        />
      </div>
      <div v-else-if="setting.kind === 'integer'">
        <LabeledInput
          v-model:value="value.value"
          data-testid="input-setting-integer"
          :label="t('advancedSettings.edit.value')"
          :mode="mode"
          type="number"
          :rules="fvGetAndReportPathRules('value')"
          :required="true"
        />
      </div>
      <div v-else>
        <LabeledInput
          v-model:value="value.value"
          data-testid="input-setting-generic"
          :localized-label="true"
          :required="true"
          :mode="mode"
          :rules="fvGetAndReportPathRules('value')"
          :label="t('advancedSettings.edit.value')"
        />
      </div>
    </div>
  </CruResource>
</template>

<style lang="scss" scoped>
.edit-change {
  align-items: center;
  display: flex;

  > h5 {
    flex: 1;
  }
}

:deep() .edit-help code {
  padding: 1px 5px;
}
</style>

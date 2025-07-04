<script>
import AsyncButton from '@shell/components/AsyncButton';
import { Banner } from '@shell/components/Banner';
import { RadioGroup } from '@shell/components/form/Radio';
import UnitInput from '@shell/components/form/UnitInput';
import { _EDIT, _VIEW } from '@shell/config/query-params';

import { exceptionToErrorsArray } from '@shell/utils/error';

export default {
  components: {
    AsyncButton,
    Banner,
    RadioGroup,
    UnitInput,
  },

  props: {
    kubeNodes: {
      type:    Array,
      default: () => [],
    },

    normanNodeId: {
      type:    String,
      default: '',
    },
  },

  data() {
    return {
      radioOptions: [
        {
          label: this.t('generic.yes'),
          value: true,
        },
        {
          label: this.t('generic.no'),
          value: false,
        },
      ],
      gracePeriodOptions: [
        {
          label: this.t('drainNode.gracePeriod.default'),
          value: false,
        },
        {
          label: this.t('drainNode.gracePeriod.custom'),
          value: true,
        },
      ],
      timeoutOptions: [
        {
          label: this.t('drainNode.timeout.default'),
          value: false,
        },
        {
          label: this.t('drainNode.timeout.custom'),
          value: true,
        },
      ],

      gracePeriod: false,
      timeout:     false,

      body: {
        deleteLocalData: false,
        force:           false,
        gracePeriod:     null,
        timeout:         null,
      },

      EDIT: _EDIT,
      VIEW: _VIEW,

      errors: [],
    };
  },

  watch: {
    gracePeriod(neu) {
      if (neu && !this.body.gracePeriod) {
        this.body.gracePeriod = 30;
      }
    },
    'body.gracePeriod'(neu) {
      if (neu && neu < 1) {
        this.body['gracePeriod'] = 1;
      }
    },
    timeout(neu) {
      if (neu && !this.body.timeout) {
        this.body.timeout = 60;
      }
    },
    'body.timeout'(neu) {
      if (neu) {
        if (neu < 1) {
          this.body['timeout'] = 1;
        } else if (neu > 10800) {
          this.body['timeout'] = 10800;
        }
      }
    },
  },

  methods: {
    close() {
      this.$emit('close');
    },

    async apply(buttonDone) {
      const { gracePeriod, timeout, ...parsedBody } = this.body;

      if (this.gracePeriod) {
        parsedBody.gracePeriod = gracePeriod;
      }
      if (this.timeout) {
        parsedBody.timeout = timeout;
      }

      try {
        await Promise.all(
          this.kubeNodes.map((node) => node.norman?.doAction('drain', parsedBody)
          )
        );
        this.close();
      } catch (e) {
        this.errors = exceptionToErrorsArray(e);
        buttonDone(false);
      }
    },
  },
};
</script>

<template>
  <a-card
    class="prompt-rotate"
    :show-highlight-border="false"
  >
    <h4
      slot="title"
      class="text-default-text"
    >
      <template v-if="kubeNodes.length > 1">
        {{ t('drainNode.titleMultiple', { count: kubeNodes.length }) }}
      </template>
      <template v-else>
        {{ t('drainNode.titleOne', { name: kubeNodes[0].name }, true) }}
      </template>
    </h4>

    <div>
      <RadioGroup
        v-model:value="body.deleteLocalData"
        name="deleteLocalData"
        :options="radioOptions"
        :row="true"
        class="mb-15"
      >
        <template #label>
          <h5>{{ t('drainNode.deleteLocalData') }}</h5>
        </template>
      </RadioGroup>
      <RadioGroup
        v-model:value="body.force"
        name="force"
        :options="radioOptions"
        :row="true"
        class="mb-15"
      >
        <template #label>
          <h5>{{ t('drainNode.force') }}</h5>
        </template>
      </RadioGroup>
      <RadioGroup
        v-model:value="gracePeriod"
        name="gracePeriod"
        :options="gracePeriodOptions"
        class="mb-15"
      >
        <template #label>
          <h5>{{ t('drainNode.gracePeriod.title') }}</h5>
        </template>
      </RadioGroup>
      <UnitInput
        v-model:value="body.gracePeriod"
        :mode="gracePeriod ? EDIT : VIEW"
        type="number"
        min="1"
        :suffix="t('suffix.sec')"
        :placeholder="t('drainNode.gracePeriod.placeholder')"
        class="mb-10"
      />
      <RadioGroup
        v-model:value="timeout"
        name="timeout"
        :options="timeoutOptions"
        class="mb-15"
      >
        <template #label>
          <h5>{{ t('drainNode.timeout.title') }}</h5>
        </template>
      </RadioGroup>
      <UnitInput
        v-model:value="body.timeout"
        :mode="timeout ? EDIT : VIEW"
        type="number"
        min="1"
        max="10800"
        :suffix="t('suffix.sec')"
        :placeholder="t('drainNode.timeout.placeholder')"
      />
    </div>
    <Banner
      v-for="(err, i) in errors"
      :key="i"
      color="error"
      :label="err"
    />

    <div
      slot="actions"
      class="buttons"
    >
      <a-button
        @click="close"
      >
        {{ t('generic.cancel') }}
      </a-button>

      <AsyncButton
        mode="drain"
        @click="apply"
      />
    </div>
  </a-card>
</template>
<style lang="scss" scoped>
.prompt-rotate {
  margin: 0;
}
.card-title h4 {
  margin-bottom: 0;
}
.buttons {
  display: flex;
  justify-content: flex-end;
  width: 100%;
}
</style>

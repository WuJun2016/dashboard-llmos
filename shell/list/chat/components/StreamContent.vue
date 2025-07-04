<template>
  <div class="mt-10">
    <EditInput
      v-if="message.role === 'user'"
      :value="message.content"
      :canEdit="props.message.canRegenerate"
      @update:value="handleRegenerate"
    />

    <template v-else>
      <div class="group">
        <div v-if="hasReasoning">
          <a-button
            class="mb-10"
            @click="toggleShowThinking"
          >
            <div class="flex items-center">
              {{ hasContent ? t('chat.thinkDone') : t('chat.thinking') }}
              <DownOutlined v-if="showThinking" />
              <UpOutlined v-else />
            </div>
          </a-button>

          <div
            v-if="showThinking"
            class="border-l-2 border-[#e5e5e5] pl-4"
          >
            <MarkedView
              :showLoading="loading"
              color="rgba(0, 0, 0, 0.5)"
              :content="props.message.reasoning"
            />
          </div>
        </div>

        <MarkedView
          :showLoading="loading"
          :content="props.message.content"
        />

        <ChatAction
          v-if="showAction"
          class="group-hover:opacity-100 opacity-0 transition-opacity duration-300"
          :showRegenerate="props.message.canRegenerate"
          @regenerate="handleRegenerate"
          @copy="handleCopy"
        />
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { DownOutlined, UpOutlined } from '@ant-design/icons-vue';
import MarkedView from '@shell/components/MarkedView';
import EditInput from '@shell/components/AI/EditInput.vue';
import ChatAction from '@shell/components/ChatAction.vue';

const emits = defineEmits(['regenerate']);
const props = defineProps({
  message: {
    type:     Object,
    required: true,
    default:  () => ({
      role:    '',
      content: '',
    }),
  },
  loading: {
    type:    Boolean,
    default: true,
  },
  uuid: {
    type:     String,
    required: true,
  },
});

const hasReasoning = computed(() => {
  return props.message.reasoning && props.message.reasoning.length > 0;
});

const hasContent = computed(() => {
  return props.message.content && props.message.content.length > 0;
});

const showAction = computed(() => {
  return props.message.content || props.message.isStop;
});

// action
const handleCopy = async() => {
  try {
    await navigator.clipboard.writeText(props.message.content);
  } catch (err) {
  }
};

const handleRegenerate = (question) => {
  emits('regenerate', props.uuid, question);
};

const showThinking = ref(true);
const userControl = ref(false);

const toggleShowThinking = () => {
  showThinking.value = !showThinking.value;
  userControl.value = true;
};

watch(() => props.message.content, () => {
  if (props.message.content && !userControl.value) {
    showThinking.value = false;
  }
}, { immediate: true });
</script>

<script>
export default {
  setup() {

  }
};
</script>

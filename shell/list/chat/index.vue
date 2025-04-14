<template>
  <div>
    <h1>Test2</h1>
    <Header class="mb-10" />

    <!-- <div class="flex grow-1 border-t-1 border-gray-200">
      <div class="flex grow-1 flex-col flex-wrap px-5">
        <a-collapse
          v-if="isChatType"
          v-model="openSystemPrompt"
          class="w-full mt-10"
        >
          <a-collapse-panel key="1" :header="t('chat.label.systemPrompt')">
            <SystemPrompt
              :systemPrompt="activeSystemPrompt"
              @update:systemPrompt="updateSystemPrompt"
            />
          </a-collapse-panel>
        </a-collapse>

        <a-row
          class="h-[300px] scroll-smooth flex grow-1 overflow-hidden my-5"
          justify="center"
        >
          <a-col
            v-if="isChatType"
            :md="22"
            :lg="22"
            :xl="18"
            :xxl="12"
            class="h-full"
          >
            <div class="overflow-y-scroll hide-scrollbar h-full">
              <ChatFlex>
                <StreamContent
                  v-for="(message, i) in filterSystemMessages.slice().reverse()"
                  :key="i"
                  :message="message"
                  :loading="message.isStop ? false : loading"
                  :uuid="activeUUID"
                  @regenerate="regenerate"
                />
              </ChatFlex>
            </div>
          </a-col>

          <a-col
            v-else
            :lg="22"
            :xl="22"
            :xxl="22"
            class="h-full overflow-hidden"
          >
            <div class="h-full flex flex-row">
              <div class="h-full flex flex-col grow-1">
                <a-row :gutter="[16, 16]" class="flex-grow">
                  <a-col
                    v-for="(uuid, idx) in compareChatIds"
                    :key="uuid"
                    class="flex"
                    :span="12"
                  >
                    <ModelCard
                      ref="modelComp"
                      :key="uuid"
                      :uuid="uuid"
                      :question="question"
                      @update:messages="updateModelMessages"
                      @handle:action="handleAction"
                    >
                      <template #title> # {{ idx + 1 }} </template>
                    </ModelCard>
                  </a-col>
                </a-row>
              </div>
            </div>
          </a-col>
        </a-row>
        <Banner
          v-if="!activeConfig.model"
          color="warning"
          :label="t('chat.emptyModelTip')"
        />
        <ChatInput
          v-model="question"
          :disabled="!activeConfig.model"
          :loading="isChatType ? loading : compareFetchLoading"
          @submit="submit"
          @update:abort="callbackAbortFetch"
        />
      </div>

      <SideConfig
        v-model:drawerOpen="drawerOpen"
        :activeConfig="activeConfig"
        @update:model="updateModel"
      />
    </div> -->
  </div>
</template>

<script>
import { ML_WORKLOAD_TYPES } from "@shell/config/types";
import { matchModelString } from "@shell/utils/ai-model";
import ChatFlex from "@shell/components/ChatFlex.vue";
import Header from "./components/Header/index.vue";
import ModelCard from "./components/ModelCard.vue";
import ChatInput from "./components/ChatInput.vue";
import StreamContent from "./components/StreamContent.vue";
import useChat from "./composables/useChat";
import SystemPrompt from "@shell/list/chat/components/SystemPrompt.vue";
import SideConfig from "@shell/list/chat/components/SideConfig.vue";
import { cloneDeep } from "lodash";
import { Banner } from "@shell/components/Banner";

export default {
  components: {
    ChatFlex,
    Header,
    ModelCard,
    ChatInput,
    StreamContent,
    SystemPrompt,
    SideConfig,
    Banner,
  },
  data() {
    return {
      drawerOpen: true,
      url: "",
      question: "",
      model: "",
      icon: "",
      modelDisplayName: "",
      openSystemPrompt: "",
      abortFetch: null,
      activeConfig: {},
      activeSystemPrompt: "",
      loading: false,
    };
  },
  computed: {
    isChatType() {
      return this.$store.getters["chat/isChatType"];
    },
    singleChatMessages() {
      return this.$store.getters["chat/chatMessages"]();
    },
    filterSystemMessages() {
      return this.singleChatMessages.filter((item) => item.role !== "system");
    },
    compareFetchLoading() {
      const loadings = this.$refs.modelComp?.map((comp) => comp?.loading) || [];
      return loadings.every((item) => item === true) || false;
    },
    chatsConfig() {
      return this.$store.state.chat.chatsConfig;
    },
    activeUUID: {
      get() {
        return this.$store.state.chat.activeUUID;
      },
      set(value) {
        this.$store.commit("chat/SET_ACTIVE_UUID", value);
      },
    },
    compareChatIds() {
      return this.$store.getters["chat/compareChatIds"];
    },
  },
  watch: {
    model: {
      immediate: true,
      deep: true,
      handler() {
        if (this.model) {
          this.modelDisplayName = this.model.modelName;
          this.icon = matchModelString(this.modelDisplayName);
        }
      },
    },
    activeUUID: {
      immediate: true,
      handler() {
        this.activeConfig = this.chatsConfig[this.activeUUID];
      },
    },
    isChatType: {
      immediate: true,
      handler() {
        if (!this.isChatType) {
          this.activeUUID = this.compareChatIds[0];
        } else {
          this.activeUUID = "single";
        }
      },
    },
    "$store.getters['chat/chatMessages'](activeUUID)": {
      immediate: true,
      handler() {
        if (this.activeUUID) {
          this.activeSystemPrompt = this.$store.getters["chat/chatMessages"](
            this.activeUUID
          )[0].content;
        }
      },
    },
  },
  created() {
    this.loadFetch();
  },
  methods: {
    async loadFetch() {
      await this.$store.dispatch("cluster/findAll", {
        type: ML_WORKLOAD_TYPES.MODEL_SERVICE,
      });
    },
    toggleDrawerOpen(value) {
      this.drawerOpen = value;
    },
    async submit(_question) {
      this.question = "";
      if (this.isChatType) {
        const { send, loading } = useChat(this.url, {
          messages: this.singleChatMessages,
          onUpdate: (chunk) => {
            this.$store.commit("chat/PUSH_CHAT_COMPLETIONS_CHUNK", { chunk });
          },
          beforeFetch: (options) => {
            const assistantRole = cloneDeep({
              role: "assistant",
              content: "",
              reasoning: "",
            });
            const userRole = cloneDeep({
              role: "user",
              content: options.payload.question,
            });
            const isRegenerate = options?.payload?.isRegenerate;
            this.$store.commit("chat/PUSH_CHAT_COMPLETIONS_CHUNK", {
              chunk: isRegenerate ? assistantRole : [userRole, assistantRole],
              type: "single",
              addRole: true,
            });
            options.body = JSON.stringify({
              ...options.body,
              messages: this.singleChatMessages
                .map(({ role, content }) => ({ role, content }))
                .slice(0, -1),
            });
            return options;
          },
        });
        this.abortFetch = await send({
          question: _question,
          config: this.activeConfig,
        });
      } else {
        this.$refs.modelComp.forEach((comp) => {
          comp.handleSend({ question: _question });
        });
      }
    },
    updateModel(resource) {
      this.url = resource.modelApi;
      this.model = resource;
    },
    addModel() {
      if (this.compareChatIds.length < 4) {
        this.$store.commit("chat/ADD_COMPARE_MODEL");
      }
    },
    updateModelMessages(uuid, messages) {
      this.$store.commit("chat/UPDATE_MODEL_MESSAGES", { uuid, messages });
    },
    updateSystemPrompt(content) {
      this.$store.commit("chat/UPDATE_SYSTEM_PROMPT", {
        content,
        key: this.activeUUID,
      });
    },
    handleAction(uuid, action) {
      switch (action) {
        case "singleModel":
          this.$store.dispatch("chat/SET_SINGLE_MODEL_FROM_COMPARE");
          break;
        case "remove":
          this.$store.dispatch("chat/REMOVE_COMPARE_MODEL", uuid);
          break;
        case "clear":
          this.$store.commit("chat/CLEAR_CHAT_MESSAGES", uuid);
          break;
        default:
          break;
      }
    },
    callbackAbortFetch() {
      if (this.isChatType) {
        this.abortFetch?.abort();
        this.$store.commit("chat/SET_ABORT_FROM_UI");
      } else {
        this.$refs.modelComp.forEach((comp) => {
          comp.handleAbort();
          this.$store.commit("chat/SET_ABORT_FROM_UI", comp.uuid);
        });
      }
    },
    async regenerate(uuid, question = "") {
      const lastQuestion = await this.$store.dispatch(
        "chat/REGENERATE_MESSAGE",
        {
          key: uuid || "single",
          question,
        }
      );
      if (lastQuestion) {
        const { send, loading } = useChat(this.url, {
          messages: this.singleChatMessages,
          onUpdate: (chunk) => {
            this.$store.commit("chat/PUSH_CHAT_COMPLETIONS_CHUNK", { chunk });
          },
          beforeFetch: (options) => {
            const assistantRole = cloneDeep({
              role: "assistant",
              content: "",
              reasoning: "",
            });
            const userRole = cloneDeep({
              role: "user",
              content: options.payload.question,
            });
            const isRegenerate = options?.payload?.isRegenerate;
            this.$store.commit("chat/PUSH_CHAT_COMPLETIONS_CHUNK", {
              chunk: isRegenerate ? assistantRole : [userRole, assistantRole],
              type: "single",
              addRole: true,
            });
            options.body = JSON.stringify({
              ...options.body,
              messages: this.singleChatMessages
                .map(({ role, content }) => ({ role, content }))
                .slice(0, -1),
            });
            return options;
          },
        });
        this.abortFetch = await send({
          question: lastQuestion,
          config: this.activeConfig,
          isRegenerate: true,
        });
      }
    },
  },
  mounted() {
    // const scrollContainer = this.$refs.scrollContainer;
    // const observerTarget = this.$refs.observerTarget;
  },
};
</script>

<script>
import Tab from '@shell/components/Tabbed/Tab';
import Tabbed from '@shell/components/Tabbed';
import { MANAGEMENT } from '@shell/config/types';
import ResourceTable from '@shell/components/ResourceTable';
import Loading from '@shell/components/Loading';
import { SUBTYPE_MAPPING, CREATE_VERBS } from '@shell/models/management.llmos.ai.globalrole';
import { NAME } from '@shell/config/product/auth';
import ButtonLink from '@shell/components/ButtonLink';
const GLOBAL = SUBTYPE_MAPPING.GLOBAL.key;
const NAMESPACE = SUBTYPE_MAPPING.NAMESPACE.key;

const createGlobalRole = {
  name:   `c-cluster-${ NAME }-roles-resource-create`,
  params: {
    cluster:  'local',
    resource: MANAGEMENT.GLOBAL_ROLE,
  }
};

const createRoleTemplate = {
  name:   `c-cluster-${ NAME }-roles-resource-create`,
  params: {
    cluster:  'local',
    resource: MANAGEMENT.ROLE_TEMPLATE,
  }
};

export default {
  name: 'Roles',

  components: {
    ButtonLink,
    Tab,
    Tabbed,
    ResourceTable,
    Loading,
  },

  async fetch() {
    const globalRoleSchema = this.$store.getters[`management/schemaFor`](
      MANAGEMENT.GLOBAL_ROLE
    );
    const roleTemplatesSchema = this.$store.getters[`management/schemaFor`](
      MANAGEMENT.ROLE_TEMPLATE
    );

    this['globalRoles'] = globalRoleSchema ? await this.$store.dispatch(`management/findAll`, { type: MANAGEMENT.GLOBAL_ROLE }) : [];
    this['roleTemplates'] = roleTemplatesSchema ? await this.$store.dispatch(`management/findAll`, { type: MANAGEMENT.ROLE_TEMPLATE }) : [];
  },

  data() {
    const globalRoleSchema = this.$store.getters[`management/schemaFor`](
      MANAGEMENT.GLOBAL_ROLE
    );
    const roleTemplatesSchema = this.$store.getters[`management/schemaFor`](
      MANAGEMENT.ROLE_TEMPLATE
    );

    const roleTemplateHeaders =
      this.$store.getters['type-map/headersFor'](roleTemplatesSchema);
    const defaultHeaderIndex = roleTemplateHeaders.findIndex(
      (header) => header.name === 'default'
    );

    return {
      tabs: {
        [GLOBAL]: {
          canFetch:       globalRoleSchema?.collectionMethods.find((verb) => verb === 'GET'),
          canCreate:      globalRoleSchema?.resourceMethods.find((verb) => CREATE_VERBS.has(verb)),
          labelKey:       SUBTYPE_MAPPING.GLOBAL.labelKey,
          weight:         2,
          schema:         globalRoleSchema,
          createLocation: {
            ...createGlobalRole,
            query: { roleContext: GLOBAL }
          },
        },
        [NAMESPACE]: {
          canFetch:       roleTemplatesSchema?.collectionMethods.find((verb) => verb === 'GET'),
          canCreate:      roleTemplatesSchema?.resourceMethods.find((verb) => CREATE_VERBS.has(verb)),
          labelKey:       SUBTYPE_MAPPING.NAMESPACE.labelKey,
          weight:         1,
          schema:         roleTemplatesSchema,
          headers:        this.applyDefaultHeaderLabel(roleTemplateHeaders, defaultHeaderIndex, 'tableHeaders.authRoles.namespaceDefault'),
          createLocation: {
            ...createRoleTemplate,
            query: { roleContext: NAMESPACE }
          },
        },
      },

      GLOBAL,
      NAMESPACE,

      globalRoles:   null,
      roleTemplates: null,
    };
  },

  computed: {
    globalResources() {
      return this.globalRoles;
    },

    namespaceResources() {
      return this.roleTemplates.filter(
        (r) => r.context === SUBTYPE_MAPPING.NAMESPACE.context
      );
    },

    type() {
      if (this.$route.hash.endsWith(NAMESPACE)) {
        return NAMESPACE;
      }

      return GLOBAL;
    },

    canCreate() {
      return this.tabs[this.type].canCreate;
    },

    createLabel() {
      return this.t(`rbac.roletemplate.subtypes.${ this.type }.createButton`);
    },

    createLocation() {
      return this.tabs[this.type].createLocation;
    },
  },

  methods: {
    applyDefaultHeaderLabel(roleTemplateHeaders, defaultHeaderIndex, labelKey) {
      const headers = [...roleTemplateHeaders];

      headers[defaultHeaderIndex] = {
        ...roleTemplateHeaders[defaultHeaderIndex],
        labelKey,
      };

      return headers;
    },
  },
};
</script>

<template>
  <Loading v-if="!globalRoles" />
  <div v-else>
    <header>
      <div class="title">
        <h1 class="m-0">
          {{ t('rbac.roletemplate.label') }}
        </h1>
      </div>
      <div class="actions-container">
        <div class="actions">
          <button-link
            v-if="canCreate"
            :to="createLocation"
          >
            {{ createLabel }}
          </button-link>
        </div>
      </div>
    </header>
    <Tabbed>
      <Tab
        v-if="tabs[GLOBAL].canFetch"
        :name="GLOBAL"
        :weight="tabs[GLOBAL].weight"
        :label-key="tabs[GLOBAL].labelKey"
      >
        <ResourceTable
          :schema="tabs[GLOBAL].schema"
          :rows="globalResources"
        />
      </Tab>

      <Tab
        v-if="tabs[NAMESPACE].canFetch"
        :name="NAMESPACE"
        :weight="tabs[NAMESPACE].weight"
        :label-key="tabs[NAMESPACE].labelKey"
      >
        <ResourceTable
          :schema="tabs[NAMESPACE].schema"
          :headers="tabs[NAMESPACE].headers"
          :rows="namespaceResources"
        />
      </Tab>
    </Tabbed>
  </div>
</template>

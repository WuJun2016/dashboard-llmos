<script>
import CreateEditView from '@shell/mixins/create-edit-view';
import ResourceTabs from '@shell/components/form/ResourceTabs';
import { COUNT, WORKLOAD_TYPES } from '@shell/config/types';
import { WORKLOAD_SCHEMA } from '@shell/config/schema';
import { getStatesByType } from '@shell/plugins/dashboard-store/resource-class';
import Tab from '@shell/components/Tabbed/Tab';
import ResourceTable from '@shell/components/ResourceTable';
import SortableTable from '@shell/components/SortableTable';
import Loading from '@shell/components/Loading';
import {
  flatten, compact, filter, findKey, values
} from 'lodash';

export default {
  components: {
    ResourceTable,
    ResourceTabs,
    Loading,
    Tab,
    SortableTable
  },

  mixins: [CreateEditView],

  props: {
    mode: {
      default: 'create',
      type:    String,
    },
    value: {
      required: true,
      type:     Object,
    },
  },

  data() {
    const headers = [
      {
        name:  'type',
        label: this.t('tableHeaders.simpleType'),
        value: 'type'
      },
      {
        name:      'error',
        label:     this.t('namespace.resourceStates.error'),
        value:     'byState.error',
        sort:      'byState.error:desc',
        formatter: 'Number'
      },
      {
        name:      'warning',
        label:     this.t('namespace.resourceStates.warning'),
        value:     'byState.warning',
        sort:      'byState.warning:desc',
        formatter: 'Number'
      },
      {
        name:      'transitioning',
        label:     this.t('namespace.resourceStates.info'),
        value:     'byState.info',
        sort:      'byState.info:desc',
        formatter: 'Number'
      },
      {
        name:  'active',
        label: this.t('namespace.resourceStates.success'),
        sort:  'byState.success:desc',
        value: 'byState.success',
      },
      {
        name:      'unknown',
        label:     this.t('namespace.resourceStates.unknown'),
        sort:      'byState.unknown:desc',
        value:     'byState.unknown',
        formatter: 'Number'
      },
      {
        name:  'total',
        label: this.t('namespace.total'),
        sort:  'totalCount:desc',
        value: 'totalCount',
      },
    ];

    return {
      allWorkloads: {
        default: () => ([]),
        type:    Array,
      },
      resourceTypes:  [],
      summaryStates:  ['success', 'info', 'warning', 'error', 'unknown'],
      headers,
      workloadSchema: WORKLOAD_SCHEMA
    };
  },

  async fetch() {
    this.allWorkloads = await this.getAllWorkloads();
  },

  computed: {
    inStore() {
      return this.$store.getters['currentProduct'].inStore;
    },

    namespacedResourceCounts() {
      const allClusterResourceCounts = this.$store.getters[`${ this.inStore }/all`](COUNT)[0].counts;

      return Object.keys(allClusterResourceCounts).reduce((namespaced, type) => {
        const typeCounts = allClusterResourceCounts[type];

        if (!typeCounts.namespaces || !typeCounts.namespaces[this.value.id]) {
          return namespaced;
        }
        const inNamespace = typeCounts.namespaces[this.value.id];

        namespaced.push({
          type,
          byState:    this.reduceStates(inNamespace.states || {}, inNamespace.count),
          totalCount: inNamespace.count,
          schema:     this.schemaFor(type)
        });

        return namespaced;
      }, []);
    },

    statesByType() {
      return getStatesByType();
    },

    /**
     * Workload table data for current namespace
     */
    workloadRows() {
      const params = this.$route.params;
      const { id } = params;
      const rows = flatten(compact(this.allWorkloads)).filter((row) => !row.ownedByWorkload);
      const namespacedRows = filter(rows, ({ metadata: { namespace } }) => namespace === id);

      return namespacedRows;
    }
  },

  methods: {
    // sort states into 5 color-delineated categories: info, error, success, warning, unknown
    reduceStates(states, total) {
      const out = { success: total };

      Object.keys(states).forEach((state) => {
        out.success -= states[state];
        if (!!out[state]) {
          out[state] += states[state];
        } else {
          const genericStateKey = findKey(
            this.statesByType,
            (stateNames) => stateNames.includes(state)
          );

          if (genericStateKey) {
            out[genericStateKey] ? out[genericStateKey] += states[state] : out[genericStateKey] = states[state];
          } else {
            out.unknown ? out.unknown += states[state] : out.unknown = states[state];
          }
        }
      });

      return out;
    },

    /**
     * Retrieve workloads for this namespace and each type
     */
    getAllWorkloads() {
      return Promise.all(values(WORKLOAD_TYPES)
        // You may not have RBAC to see some of the types
        .filter((type) => Boolean(this.schemaFor(type)))
        .map((type) => this.$store.dispatch('cluster/findAll', { type }))
      );
    },

    schemaFor(type) {
      return this.$store.getters[`${ this.inStore }/schemaFor`](type);
    },

    typeListLocation(schema) {
      if (schema?.links?.collection) {
        const location = {
          name:   'c-cluster-product-resource',
          params: { ...this.$route.params, resource: schema.shortId }
        };

        return location;
      }

      return null;
    },
  }
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <div class="mb-20">
      <h3>{{ t('namespace.resources') }}</h3>
    </div>
    <ResourceTabs
      :value="value"
      :mode="mode"
    >
      <Tab :name="t('namespace.resources')">
        <SortableTable
          default-sort-by="error"
          :table-actions="false"
          :row-actions="false"
          :rows="namespacedResourceCounts"
          :headers="headers"
        >
          <template #col:type="{row}">
            <td>
              <router-link
                v-if="typeListLocation(row.schema)"
                :to="typeListLocation(row.schema)"
              >
                {{ row.schema.pluralName }}
              </router-link>
              <span v-else>{{ row.schema.pluralName }}</span>
            </td>
          </template>
        </SortableTable>
      </Tab>

      <Tab :name="t('namespace.workloads')">
        <ResourceTable
          name="workloads"
          :groupable="false"
          v-bind="$attrs"
          :schema="workloadSchema"
          :rows="workloadRows"
        />
      </Tab>
    </ResourceTabs>
  </div>
</template>

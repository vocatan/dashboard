<script>
import Certificate from '@/shared/networking.k8s.io.ingress/Certificate';
import DefaultBackend from '@/shared/networking.k8s.io.ingress/DefaultBackend';
import { allHash } from '@/utils/promise';
import { SECRET, TLS_CERT, WORKLOAD_TYPES } from '@/config/types';
import NameNsDescription from '@/components/form/NameNsDescription';
import CreateEditView from '@/mixins/create-edit-view';
import Tab from '@/components/Tabbed/Tab';
import Footer from '@/components/form/Footer';
import ResourceTabs from '@/components/form/ResourceTabs';
import Rule from './Rule';

export default {
  name: 'CRUIngress',

  components: {
    DefaultBackend,
    NameNsDescription,
    Rule,
    Tab,
    Certificate,
    Footer,
    ResourceTabs
  },

  mixins: [CreateEditView],

  props: {
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },

    mode: {
      type:    String,
      default: 'edit'
    }
  },

  async fetch() {
    const hash = await allHash({
      secrets: this.$store.dispatch('cluster/findAll', { type: SECRET }),
      ...Object.values(WORKLOAD_TYPES).reduce((all, type) => {
        all[type] = this.$store.dispatch('cluster/findAll', { type });

        return all;
      }, {})
    });

    const workloads = Object.values(WORKLOAD_TYPES).map(type => hash[type]);

    const flattened = workloads.reduce((all, type) => {
      all.push(...type);

      return all;
    }, []);

    this.allSecrets = hash.secrets;
    this.allWorkloads = flattened;
  },

  data() {
    return { allSecrets: [], allWorkloads: [] };
  },

  computed: {
    workloads() {
      return this.filterByNamespace(this.filterByOwner(this.allWorkloads)).reduce((all, workload) => {
        all.push(workload?.metadata?.name);

        return all;
      }, []);
    },

    certificates() {
      return this.filterByNamespace(this.allSecrets.filter(secret => secret._type === TLS_CERT)).map((secret) => {
        const { id } = secret;

        return id.slice(id.indexOf('/') + 1);
      });
    },

  },

  created() {
    if (!this.value.spec) {
      this.value.spec = {};
    }

    if (!this.value.spec.rules) {
      this.value.spec.rules = [{}];
    }

    if (!this.value.spec.backend) {
      this.value.spec.backend = { };
    }

    if (!this.value.spec.tls) {
      this.value.spec.tls = [{ }];
    }

    this.registerBeforeHook(this.willSave, 'willSave');
  },

  methods: {
    addRule() {
      this.value.spec.rules = [...this.value.spec.rules, {}];
      this.$forceUpdate();
    },

    removeRule(idx) {
      const neu = [...this.value.spec.rules];

      neu.splice(idx, 1);

      this.$set(this.value.spec, 'rules', neu);
      this.$forceUpdate();
    },

    updateRule(neu, idx) {
      this.$set(this.value.spec.rules, idx, neu);
    },

    addCert() {
      this.value.spec.tls = [...this.value.spec.tls, {}];
      this.$forceUpdate();
    },

    removeCert(idx) {
      const neu = [...this.value.spec.tls];

      neu.splice(idx, 1);
      this.$set(this.value.spec, 'tls', neu);
      this.$$forceUpdate();
    },

    // filter a given list of resources by currently selected namespaces
    filterByNamespace(list) {
      const namespaces = this.$store.getters['namespaces']();

      return list.filter((resource) => {
        return !!namespaces[resource.metadata.namespace];
      });
    },

    // filter by ownerReference id OR filter by lack of owner
    filterByOwner(list, id) {
      return list.filter((resource) => {
        const owners = resource.metadata.ownerReferences;

        if (!id) {
          return !owners;
        }

        const owner = owners[0] || {};

        return (owner.name === id);
      });
    },

    willSave() {
      if (this.value?.spec?.backend && (!this.value?.spec?.backend?.serviceName || !this.value?.spec?.backend?.servicePort)) {
        this.value.spec.backend = null;
      }
    }
  }
};
</script>

<template>
  <form>
    <NameNsDescription :value="value" :mode="mode" />
    <div>
      <h3>
        Rules
      </h3>
      <Rule
        v-for="(rule, i) in value.spec.rules"
        :key="i"
        :value="rule"
        :workloads="workloads"
        @remove="e=>removeRule(i)"
        @input="e=>updateRule(e,i)"
      />
      <button class="btn btn-sm role-primary mt-20 " type="button" @click="addRule">
        Add Rule
      </button>
    </div>
    <div>
      <ResourceTabs v-model="value" :mode="mode">
        <template #before>
          <Tab label="Certificates" name="certificates">
            <Certificate
              v-for="(cert,i) in value.spec.tls"
              :key="i"
              :mode="mode"
              :certs="certificates"
              :value="cert"
              @input="e=>$set(value.spec.tls, i, e)"
              @remove="e=>removeCert(i)"
            />
            <button class="btn btn-sm role-primary mt-20 " type="button" @click="addCert">
              Add Certificate
            </button>
          </Tab>
          <Tab label="Default Backend" name="default-backend">
            <DefaultBackend v-model="value.spec.backend" :targets="workloads" :mode="mode" />
          </Tab>
        </template>
      </ResourceTabs>
    </div>
    <Footer :errors="errors" :mode="mode" @save="save" @done="done" />
  </form>
</template>

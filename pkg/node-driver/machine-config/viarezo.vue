<script>
import Loading from '@shell/components/Loading';
import { Banner } from '@components/Banner';
import CreateEditView from '@shell/mixins/create-edit-view';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { LabeledInput } from '@components/Form/LabeledInput';
import { NORMAN, SECRET } from '@shell/config/types';
import { stringify } from '@shell/utils/error';
import { _VIEW } from '@shell/config/query-params';
import { Openstack } from '../openstack';

function initOptions() {
  return {
    options: [],
    selected: null,
    busy: false,
    enabled: false,
  };
}

export default {
  components: {
    Banner, Loading, LabeledInput, LabeledSelect
  },

  mixins: [CreateEditView],

  props: {
    uuid: {
      type: String,
      required: true,
    },

    cluster: {
      type: Object,
      default: () => ({})
    },

    credentialId: {
      type: String,
      required: true,
    },

    disabled: {
      type: Boolean,
      default: false
    },

    busy: {
      type: Boolean,
      default: false
    },

    provider: {
      type: String,
      required: true,
    }
  },

  async fetch() {
    this.errors = [];
    if (!this.credentialId) {
      return;
    }

    if (this.mode === _VIEW) {
      this.initForViewMode();

      return;
    }

    try {
      this.credential = await this.$store.dispatch('rancher/find', { type: NORMAN.CLOUD_CREDENTIAL, id: this.credentialId });
    } catch (e) {
      this.credential = null;
    }

    // Try and get the secret for the Cloud Credential as we need the plain-text password
    try {
      const id = this.credentialId.replace(':', '/');
      const secret = await this.$store.dispatch('management/find', { type: SECRET, id });
      const data = secret.data['viarezocredentialConfig-password'];
      const password = atob(data);

      this.password = password;
      this.ready = true;
    } catch (e) {
      this.password = '';
      console.error(e); // eslint-disable-line no-console
    }

    this.authenticating = true;

    const os = new Openstack(this.$store, this.credential);

    os.password = this.password;

    this.os = os;

    // Fetch a token - if this succeeds, kick off async fetching the lists we need
    this.os.getToken().then((res) => {
      if (res.error) {
        this.authenticating = false;
        this.$emit('validationChanged', false);

        this.errors.push('Unable to authenticate with the OpenStack server');

        return;
      }

      this.authenticating = false;

      os.getFlavors(this.flavors, this.value?.flavorName);
      os.getImages(this.images, this.value?.imageName);
      os.getSecurityGroups(this.securityGroups, this.value?.secGroups);
      os.getNetworkNames(this.networks, this.value?.netName);
    });

    this.$emit('validationChanged', true);
  },

  data() {
    return {
      authenticating: false,
      ready: false,
      os: null,
      password: null,
      flavors: initOptions(),
      images: initOptions(),
      securityGroups: initOptions(),
      networks: initOptions(),
      sshUser: this.value?.sshUser || 'ubuntu',
      errors: null,
    };
  },

  watch: {
    'credentialId'() {
      this.$fetch();
    },
  },

  methods: {
    stringify,

    initForViewMode() {
      this.fakeSelectOptions(this.flavors, this.value?.flavorId);
      this.fakeSelectOptions(this.images, this.value?.imageId);
      this.fakeSelectOptions(this.securityGroups, this.value?.secGroups);
      this.fakeSelectOptions(this.networks, this.value?.netId);
    },

    fakeSelectOptions(list, value) {
      list.busy = false;
      list.enabled = false;
      list.options = [];

      if (value) {
        list.options.push({
          label: value,
          value,
        });
      }

      list.selected = value;
    },

    syncValue() {
      // Note: We don't need to provide password as this is picked up via the credential

      // Copy the values from the form to the correct places on the value
      this.value.authUrl = this.os.endpoint;
      this.value.domainId = this.os.domainId;
      this.value.username = this.os.username;
      this.value.projectId = this.os.projectId;
      this.value.flavorId = this.flavors.selected?.id;
      this.value.imageId = this.images.selected?.id;
      this.value.netId = this.networks.selected?.id;
      this.value.secGroups = this.securityGroups.selected?.name;
      this.value.sshUser = this.sshUser;
      this.value.region = this.os.region;
      this.value.serverGroupName = this.value.id.split("/")[1];

      // Not configurable
      this.value.endpointType = 'publicURL';
      this.value.availabilityZone = "nova";

      console.log(this.value);
    },

    test() {
      this.syncValue();
    }
  }
};
</script>

<template>
  <div>
    <Loading v-if="$fetchState.pending" :delayed="true" />
    <div v-if="errors.length">
      <div v-for="(err, idx) in errors" :key="idx">
        <Banner color="error" :label="stringify(err)" />
      </div>
    </div>
    <div>
      <div class="openstack-config">
        <div class="title">
          Openstack Configuration
        </div>
        <div v-if="authenticating" class="loading">
          <i class="icon-spinner icon-spin icon-lg" />
          <span>
            Authenticating with the Openstack server ...
          </span>
        </div>
      </div>
      <div class="row mt-10">
        <div class="col span-6">
          <LabeledSelect v-model="flavors.selected" label="Flavor" :options="flavors.options"
            :disabled="!flavors.enabled || busy" :loading="flavors.busy" :searchable="false" />
        </div>

        <div class="col span-6">
          <LabeledSelect v-model="images.selected" label="Image" :options="images.options"
            :disabled="!images.enabled || busy" :loading="images.busy" :searchable="false" />
        </div>
      </div>
      <div class="row mt-10">
        <div class="col span-6">
          <LabeledSelect v-model="securityGroups.selected" label="Security Groups" :options="securityGroups.options"
            :disabled="!securityGroups.enabled || busy" :loading="securityGroups.busy" :searchable="false" />
        </div>
        <div class="col span-6">
          <LabeledSelect v-model="networks.selected" label="Networks" :options="networks.options"
            :disabled="!networks.enabled || busy" :loading="networks.busy" :searchable="false" />
        </div>
      </div>
      <div class="row mt-10">
        <div class="col span-6">
          <LabeledInput v-model="sshUser" :mode="mode" :disabled="busy" label="SSH User ID" placeholder="ubuntu" />
        </div>
      </div>
    </div>
  </div>
</template>
<style scoped lang="scss">
.file-button {
  align-items: center;
  position: absolute;
  top: 0;
  right: 0;
  height: 100%;
  display: flex;

  >.file-selector {
    height: calc($input-height - 2px);
    border-top-left-radius: 0;
    border-bottom-left-radius: 0;
  }
}

.openstack-config {
  display: flex;
  align-items: center;

  >.title {
    font-weight: bold;
    padding: 4px 0;
  }

  >.loading {
    margin-left: 20px;
    display: flex;
    align-items: center;

    >i {
      margin-right: 4px;
      ;
    }
  }
}
</style>

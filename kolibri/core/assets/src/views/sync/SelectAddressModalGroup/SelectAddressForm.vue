<template>

  <KModal
    :title="$attrs.title || $tr('header')"
    :submitText="coreString('continueAction')"
    :cancelText="coreString('cancelAction')"
    size="medium"
    :submitDisabled="submitDisabled"
    @submit="handleSubmit"
    @cancel="$emit('cancel')"
  >
    <template>
      <p v-if="initialFetchingComplete && !addresses.length">
        {{ $tr('noAddressText') }}
      </p>
      <UiAlert
        v-if="uiAlertProps"
        v-show="showUiAlerts"
        :type="uiAlertProps.type"
        :dismissible="false"
      >
        {{ uiAlertProps.text }}
        <KButton
          v-if="requestsFailed"
          appearance="basic-link"
          :text="$tr('refreshAddressesButtonLabel')"
          @click="refreshSavedAddressList"
        />
      </UiAlert>

      <KButton
        v-show="!newAddressButtonDisabled"
        class="new-address-button"
        :text="$tr('newAddressButtonLabel')"
        appearance="basic-link"
        @click="$emit('click_add_address')"
      />

      <template v-for="(a, idx) in savedAddresses">
        <div :key="`div-${idx}`">
          <KRadioButton
            :key="idx"
            v-model="selectedAddressId"
            class="radio-button"
            :value="a.id"
            :label="a.nickname"
            :description="a.base_url"
            :disabled="!a.available || !a.hasContent"
          />
          <KButton
            v-if="!hideSavedAddresses"
            :key="`forget-${idx}`"
            :text="$tr('forgetAddressButtonLabel')"
            appearance="basic-link"
            @click="removeSavedAddress(a.id)"
          />
        </div>
      </template>

      <hr v-if="!hideSavedAddresses && discoveredAddresses.length > 0">

      <template v-for="d in discoveredAddresses">
        <div :key="`div-${d.id}`">
          <KRadioButton
            :key="d.id"
            v-model="selectedAddressId"
            class="radio-button"
            :value="d.instance_id"
            :label="formatNameAndId(d.device_name, d.id)"
            :description="d.base_url"
            :disabled="!d.available || discoveryFailed"
          />
        </div>
      </template>
    </template>

    <slot name="underbuttons"></slot>

    <KFixedGrid slot="actions" class="actions" numCols="4">
      <KFixedGridItem span="1">
        <transition name="spinner-fade">
          <div v-if="discoveringPeers">
            <KLabeledIcon>
              <KCircularLoader slot="icon" :size="16" :stroke="6" class="loader" />
            </KLabeledIcon>
          </div>
        </transition>
      </KFixedGridItem>
      <KFixedGridItem span="3" alignment="right">
        <KButtonGroup style="margin-top: 8px;">
          <KButton
            :text="coreString('cancelAction')"
            appearance="flat-button"
            @click="$emit('cancel')"
          />
          <KButton
            :text="coreString('continueAction')"
            :primary="true"
            :disabled="submitDisabled"
            type="submit"
          />
        </KButtonGroup>
      </KFixedGridItem>
    </KFixedGrid>

  </KModal>

</template>


<script>

  import find from 'lodash/find';
  import UiAlert from 'kolibri-design-system/lib/keen/UiAlert';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import commonSyncElements from 'kolibri.coreVue.mixins.commonSyncElements';
  import { deleteAddress, fetchStaticAddresses, fetchDynamicAddresses } from './api';

  const Stages = {
    FETCHING_ADDRESSES: 'FETCHING_ADDRESSES',
    FETCHING_SUCCESSFUL: 'FETCHING_SUCCESSFUL',
    FETCHING_FAILED: 'FETCHING_FAILED',
    DELETING_ADDRESS: 'DELETING_ADDRESS',
    DELETING_SUCCESSFUL: 'DELETING_SUCCESSFUL',
    DELETING_FAILED: 'DELETING_FAILED',
    PEER_DISCOVERY_STARTED: 'PEER_DISCOVERY_STARTED',
    PEER_DISCOVERY_SUCCESSFUL: 'PEER_DISCOVERY_SUCCESSFUL',
    PEER_DISCOVERY_FAILED: 'PEER_DISCOVERY_FAILED',
  };

  export default {
    name: 'SelectAddressForm',
    components: {
      UiAlert,
    },
    mixins: [commonCoreStrings, commonSyncElements],
    props: {
      discoverySpinnerTime: { type: Number, default: 2500 },
      // Arg that's passed to fetchDynamic/StaticAddresses
      fetchAddressArgs: {
        type: String,
        required: false,
        default: '',
      },
      // Hides "New address" button and other saved locations
      hideSavedAddresses: {
        type: Boolean,
        default: false,
      },
      // If an ID is provided, that address's radio button will be automatically selected
      selectedId: {
        type: String,
        required: false,
      },
    },
    data() {
      return {
        savedAddresses: [],
        savedAddressesInitiallyFetched: false,
        discoveredAddresses: [],
        discoveredAddressesInitiallyFetched: true,
        selectedAddressId: '',
        showUiAlerts: false,
        stage: '',
        discoveryStage: '',
        Stages,
      };
    },
    computed: {
      addresses() {
        return this.savedAddresses.concat(this.discoveredAddresses);
      },
      initialFetchingComplete() {
        return this.savedAddressesInitiallyFetched && this.discoveredAddressesInitiallyFetched;
      },
      submitDisabled() {
        return (
          this.selectedAddressId === '' ||
          this.stage === this.Stages.FETCHING_ADDRESSES ||
          this.stage === this.Stages.DELETING_ADDRESS ||
          this.discoveryStage === this.Stages.PEER_DISCOVERY_FAILED
        );
      },
      newAddressButtonDisabled() {
        return this.hideSavedAddresses || this.stage === this.Stages.FETCHING_ADDRESSES;
      },
      requestsFailed() {
        return (
          this.stage === this.Stages.FETCHING_FAILED || this.stage === this.Stages.DELETING_FAILED
        );
      },
      discoveringPeers() {
        return this.discoveryStage === this.Stages.PEER_DISCOVERY_STARTED;
      },
      discoveryFailed() {
        return this.discoveryStage === this.Stages.PEER_DISCOVERY_FAILED;
      },
      uiAlertProps() {
        if (this.stage === this.Stages.FETCHING_FAILED) {
          return {
            text: this.$tr('fetchingFailedText'),
            type: 'error',
          };
        }
        if (this.discoveryStage === this.Stages.PEER_DISCOVERY_FAILED) {
          return {
            text: this.$tr('fetchingFailedText'),
            type: 'error',
          };
        }
        if (this.stage === this.Stages.DELETING_FAILED) {
          return {
            text: this.$tr('deletingFailedText'),
            type: 'error',
          };
        }
        return null;
      },
    },
    beforeMount() {
      this.startDiscoveryPolling();
      return this.refreshSavedAddressList();
    },
    mounted() {
      // Wait a little bit of time before showing UI alerts so there is no flash
      // if data comes back quickly
      setTimeout(() => {
        this.showUiAlerts = true;
      }, 100);
    },
    destroyed() {
      this.stopDiscoveryPolling();
    },
    methods: {
      refreshSavedAddressList() {
        this.stage = this.Stages.FETCHING_ADDRESSES;
        this.savedAddresses = [];
        return fetchStaticAddresses(this.fetchAddressArgs)
          .then(addresses => {
            this.savedAddresses = addresses;
            this.resetSelectedAddress();
            this.stage = this.Stages.FETCHING_SUCCESSFUL;
            this.savedAddressesInitiallyFetched = true;
            if (this.savedAddresses.find(({ id }) => this.selectedId === id)) {
              this.selectedAddressId = this.selectedId;
            }
          })
          .catch(() => {
            this.stage = this.Stages.FETCHING_FAILED;
          });
      },
      resetSelectedAddress() {
        const availableAddress = find(this.addresses, { available: true });
        if (availableAddress) {
          this.selectedAddressId = availableAddress.id;
        } else {
          this.selectedAddressId = '';
        }
      },
      removeSavedAddress(id) {
        this.stage = this.Stages.DELETING_ADDRESS;
        return deleteAddress(id)
          .then(() => {
            this.savedAddresses = this.savedAddresses.filter(a => a.id !== id);
            this.resetSelectedAddress(this.savedAddresses);
            this.stage = this.Stages.DELETING_SUCCESSFUL;
            this.$emit('removed_address');
          })
          .catch(() => {
            this.stage = this.Stages.DELETING_FAILED;
          });
      },

      discoverPeers() {
        this.$parent.$emit('started_peer_discovery');
        this.discoveryStage = this.Stages.PEER_DISCOVERY_STARTED;
        return fetchDynamicAddresses(this.fetchAddressArgs)
          .then(devices => {
            this.discoveredAddresses = devices;
            this.$parent.$emit('finished_peer_discovery');
            setTimeout(() => {
              this.discoveryStage = this.Stages.PEER_DISCOVERY_SUCCESSFUL;
            }, this.discoverySpinnerTime);
            this.discoveredAddressesInitiallyFetched = true;
          })
          .catch(() => {
            this.$parent.$emit('peer_discovery_failed');
            this.discoveryStage = this.Stages.PEER_DISCOVERY_FAILED;
          });
      },

      startDiscoveryPolling() {
        this.discoverPeers();
        if (!this.intervalId) {
          this.intervalId = setInterval(this.discoverPeers, 5000);
        }
      },

      stopDiscoveryPolling() {
        if (this.intervalId) {
          this.intervalId = clearInterval(this.intervalId);
        }
      },

      handleSubmit() {
        if (this.selectedAddressId) {
          const match = find(this.addresses, { id: this.selectedAddressId });
          match.isDynamic = Boolean(find(this.discoveredAddresses, { id: this.selectedAddressId }));
          this.$emit('submit', match);
        }
      },
    },
    $trs: {
      deletingFailedText: 'There was a problem removing this address',
      fetchingFailedText: 'There was a problem getting the available addresses',
      forgetAddressButtonLabel: 'Forget',
      header: 'Select network address',
      newAddressButtonLabel: 'Add new address',
      noAddressText: 'There are no addresses yet',
      refreshAddressesButtonLabel: 'Refresh addresses',
    },
  };

</script>


<style lang="scss" scoped>

  .new-address-button {
    margin-bottom: 16px;
  }

  .radio-button {
    display: inline-block;
    width: 75%;
  }

  .spinner-fade-leave-active,
  .spinner-fade-enter-active {
    transition: opacity 0.5s;
  }

  .spinnner-fade-enter-to,
  .spinner-fade-leave {
    opacity: 1;
  }

  .spinner-fade-enter,
  .spinner-fade-leave-to {
    opacity: 0;
  }

  .ui-progress-circular {
    display: inline-block;
    margin-right: 2px;
    margin-bottom: 2px;
    vertical-align: middle;
  }

  hr {
    border: 0;
    border-bottom: 1px solid #cbcbcb;
  }

  .loader {
    position: relative;
    top: 12px;
  }

</style>

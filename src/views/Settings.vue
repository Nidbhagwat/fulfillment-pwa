<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-menu-button slot="start" />
        <ion-title>{{ $t("Store Settings") }}</ion-title>
      </ion-toolbar>
    </ion-header>
    
    <ion-content>
      <ion-item>
        <ion-label>{{ $t("Store") }}</ion-label>
        <ion-select interface="popover" :value="currentFacility.facilityId" @ionChange="setFacility($event)">
          <ion-select-option v-for="facility in (userProfile ? userProfile.facilities : [])" :key="facility.facilityId" :value="facility.facilityId" >{{ facility.name }}</ion-select-option>
        </ion-select>
      </ion-item>

      <ion-item>
        <ion-icon :icon="globeOutline" slot="start" />
        <ion-label>{{$t("eCom Store")}}</ion-label>
        <ion-select interface="popover" :value="currentEComStore.productStoreId" @ionChange="setEComStore($event)">
          <ion-select-option v-for="store in (userProfile ? userProfile.stores : [])" :key="store.productStoreId" :value="store.productStoreId" >{{ store.storeName }}</ion-select-option>
        </ion-select>
      </ion-item>

      <ion-item>
        <ion-icon :icon="codeWorkingOutline" slot="start"/>
        <ion-label>{{ $t("OMS") }}</ion-label>
        <p slot="end">{{ baseURL ? baseURL : instanceUrl }}</p>
      </ion-item>

      <!-- TODO: Add functionality to make document printing functionality global -->
      <ion-card>
        <ion-item lines="none">
          <ion-label class="text-wrap">{{ $t("Documents to print when packing orders") }}</ion-label>
        </ion-item>
        <ion-item>
          <ion-label>{{ $t("Shipping label") }}</ion-label>
          <ion-checkbox :checked="userPreference.printShippingLabel" @ionChange="setPrintShippingLabelPreference($event)" slot="end" />
        </ion-item>
        <ion-item lines="none">
          <ion-label>{{ $t("Packing slip") }}</ion-label>
          <ion-checkbox :checked="userPreference.printPackingSlip" @ionChange="setPrintPackingSlipPreference($event)" slot="end" />
        </ion-item>
      </ion-card>

      <ion-card>
        <ion-item lines="none">
          <ion-label>{{ $t("Fulfillment") }} : {{ isStoreFulfillmentTurnOn ? $t("On") : $t("Off") }}</ion-label>
        </ion-item>
        <ion-item lines="none">
          <ion-label class="text-wrap">{{ $t("has outstanding orders and in progress orders.", {storeName: currentFacility.name, outstandingOrdersCount, inProgressOrdersCount}) }}</ion-label>
        </ion-item>
        <div class="actions">
          <div>
            <ion-button :disabled="!hasPermission(Actions.APP_RECYCLE_ORDER)" fill="outline" color="secondary" size="medium" @click="recycleOutstandingOrders()">{{ $t("Recycle all open orders") }}</ion-button>
            <ion-button :disabled="!hasPermission(Actions.APP_RECYCLE_ORDER)" fill="outline" color="secondary" size="medium" @click="recycleInProgressOrders()">{{ $t("Recycle all in progress orders") }}</ion-button>
            <!-- <ion-button fill="outline" color="secondary" size="medium">{{ $t("Recycle all orders") }}</ion-button> -->
          </div>
          <div>
            <ion-button :disabled="!hasPermission(Actions.APP_TURN_OFF_STORE)" v-if="isStoreFulfillmentTurnOn" fill="outline" color="danger" size="medium" @click="turnOffFulfillment()">{{ $t("Turn off fulfillment") }}</ion-button>
            <ion-button v-else fill="outline" color="success" size="medium" @click="turnOnFulfillment()">{{ $t("Turn on fulfillment") }}</ion-button>
          </div>
        </div>

        <ion-item class="mobile-only">
          <ion-button fill="clear">{{ $t("Recycle all open orders") }}</ion-button>
          <ion-button slot="end" fill="clear" color="medium" @click="openRecyclePopover">
            <ion-icon :icon="ellipsisVerticalOutline" slot="icon-only" />
          </ion-button>
        </ion-item>
      </ion-card>

      <ion-item>
        <ion-icon :icon="timeOutline" slot="start"/>                
        <ion-label> {{ userProfile && userProfile.userTimeZone ? userProfile.userTimeZone : '-' }} </ion-label>
        <ion-button @click="changeTimeZone()" fill="outline" color="dark">{{ $t("Change") }}</ion-button>
      </ion-item>
      <ion-item>
        <ion-label>{{ userProfile !== null ? userProfile.partyName : '' }}</ion-label>
        <ion-button fill="outline" color="medium" @click="logout()">{{ $t("Logout") }}</ion-button>
      </ion-item>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { 
  IonButton, 
  IonCard, 
  IonCheckbox, 
  IonContent, 
  IonHeader,
  IonIcon, 
  IonItem, 
  IonLabel, 
  IonMenuButton,
  IonPage, 
  IonSelect, 
  IonSelectOption, 
  IonTitle, 
  IonToolbar,
  popoverController,
  modalController,
alertController} from '@ionic/vue';
import { defineComponent } from 'vue';
import { codeWorkingOutline, ellipsisVerticalOutline, globeOutline, timeOutline } from 'ionicons/icons'
import RecyclePopover from '@/views/RecyclePopover.vue'
import { mapGetters, useStore } from 'vuex';
import { useRouter } from 'vue-router';
import TimeZoneModal from '@/views/timezone-modal.vue'
import { UserService } from '@/services/UserService';
import { showToast } from '@/utils';
import { hasError } from '@/adapter';
import { translate } from '@/i18n';
import logger from '@/logger';
import { Actions, hasPermission } from '@/authorization'

export default defineComponent({
  name: 'Settings',
  components: { 
    IonButton,
    IonCard,
    IonCheckbox,
    IonContent, 
    IonHeader, 
    IonIcon,
    IonItem, 
    IonLabel, 
    IonMenuButton,
    IonPage, 
    IonSelect, 
    IonSelectOption,
    IonTitle, 
    IonToolbar
  },
  data() {
    return {
      baseURL: process.env.VUE_APP_BASE_URL,
      currentFacilityDetails: {} as any,
      outstandingOrdersCount: 0,
      inProgressOrdersCount: 0
    };
  },
  computed: {
    ...mapGetters({
      userProfile: 'user/getUserProfile',
      currentFacility: 'user/getCurrentFacility',
      instanceUrl: 'user/getInstanceUrl',
      currentEComStore: 'user/getCurrentEComStore',
      userPreference: 'user/getUserPreference'
    }),
    isStoreFulfillmentTurnOn() {
      // considered that if facility details are not available then also fulfillment will be turned on
      return this.currentFacilityDetails.maximumOrderLimit != 0
    }
  },
  ionViewWillEnter() {
    this.getCurrentFacilityDetails()
    this.getOutstandingOrdersCount()
    this.getInProgressOrdersCount()
  },
  methods: {
    async getOutstandingOrdersCount() {
      let resp;

      try {
        this.outstandingOrdersCount = 0

        const payload = {
          "json": {
            "params": {
              "rows": "0",
              "group": true,
              "group.field": "orderId",
              "group.limit": 10000,
              "group.ngroups": true,
              "q.op": "AND"
            },
            "query": "(*:*)",
            "filter": ["docType: OISGIR", "quantityNotAvailable: 0", "isPicked: N", "-shipmentMethodTypeId: STOREPICKUP", "-fulfillmentStatus: Cancelled", "orderStatusId: ORDER_APPROVED", "orderTypeId: SALES_ORDER", `facilityId: ${this.currentFacility.facilityId}`, `productStoreId: ${this.currentEComStore.productStoreId}`]
          }
        }

        resp = await UserService.getOutstandingOrdersCount(payload)

        if(!hasError(resp)) {
          this.outstandingOrdersCount = resp.data.grouped.orderId.ngroups
        } else {
          throw resp.data
        }
      } catch(err) {
        logger.error('Failed to get outstanding orders count', err)
      }
    },
    async getInProgressOrdersCount() {
      let resp;

      try {
        this.inProgressOrdersCount = 0

        const payload = {
          "json": {
            "params": {
              "rows": "0",
              "group": true,
              "group.field": "picklistBinId",
              "group.limit": 10000,
              "group.ngroups": true,
              "q.op": "AND"
            },
            "query": "(*:*)",
            "filter": ["docType: OISGIR", "picklistItemStatusId: PICKITEM_PENDING", "-shipmentMethodTypeId: STOREPICKUP", "-fulfillmentStatus: Rejected", `facilityId: ${this.currentFacility.facilityId}`, `productStoreId: ${this.currentEComStore.productStoreId}`]
          }
        }

        resp = await UserService.getInProgressOrdersCount(payload)

        if(!hasError(resp)) {
          this.inProgressOrdersCount = resp.data.grouped.picklistBinId.ngroups
        } else {
          throw resp.data
        }
      } catch(err) {
        logger.error('Failed to get inProgress orders count', err)
      }
    },
    async getCurrentFacilityDetails() {
      let resp: any;
      try {
        this.currentFacilityDetails = {}

        resp = await UserService.getFacilityDetails({
          "entityName": "Facility",
          "inputFields": {
            "facilityId": this.currentFacility.facilityId
          },
          "viewSize": 1,
          "fieldList": ["maximumOrderLimit", "facilityId"]
        })

        if(!hasError(resp) && resp.data.count) {
          // using index 0 as we will only get a single record
          this.currentFacilityDetails = resp.data.docs[0]
        } else {
          throw resp.data
        }
      } catch(err) {
        logger.error('Failed to fetch current facility details', err)
      }
    },
    async openRecyclePopover(ev: Event) {
      const popover = await popoverController.create({
        component: RecyclePopover,
        event: ev,
        translucent: true,
        showBackdrop: false,
      });
      return popover.present();
    },
    logout () {
      this.store.dispatch('user/logout').then(() => {
        this.store.dispatch('order/clearOrders')
        this.router.push('/login');
      })
    },
    async setFacility (event: any) {
      // not updating the facility when the current facility in vuex state and the selected facility are same
      // or when an empty value is given (on logout)
      if (this.currentFacility.facilityId === event.detail.value || !event.detail.value) {
        return;
      }

      if (this.userProfile){
        await this.store.dispatch('user/setFacility', {
          'facility': this.userProfile.facilities.find((fac: any) => fac.facilityId == event.detail.value)
        });
        this.store.dispatch('order/clearOrders')
        this.getCurrentFacilityDetails();
        this.getOutstandingOrdersCount();
        this.getInProgressOrdersCount();
      }
    },
    async changeTimeZone() {
      const timeZoneModal = await modalController.create({
        component: TimeZoneModal,
      });
      return timeZoneModal.present();
    },
    async updateFacility(maximumOrderLimit: number) {
      let resp;

      try {
        resp = await UserService.updateFacility({
          "facilityId": this.currentFacility.facilityId,
          maximumOrderLimit
        })

        if(!hasError(resp)) {
          this.currentFacilityDetails.maximumOrderLimit = maximumOrderLimit
          showToast(translate('Facility updated successfully'))
        } else {
          throw resp.data
        }
      } catch(err) {
        showToast(translate('Failed to update facility'))
        logger.error('Failed to update facility', err)
      }
    },
    async turnOnFulfillment() {
      const alert = await alertController.create({
        header: this.$t('Turn on fulfillment for ', { facilityName: this.currentFacility.name }),
        buttons: [{
          text: translate('Cancel'),
          role: 'cancel'
        }, {
          text: translate('Save'),
          handler: (data) => {
            // Adding this extra check as min attribute does not work when providing input using keyboard
            if (data.setLimit <= 0) {
              showToast(translate('Provide a value greater than 0'))
              return;
            }
            this.updateFacility(data.setLimit)
          }
        }],
        inputs: [{
          name: 'setLimit',
          min: 1,
          type: 'number',
          placeholder: translate('Set Limit')
        }],
      });

      await alert.present();
    },
    async turnOffFulfillment() {
      const alert = await alertController.create({
        header: this.$t('Turn off fulfillment for ', { facilityName: this.currentFacility.name }),
        message: translate('Are you sure you want perform this action?'),
        buttons: [{
          text: translate('Cancel'),
          role: 'cancel'
        }, {
          text: translate('Save'),
          handler: () => {
            this.updateFacility(0);
          }
        }]
      });

      await alert.present();
    },
    async recycleOutstandingOrders() {
      const alert = await alertController.create({
        header: translate('Recycle outstanding orders'),
        message: this.$t('Are you sure you want to recycle outstanding order(s)?', { ordersCount: this.outstandingOrdersCount }),
        buttons: [{
          text: translate('No'),
          role: 'cancel'
        }, {
          text: translate('Yes'),
          handler: async () => {
            let resp;

            try {
              resp = await UserService.recycleOutstandingOrders({
                "facilityId": this.currentFacility.facilityId,
                "productStoreId": this.currentEComStore.productStoreId,
                "reasonId": "INACTIVE_STORE"
              })

              if(!hasError(resp)) {
                showToast(translate('Recycling has been started. All outstanding orders will be recycled shortly.'))
              } else {
                throw resp.data
              }
            } catch(err) {
              showToast(translate('Failed to recycle outstanding orders'))
              logger.error('Failed to recycle outstanding orders', err)
            }
          }
        }]
      });

      await alert.present();
    },
    async recycleInProgressOrders() {
      const alert = await alertController.create({
        header: translate('Recycle in progress orders'),
        message: this.$t('Are you sure you want to recycle in progress order(s)?', { ordersCount: this.inProgressOrdersCount }),
        buttons: [{
          text: translate('No'),
          role: 'cancel'
        }, {
          text: translate('Yes'),
          handler: async () => {
            let resp;

            try {
              resp = await UserService.recycleInProgressOrders({
                "facilityId": this.currentFacility.facilityId,
                "productStoreId": this.currentEComStore.productStoreId,
                "reasonId": "INACTIVE_STORE"
              })

              if(!hasError(resp)) {
                showToast(translate('Recycling has been started. All in progress orders will be recycled shortly.'))
              } else {
                throw resp.data
              }
            } catch(err) {
              showToast(translate('Failed to recycle in progress orders'))
              logger.error('Failed to recycle in progress orders', err)
            }
          }
        }]
      });

      await alert.present();
    },
    async setEComStore(event: any) {
      // not updating the ecomstore when the current value in vuex state and selected value are same
      // or when an empty value is given (on logout)
      if (this.currentEComStore.productStoreId === event.detail.value || !event.detail.value) {
        return;
      }

      if(this.userProfile) {
        await this.store.dispatch('user/setEComStore', {
          'eComStore': this.userProfile.stores.find((str: any) => str.productStoreId == event.detail.value)
        })
        this.getOutstandingOrdersCount();
        this.getInProgressOrdersCount();
      }
    },
    setPrintShippingLabelPreference (ev: any) {
      this.store.dispatch('user/setUserPreference', { printShippingLabel: ev.detail.checked })
    },
    setPrintPackingSlipPreference (ev: any){
      this.store.dispatch('user/setUserPreference', { printPackingSlip: ev.detail.checked })
    }
  },
  setup() {
    const store = useStore();
    const router = useRouter();

    return {
      Actions,
      codeWorkingOutline,
      ellipsisVerticalOutline,
      globeOutline,
      timeOutline,
      router,
      store,
      hasPermission
    }
  }
});
</script>

<style scoped>
.text-wrap {
  white-space: normal;
}
</style>
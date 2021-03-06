<template>
  <div
    id="app"
    class="grid-y medium-grid-frame"
  >
    <PhilaModal
      v-show="isModalOpen"
      @close="closeModal"
    >
      <div slot="body">
        <div class="step-group">
          <div class="step-label">
            1
          </div>
          <div class="step">
            <div class="step-title">
              Step 1
            </div>
            <div class="step-content" />
          </div>
          <div class="step-label">
            2
          </div>
          <div class="step">
            <div class="step-title">
              Step 2
            </div>
            <div class="step-content" />
          </div>
          <div class="step-label">
            3
          </div>
          <div class="step">
            <div class="step-title">
              Step 3
            </div>
            <div class="step-content" />
          </div>
        </div>
      </div>
    </PhilaModal>
    <PhilaHeader
      :app-title="this.$config.app.title"
      :app-tag-line="this.$config.app.tagLine"
    >
      <div slot="mobile-menu">
        <PhilaFooter
          @howToUseLink="toggleModal()"
        />
      </div>
      <!-- <RefinePanel slot="after-stripe" /> -->
    </PhilaHeader>

    <div class="cell medium-auto medium-cell-block-container main-content">
      <div class="grid-x">
        <LocationsPanel
          v-show="isMapVisible || isLarge"
        />
        <MapPanel
          v-show="!isMapVisible || isLarge"
        >
          <cyclomedia-widget
            v-if="this.shouldLoadCyclomediaWidget"
            v-show="cyclomediaActive"
            slot="cycloWidget"
            screen-percent="2"
          />
        </MapPanel>
      </div>
    </div>

    <PhilaButton
      class="button toggle-map hide-for-medium"
      @click.native="toggleMap"
    >
      {{ buttonText }}
    </PhilaButton>
    <PhilaFooter
      v-show="isLarge"
      @howToUseLink="toggleModal()"
    />
  </div>
</template>
<script>

import { point } from '@turf/helpers';
import buffer from '@turf/buffer';
import booleanPointInPolygon from '@turf/boolean-point-in-polygon';

import PhilaButton from './components/PhilaButton.vue';
import PhilaHeader from './components/PhilaHeader.vue';
import PhilaFooter from './components/PhilaFooter.vue';
import PhilaModal from './components/PhilaModal.vue';
// import RefinePanel from './components/RefinePanel.vue';
import LocationsPanel from './components/LocationsPanel.vue';
import MapPanel from './components/MapPanel.vue';

export default {
  name: 'App',
  components: {
    PhilaButton,
    PhilaHeader,
    PhilaFooter,
    PhilaModal,
    // RefinePanel,
    LocationsPanel,
    MapPanel,
    CyclomediaWidget: () => import(/* webpackChunkName: "mbmb_pvm_CyclomediaWidget" */'@philly/vue-mapping/src/cyclomedia/Widget.vue'),
  },
  data() {
    return {
      isMapVisible: true,
      isModalOpen: false,
      isLarge: true,
      buffer: null,
      buttonText: 'Toggle map',
    };
  },
  computed: {
    geocodeStatus() {
      return this.$store.state.geocode.status;
    },
    geocodeResult() {
      return this.$store.state.geocode.data || {};
    },
    geocodeGeom() {
      return this.geocodeResult.geometry;
    },
    bufferList() {
      return this.$store.state.bufferList;
    },
    selectedKeywords() {
      return this.$store.state.selectedKeywords;
    },
    selectedServices() {
      return this.$store.state.selectedServices;
    },
    dataStatus() {
      return this.$store.state.pinSources[this.$appType].status;
    },
    database() {
      return this.$store.state.pinSources[this.$appType].data;
    },
    shouldLoadCyclomediaWidget() {
      return this.$config.cyclomedia.enabled && !this.isMobileOrTablet;
    },
    cyclomediaActive() {
      return this.$store.state.cyclomedia.active;
    },
    cycloLatlng() {
      if (this.$store.state.cyclomedia.orientation.xyz !== null) {
        const xyz = this.$store.state.cyclomedia.orientation.xyz;
        return [ xyz[1], xyz[0] ];
      } 
      const center = this.$config.map.center;
      return center;
      
    },
    cycloRotationAngle() {
      return this.$store.state.cyclomedia.orientation.yaw * (180/3.14159265359);
    },
    cycloHFov() {
      return this.$store.state.cyclomedia.orientation.hFov;
    },
    selectedResources() {
      return this.$store.state.selectedResources;
    },
  },
  watch: {
    geocodeStatus(nextGeocodeStatus) {
      if (nextGeocodeStatus === 'success') {
        this.runBuffer();
      } else if (nextGeocodeStatus === null) {
        this.$data.buffer = null;
      }
    },
    selectedResources(nextSelectedResources) {
      console.log('watch selectedResources is firing');
      this.getSelectedData(nextSelectedResources);
    },
    // buffer() {
    //   this.filterPoints();
    // },
    // selectedServices() {
    //   if (this.$store.state.pinSources[this.$appType].data) {
    //     this.filterPoints();
    //   }
    // },
    // selectedKeywords() {
    //   if (this.$store.state.pinSources[this.$appType].data) {
    //     this.filterPoints();
    //   }
    // },
    dataStatus(nextDataStatus) {
      if (nextDataStatus === 'success') {
        this.filterPoints();
      }
    },
  },
  mounted() {
    console.log('in App.vue mounted, this.$config:', this.$config);
    if (this.$config.pinSources) {
      this.$controller.dataManager.fetchData('pins');
    }
    this.$controller.appDidLoad();
    this.onResize();
  },
  created() {
    window.addEventListener('resize', this.onResize);
  },

  beforeDestroy() {
    window.removeEventListener('resize', this.onResize);
  },
  methods: {
    init() {
      console.log('in App.vue mounted, this.$config:', this.$config);
      if (this.$config.pinSources) {
        this.$controller.dataManager.fetchData('pins');
      }
      this.onResize();
    },
    runBuffer() {
      const geocodePoint = point(this.geocodeGeom.coordinates);
      const pointBuffer = buffer(geocodePoint, 1, { units: 'miles' });
      this.$data.buffer = pointBuffer;
    },
    filterPoints() {
      console.log('App.vue filterPoints is running');
      const filteredRows = this.database.slice(0,100);
      // const filteredRows = [];
      //
      // for (const row of this.database) {
      //     let booleanServices;
      //     const servicesSplit = row.services_offered.split(',');
      //     const { selectedServices } = this.$store.state;
      //     if (selectedServices.length === 0) {
      //       booleanServices = true;
      //     } else {
      //       const servicesFiltered = servicesSplit.filter(f => selectedServices.includes(f));
      //       booleanServices = servicesFiltered.length > 0;
      //     }
      //
      //     let booleanBuffer = false;
      //     if (!this.$data.buffer) {
      //       booleanBuffer = true;
      //     } else if (typeof row.lon === 'number' && row.lon !== null) {
      //       const rowPoint = point([ row['y-lat'], row['x-lat'] ]);
      //       if (booleanPointInPolygon(rowPoint, this.$data.buffer)) {
      //         booleanBuffer = true;
      //       }
      //     }
      //
      //     let booleanKeywords = true;
      //     if (this.selectedKeywords.length > 0) {
      //       booleanKeywords = false;
      //       const description = row.description.split(/,| /);
      //       const keywordsFiltered = this.selectedKeywords.filter(f => description.includes(f));
      //       if (keywordsFiltered.length > 0) {
      //         booleanKeywords = true;
      //       }
      //     }
      //
      //     if (booleanServices && booleanBuffer && booleanKeywords) {
      //       filteredRows.push(row);
      //     }
      //   }
      this.$store.commit('setCurrentData', filteredRows);
    },
    toggleMap() {
      if (window.innerWidth > 749) {
        this.$data.isMapVisible = true;
      } else {
        this.$data.isMapVisible = !this.$data.isMapVisible;
      }
      this.$data.buttonText = this.$data.isMapVisible ? 'Toggle map' : 'Toggle resource list';
    },
    toggleModal() {
      this.isModalOpen = !this.isModalOpen;
      this.toggleBodyClass('no-scroll');
    },
    showModal() {
      this.isModalOpen = true;
      this.toggleBodyClass('no-scroll');
    },
    closeModal() {
      this.isModalOpen = false;
      this.toggleBodyClass('no-scroll');
    },
    toggleBodyClass(className) {
      const el = document.body;
      return this.isOpen ? el.classList.add(className) : el.classList.remove(className);
    },
    onResize() {
      if (window.innerWidth > 749) {
        this.$data.isMapVisible = true;
        this.$data.isLarge = true;
      } else {
        this.$data.isLarge = false;
      }
    },
    getSelectedData(selectedResources) {
      console.log('getSelectedData is running, selectedResources:', selectedResources);
      let currentData = this.$store.state.currentData;
      let currentSelectedData;
      currentSelectedData = currentData.filter(cd => selectedResources.includes(cd._featureId));
      this.$store.commit('setCurrentSelectedData', currentSelectedData);
      this.$controller.handleSearchFormSubmit(currentSelectedData[0].StreetAddress, 'Address');
    },
  },
};
</script>

<style lang="scss">
@import "@/scss/global.scss";

.toggle-map{
  margin:0 !important;
}
.main-content{
  margin-top:.5rem;
}

//TODO, move to standards
@each $value in $colors {
  //sass-lint:disable-block no-important
  .#{nth($value, 1)} {
    color: nth($value, 2) !important;
  }
  .bg-#{nth($value, 1)} {
    background-color: nth($value, 2) !important;
  }
  .bdr-#{nth($value, 1)} {
    border-color: nth($value, 2) !important;
  }
}
@media screen and (max-width: 749px) {
  .main-content{
    margin-top:9rem;
    margin-bottom:2rem;
  }
}
.no-scroll{
  overflow: hidden;
  height: 100vh;
}
.toggle-map{
  position: fixed;
  bottom:0;
  width: 100%;
  z-index: 1002;
}

.step-group{
  margin-left:$spacing-medium;

  .step-label {
    @include secondary-font(400);
    display: inline-block;
    margin-left: -$spacing-medium;
    background: black;
    border-radius: $spacing-extra-large;
    color:white;
    padding: 0 $spacing-small;
    width:$spacing-large;
    height:$spacing-large;
    line-height: $spacing-large;
    text-align: center;
  }
  .step{
    margin-top: -$spacing-large;
    padding-left: $spacing-large;
    padding-bottom: $spacing-large;
    border-left:1px solid black;

    &:last-of-type {
      border:none;
    }

    .step-title{
      @include secondary-font(400);
      font-size:1.2rem;
      margin-bottom: $spacing-small;
    }
  }
}
</style>

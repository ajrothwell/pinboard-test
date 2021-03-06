<template>
  <div
    class="cell medium-cell-block-container bg-ghost-gray refine-panel"
    :class="{ 'refine-open': refineOpen }"
  >
    <div class="grid-x">
      <fieldset class="cell">
        <div
          class="refine-title"
          @click="expandRefine"
        >
          <legend class="legend-title h3">
            {{ legendTitle }}
          </legend>
          <PhilaButton
            class="hide-for-small-only"
            @click.native="clearAll"
          >
            Clear all
          </PhilaButton>
        </div>
        <div
          v-if="dataStatus === 'success'"
          class="grid-x service-list"
        >
          <div
            v-for="(item, index) in getRefineSearchList()"
            :key="index"
            class="cell medium-6"
          >
            <label :for="item">
              <input
                :id="item"
                v-model="selected"
                type="checkbox"
                :name="item"
                :value="item"
              >
              <span class="service-item">{{ item }}</span>
            </label>
          </div>
        </div>
        <div class="mobile-filter-actions show-for-small-only">
          <PhilaButton
            @click.native="expandRefine(); scrollToTop();"
          >
            <font-awesome-icon icon="filter" /> Apply filters
          </PhilaButton>
          <PhilaButton
            @click.native="closeRefinePanel"
          >
            Clear all
          </PhilaButton>
        </div>
      </fieldset>
    </div>
  </div>
</template>
<script>

import { mapState } from 'vuex';
import PhilaButton from './PhilaButton.vue';

export default {
  components: {
    PhilaButton,
  },
  props: {
    legendTitle: {
      type: String,
      default: 'Refine',
    },
  },
  data() {
    return {
      baseUrl: process.env.VUE_APP_BASE_URL,
      refineList: null,
      selected: [],
      refineOpen: false,
      // addressEntered: null,
    };
  },
  computed: {
    ...mapState([ 'sources', 'geocode', 'selectedServices' ]),
    addressEntered() {
      let address;

      if (this.geocode.status === 'success') {
        address = this.geocode.data.properties.street_address;
      }

      return address;
    },
    keywordsEntered() {
      return this.$store.state.selectedKeywords.toString();
    },
    dataStatus() {
      return this.$store.state.sources[this.$appType].status;
    },
  },
  watch: {
    selected(nextSelected) {
      console.log('RefinePanel watch selected is firing, nextSelected', nextSelected);
      this.$store.commit('setSelectedServices', nextSelected);
      this.$controller.handleRefinePanelClick(nextSelected);
    },
    selectedServices(nextSelectedServices) {
      // console.log('watch selectedServices is firing:', nextSelectedServices);
      this.$data.selected = nextSelectedServices;
    },
  },
  mounted() {
    console.log('RefinePanel is mounted, this.$store.state.selectedServices:', this.$store.state.selectedServices);
    if (this.$store.state.selectedServices.length > 0) {
      console.log('there are services');
    }
    // this.$data.selected = this.$store.state.selectedServices;
  },
  methods: {
    clearAll() {
      console.log('RefinePanel clearAll is running');
      if (this.selected.length) {
        this.selected = [];
      }
    },
    getRefineSearchList() {
      const refineData = this.sources[this.$appType].data.rows;

      let service = '';

      refineData.forEach((arrayElem) => {
        service += `${arrayElem.services_offered},`;
      });

      // TODO: break this into smaller chunks
      let serviceArray = service.split(/(,|;)/);
      serviceArray = serviceArray.map(s => s.trim());

      const uniqArray = [ ...new Set(serviceArray) ];

      // clean up any dangling , or ;
      const uniq = uniqArray.filter(a => a.length > 2);

      uniq.filter(Boolean); // remove empties
      uniq.sort();
      return uniq;
    },
    scrollToTop() {
      const container = document.querySelector('.refine-panel');
      container.scrollTo(0, 0);
    },
    expandRefine() {
      if (window.innerWidth <= 749) { // converted from rems
        this.refineOpen = !this.refineOpen;
      }
    },
    closeRefinePanel(){
      this.scrollToTop();
      this.expandRefine();
      this.clearAll();
    },
  },
};
</script>
<style lang="scss">

$refine-panel-height: 19vh;
.refine-panel{
  max-height: $refine-panel-height;
  overflow-y: hidden;
  padding: 1rem;

  .refine-title{
    color: color(dark-ben-franklin);
    position: relative;

    .clear-all{
      margin: 0 0 0 4rem;
    }

    .clear-button{
      background-color: rgb(0, 204, 255);
      height: 30px;
      width: 80px;
      margin-left: 10px;
      margin-right: 10px;
    }
  }

  .margin-add{
    margin-left: 10px;
    margin-right: 10px;
    display: inline-block;
  }

  .min-width-needed{
    min-width: 140px;
    min-height: 30px;
    border-style: solid;
    border-width: 1px;
  }

  legend.legend-title, .clear-all{
    float: left;
  }

  .service-list{
    input{
      margin-right: -2rem;
    }
    .service-item {
      margin-left: 3rem;
      display: inline-block;
      line-height: 1.3rem;
      vertical-align: text-top;
    }
  }
  label {
    font-weight: normal;
    cursor: pointer;
  }
  @media screen and (max-width: 749px) {
    height: 3rem;
    padding: .5rem;
    position: relative;
    z-index: 10000;
    .refine-title{
      cursor: pointer;
      height:7vh;
      padding: .5rem;

      &::after{
        content: '+';
        font-weight: 900;
        position: absolute;
        font-size: 1.2rem;
        right: 0;
        top: 0;
      }
    }
  }
  &.refine-open{
    overflow-y: scroll;
    height: calc(100vh - 100px);
    max-height: 100vh;
    z-index: 1002;
    .refine-title{
      &::after{
        content: '-';
      }
    }
  }
}

</style>

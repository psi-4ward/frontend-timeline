<template>
  <header :class="$mq === 'sm' ? 'header-sm' : null">
    <div v-if="$mq === 'sm'" class="header-item">
      <button
        class="button button-flat button-icon"
        @click="showMobileNav = !showMobileNav"
      >
        <MenuIcon size="1x" aria-hidden="true" role="img" />
      </button>
    </div>
    <nav
      v-if="$mq === 'sm' ? showMobileNav : true"
      class="header-item header-item-grow"
      :class="$mq === 'sm' ? 'nav-sm' : null"
    >
      <div class="nav-item">
        <CrosshairIcon
          v-if="$mq === 'sm'"
          size="1x"
          aria-hidden="true"
          role="img"
        />
        <button
          class="button button-outline"
          :title="
            $t('Automatically center the map view and zoom in to relevant data')
          "
          @click="$root.$emit('fitView')"
        >
          {{ $t("Fit view") }}
        </button>
      </div>
      <div class="nav-item">
        <LayersIcon size="1x" aria-hidden="true" role="img" />
        <DropdownButton
          :label="$t('Layer settings')"
          :title="$t('Show/hide layers')"
        >
          <label v-for="option in layerSettingsOptions" :key="option.layer">
            <input
              type="checkbox"
              :checked="map.layers[option.layer]"
              @change="
                setMapLayerVisibility({
                  layer: option.layer,
                  visibility: $event.target.checked,
                })
              "
            />
            {{ option.label }}
          </label>
        </DropdownButton>
      </div>
      <div class="nav-item">
        <CalendarIcon size="1x" aria-hidden="true" role="img" />
        <VueCtkDateTimePicker
          v-model="startDateTime"
          :format="DATE_TIME_FORMAT"
          :color="$config.primaryColor"
          :locale="$config.locale"
          :max-date="endDateTime"
          :button-now-translation="$t('Now')"
        >
          <button
            type="button"
            class="dropdown-button button"
            :title="$t('Select start date')"
          />
        </VueCtkDateTimePicker>
        <span>{{ $t("to") }}</span>
        <VueCtkDateTimePicker
          v-model="endDateTime"
          :format="DATE_TIME_FORMAT"
          :color="$config.primaryColor"
          :locale="$config.locale"
          :min-date="startDateTime"
          :button-now-translation="$t('Now')"
        >
          <button
            type="button"
            class="dropdown-button button"
            :title="$t('Select end date')"
          />
        </VueCtkDateTimePicker>
      </div>
      <div class="nav-item">
        <UserIcon size="1x" aria-hidden="true" role="img" />
        <select
          v-model="selectedUser"
          class="dropdown-button button"
          :title="$t('Select user')"
        >
          <option :value="null">
            {{ $t("Show all") }}
          </option>
          <option v-for="user in users" :value="user" :key="user">
            {{ user }}
          </option>
        </select>
      </div>
      <div v-if="selectedUser" class="nav-item">
        <SmartphoneIcon size="1x" aria-hidden="true" role="img" />
        <select
          v-model="selectedDevice"
          class="dropdown-button button"
          :title="$t('Select device')"
        >
          <option :value="null">
            {{ $t("Show all") }}
          </option>
          <option
            v-for="device in devices[selectedUser]"
            :value="device"
            :key="`${selectedUser}-${device}`"
          >
            {{ device }}
          </option>
        </select>
      </div>
      <div class="nav-item">
        <button
          class="button button-outline"
          :title="$t('Toggle the timeline view')"
          @click="$root.$emit('toggle-timeline')"
        >
          {{ $t("Timeline") }}
        </button>
      </div>
    </nav>
    <nav class="header-item header-item-right">
      <div
        v-if="$config.showDistanceTravelled && distanceTravelled"
        class="nav-item distance-travelled"
      >
        <span :title="$t('Distance travelled')">
          {{ humanReadableDistance(distanceTravelled) }}
        </span>
        <br />
        <span :title="$t('Elevation gain / loss')">
          <ArrowUpIcon size="0.8x" role="img" />
          {{ humanReadableDistance(elevationGain) }}
          /
          <ArrowDownIcon size="0.8x" role="img" />
          {{ humanReadableDistance(elevationLoss) }}
        </span>
      </div>
      <div class="nav-item">
        <button
          class="button button-flat button-icon"
          :title="$t('Download raw data')"
          @click="$modal.show('download')"
        >
          <DownloadIcon
            size="1x"
            :aria-label="$t('Download raw data')"
            role="img"
          />
        </button>
      </div>
      <div class="nav-item">
        <button
          class="button button-flat button-icon"
          :title="$t('Information')"
          @click="$modal.show('information')"
        >
          <InfoIcon size="1x" :aria-label="$t('Information')" role="img" />
        </button>
      </div>
    </nav>
  </header>
</template>

<style lang="scss" scoped>
.distance-travelled {
  text-align: right;
  line-height: 1.2;

  .feather {
    margin-top: 3px;
    margin-right: 0 !important;
  }
}
</style>

<script>
import moment from "moment";
import { mapActions, mapGetters, mapMutations, mapState } from "vuex";
import {
  ArrowDownIcon,
  ArrowUpIcon,
  CalendarIcon,
  CrosshairIcon,
  DownloadIcon,
  InfoIcon,
  LayersIcon,
  MenuIcon,
  SmartphoneIcon,
  UserIcon,
} from "vue-feather-icons";
import VueCtkDateTimePicker from "vue-ctk-date-time-picker";
import "vue-ctk-date-time-picker/dist/vue-ctk-date-time-picker.css";

import DropdownButton from "@/components/DropdownButton";
import { DATE_TIME_FORMAT } from "@/constants";
import * as types from "@/store/mutation-types";
import { humanReadableDistance } from "@/util";

export default {
  components: {
    ArrowDownIcon,
    ArrowUpIcon,
    CalendarIcon,
    CrosshairIcon,
    DownloadIcon,
    InfoIcon,
    LayersIcon,
    MenuIcon,
    SmartphoneIcon,
    UserIcon,
    VueCtkDateTimePicker,
    DropdownButton,
  },
  data() {
    return {
      DATE_TIME_FORMAT,
      layerSettingsOptions: [
        { layer: "last", label: this.$t("Show last known locations") },
        { layer: "line", label: this.$t("Show location history (line)") },
        { layer: "points", label: this.$t("Show location history (points)") },
        { layer: "heatmap", label: this.$t("Show location heatmap") },
        { layer: "stayPoints", label: this.$t("Show stay points") },
      ],
      showMobileNav: false,
    };
  },
  computed: {
    ...mapState([
      "users",
      "devices",
      "map",
      "distanceTravelled",
      "elevationGain",
      "elevationLoss",
    ]),
    selectedUser: {
      get() {
        return this.$store.state.selectedUser;
      },
      set(value) {
        this.setSelectedUser(value);
      },
    },
    selectedDevice: {
      get() {
        return this.$store.state.selectedDevice;
      },
      set(value) {
        this.setSelectedDevice(value);
      },
    },
    startDateTime: {
      get() {
        return moment
          .utc(this.$store.state.startDateTime, DATE_TIME_FORMAT)
          .local()
          .format(DATE_TIME_FORMAT);
      },
      set(value) {
        this.setStartDateTime(
          moment(value, DATE_TIME_FORMAT).utc().format(DATE_TIME_FORMAT)
        );
      },
    },
    endDateTime: {
      get() {
        return moment
          .utc(this.$store.state.endDateTime, DATE_TIME_FORMAT)
          .local()
          .format(DATE_TIME_FORMAT);
      },
      set(value) {
        this.setEndDateTime(
          moment(value, DATE_TIME_FORMAT)
            .set("seconds", 59)
            .utc()
            .format(DATE_TIME_FORMAT)
        );
      },
    },
  },
  methods: {
    ...mapMutations({
      setMapLayerVisibility: types.SET_MAP_LAYER_VISIBILITY,
    }),
    ...mapActions([
      "setSelectedUser",
      "setSelectedDevice",
      "setStartDateTime",
      "setEndDateTime",
    ]),
    humanReadableDistance,
  },
};
</script>

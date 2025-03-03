<template>
  <div class="gated-content-video-page">
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <div v-else-if="error">Error loading</div>
    <template v-else>
      <div class="video-wrapper">
        <div class="video gated-containerV2 px--20-10 pt-40-20">
          <MediaPlayer
            :media="video.attributes.field_gc_video_media"
            :autoplay="config.components.gc_video.autoplay_videos"
            @playerEvent="logPlaybackEvent($event)"
          />
        </div>
      </div>
      <div class="video-footer-wrapper bg-black">
        <div class="video-footer gated-containerV2 text-white px--20-10 py-40-20">
          <div class="pb-20-10 cachet-book-32-28">{{ video.attributes.title }}</div>
          <div class="video-footer__fav pb-40-20">
            <AddToFavorite
              :id="video.attributes.drupal_internal__nid"
              :type="'node'"
              :bundle="'gc_video'"
              class="rounded-border border-thunder white"
            ></AddToFavorite>
            <div class="timer" :style="{ visibility: this.video
              .attributes.field_gc_video_duration ? 'visible': 'hidden'}">
              {{ video_length }}
            </div>
          </div>
          <div class="verdana-14-12">
            <div class="video-footer__block">
              <SvgIcon icon="date-icon" class="fill-white" :growByHeight=false></SvgIcon>
              {{ date }}
            </div>
            <div
              v-if="video.attributes.field_gc_video_instructor"
              class="video-footer__block">
              <SvgIcon icon="instructor-icon"
                       class="fill-white"
                       :growByHeight=false></SvgIcon>
              {{ video.attributes.field_gc_video_instructor }}
            </div>
            <div
              class="video-footer__block"
              v-if="video.attributes.field_gc_video_level"
            >
              <SvgIcon icon="difficulty-icon-white" :css-fill="false"></SvgIcon>
              {{ video.attributes.field_gc_video_level.name | capitalize }}
            </div>
            <div v-if="videoCategories" class="video-footer__block video-footer__category">
              <SvgIcon icon="categories"
                       class="fill-white"
                       :growByHeight=false></SvgIcon>
              <ul>
                <li
                  v-for="tid in videoCategories"
                  class="video-footer__category-list-item"
                  :key="tid"
                >
                  <CategoryLinks :tid="tid" />
                </li>
              </ul>
            </div>
            <div
              v-if="video.attributes.field_gc_video_equipment.length > 0"
              class="video-footer__block">
              <SvgIcon icon="cubes-solid" class="fill-white" :growByHeight=false></SvgIcon>
              Equipment:
            </div>
            <ul class="video-footer__equipment">
              <li v-for="equip in video.attributes.field_gc_video_equipment"
                  :key="equip.drupal_internal__tid">
                {{ equip.name }}
              </li>
            </ul>
          </div>
          <div
            v-if="video.attributes.field_gc_video_description"
            class="verdana-16-14"
            v-html="video.attributes.field_gc_video_description.processed"
          ></div>
        </div>
      </div>
      <VideoListing
        v-if="videoCategories"
        :title="config.components.gc_video.up_next_title"
        :excluded-video-id="video.id"
        :categories="videoCategories"
        :viewAll="true"
        :limit="8"
      />
    </template>
  </div>
</template>

<script>
import dayjs from 'dayjs';
import client from '@/client';
import AddToFavorite from '@/components/AddToFavorite.vue';
import Spinner from '@/components/Spinner.vue';
import VideoListing from '@/components/video/VideoListing.vue';
import MediaPlayer from '@/components/MediaPlayer.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { SettingsMixin } from '@/mixins/SettingsMixin';
import SvgIcon from '@/components/SvgIcon.vue';
import CategoryLinks from '@/components/category/CategoryLinks.vue';

export default {
  name: 'VideoPage',
  mixins: [JsonApiCombineMixin, SettingsMixin],
  components: {
    SvgIcon,
    MediaPlayer,
    VideoListing,
    Spinner,
    AddToFavorite,
    CategoryLinks,
  },
  props: {
    id: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      loading: true,
      error: false,
      video: null,
      response: null,
      params: [
        'field_gc_video_category',
        'field_gc_video_media',
        'field_gc_video_equipment',
        'field_gc_video_level',
      ],
    };
  },
  watch: {
    $route: 'load',
  },
  async mounted() {
    await this.load();
  },
  computed: {
    videoCategories() {
      const fieldValues = this.video.attributes.field_gc_video_category;
      if (!fieldValues || fieldValues.length === 0) {
        return null;
      }
      return fieldValues.map((category) => category.drupal_internal__tid);
    },
    date() {
      return this.$dayjs.date(this.video.attributes.created).format('dddd, MMMM Do, YYYY');
    },
    video_length() {
      const sec = this.video.attributes.field_gc_video_duration;
      if (sec > 0 && sec < 60) {
        return `${sec} ${this.$options.filters.simplePluralize('second', sec)}`;
      }

      const min = Math.floor(dayjs.duration(sec, 'seconds').asMinutes());
      return `${min} ${this.$options.filters.simplePluralize('minute', min)}`;
    },
  },
  methods: {
    async load() {
      this.loading = true;
      const params = {};
      if (this.params) {
        params.include = this.params.join(',');
      }
      client
        .get(`jsonapi/node/gc_video/${this.id}`, { params })
        .then((response) => {
          this.video = this.combine(response.data.data, response.data.included, this.params);
          this.loading = false;
        }).then(() => {
          this.logPlaybackEvent('entityView');
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
          throw error;
        });
    },
    logPlaybackEvent(eventType) {
      this.$log.trackEvent(eventType, 'node', 'gc_video', this.video.attributes.drupal_internal__nid);
    },
  },
};
</script>

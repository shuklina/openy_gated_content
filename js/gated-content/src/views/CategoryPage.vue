<template>
  <div class="gated-content-category-page">
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <div v-else-if="error">Error loading</div>
    <template v-else>
      <Modal v-if="showModal" @close="showModal = false" class="adjust-modal">
        <template v-slot:header>
          <h3>Filter</h3>
        </template>
        <template v-slot:body>
          <div class="filter">
            <h4>Content types</h4>
            <div class="form-check" v-for="option in contentTypeOptions" v-bind:key="option.value">
              <label :for="option.value">
                <input
                  type="radio"
                  :id="option.value"
                  :value="option.value"
                  :disabled="option.value !== 'all' && !showComponent[option.value]"
                  autocomplete="off"
                  v-model="preSelectedComponent"
                >
                <span class="checkmark"></span>
                <span class="caption">{{ option.label }}</span>
              </label>
            </div>
          </div>
          <div class="sort">
            <h4>Sort order</h4>
            <div class="form-check" v-for="option in filterOptions" v-bind:key="option.value">
              <label :for="option.value">
                <input
                  type="radio"
                  :id="option.value"
                  :value="option.value"
                  autocomplete="off"
                  v-model="preSelectedSort"
                >
                <span class="checkmark"></span>
                <span class="caption">{{ option.label }}</span>
              </label>
            </div>
          </div>
        </template>
        <template v-slot:footer>
          <button type="button" class="btn btn-outline-primary" @click="showModal = false">
            Cancel
          </button>
          <button type="button" class="btn btn-primary" @click="applyFilters">Apply</button>
        </template>
      </Modal>

      <div class="gated-containerV2 my-40-20 px--20-10 title-wrapper">
        <div>
          <ul v-if="isCategoriesLoaded" class="title-inline cachet-book-32-28">
            <li
              v-for="categoryData in getAncestors"
              :key="categoryData.tid"
            >
              <router-link
                v-if="categoryData.uuid !== id"
                :to="{
                  name: 'Category',
                  params: { id: categoryData.uuid }
                }" class="text-thunder">{{ categoryData.label }}</router-link>
              <span v-else class="text-gray">
                {{ categoryData.label }}
              </span>
            </li>
          </ul>
          <AddToFavorite
            :id="category.attributes.drupal_internal__tid"
            :type="'taxonomy_term'"
            :bundle="'gc_category'"
          ></AddToFavorite>
        </div>
        <button type="button"
                class="adjust-button" @click="showModal = true">Filter</button>
      </div>

      <CategoriesListing
        :title="'Subcategories'"
        :parent="category.attributes.drupal_internal__tid"
        :bundle="selectedBundle"
        :sort="sortData('taxonomy_term')"
        :limit="50"
      />

      <div v-for="component in componentsOrder" :key="component">
        <div class="live-stream-wrapper" v-if="showComponent.live_stream
          && showOnCurrentIteration('live_stream', component)">
          <EventListing
            v-if="selectedComponent === 'live_stream' || selectedComponent === 'all'"
            :title="config.components.live_stream.title"
            :categories="[category.attributes.drupal_internal__tid]"
            :sort="sortData('eventinstance', 'live_stream')"
            :pagination="selectedComponent === 'live_stream'"
            :limit="viewAllContentMode ? 50 : itemsLimit"
            @listing-not-empty="listingIsNotEmpty('live_stream', ...arguments)"
          >
            <template #filterButton>
              <button
                v-if="selectedComponent === 'all'"
                type="button"
                class="view-all"
                @click="preSelectedComponent = 'live_stream'; applyFilters()">
                More
              </button>
            </template>
          </EventListing>
        </div>

        <div class="virtual-meeting-wrapper" v-if="showComponent.virtual_meeting
          && showOnCurrentIteration('virtual_meeting', component)">
          <EventListing
            v-if="selectedComponent === 'virtual_meeting' || selectedComponent === 'all'"
            :title="config.components.virtual_meeting.title"
            :categories="[category.attributes.drupal_internal__tid]"
            :eventType="'virtual_meeting'"
            :sort="sortData('eventinstance', 'virtual_meeting')"
            :pagination="selectedComponent === 'virtual_meeting'"
            :limit="viewAllContentMode ? 50 : itemsLimit"
            @listing-not-empty="listingIsNotEmpty('virtual_meeting', ...arguments)"
          >
            <template #filterButton>
              <button
                v-if="selectedComponent === 'all'"
                type="button"
                class="view-all"
                @click="preSelectedComponent = 'virtual_meeting'; applyFilters()">
                More
              </button>
            </template>
          </EventListing>
        </div>

        <div class="videos-wrapper" v-if="showComponent.gc_video
          && showOnCurrentIteration('gc_video', component)">
          <VideoListing
            v-if="selectedComponent === 'gc_video' || selectedComponent === 'all'"
            :title="config.components.gc_video.title"
            :categories="[category.attributes.drupal_internal__tid]"
            :pagination="selectedComponent === 'gc_video'"
            :viewAll="false"
            :sort="sortData('node', 'gc_video')"
            :limit="itemsLimit"
            @listing-not-empty="listingIsNotEmpty('gc_video', ...arguments)"
          >
            <template #filterButton>
              <button
                v-if="selectedComponent === 'all'"
                type="button"
                class="view-all"
                @click="preSelectedComponent = 'gc_video'; applyFilters()">
                More
              </button>
            </template>
          </VideoListing>
        </div>

        <div class="blogs-wrapper" v-if="showComponent.vy_blog_post
          && showOnCurrentIteration('vy_blog_post', component)">
          <BlogListing
            v-if="selectedComponent === 'vy_blog_post' || selectedComponent === 'all'"
            :title="config.components.vy_blog_post.title"
            :categories="[category.attributes.drupal_internal__tid]"
            :viewAll="false"
            :sort="sortData('node', 'vy_blog_post')"
            :pagination="selectedComponent === 'vy_blog_post'"
            :limit="itemsLimit"
            @listing-not-empty="listingIsNotEmpty('vy_blog_post', ...arguments)"
            class="my-40-20"
          >
            <template #filterButton>
              <button
                v-if="selectedComponent === 'all'"
                type="button"
                class="view-all"
                @click="preSelectedComponent = 'vy_blog_post'; applyFilters()">
                More
              </button>
            </template>
          </BlogListing>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import client from '@/client';
import Spinner from '@/components/Spinner.vue';
import AddToFavorite from '@/components/AddToFavorite.vue';
import CategoriesListing from '@/components/category/CategoriesListing.vue';
import VideoListing from '@/components/video/VideoListing.vue';
import BlogListing from '@/components/blog/BlogListing.vue';
import EventListing from '@/components/event/EventListing.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { FilterAndSortMixin } from '@/mixins/FilterAndSortMixin';
import { SettingsMixin } from '@/mixins/SettingsMixin';

export default {
  name: 'CategoryPage',
  mixins: [JsonApiCombineMixin, SettingsMixin, FilterAndSortMixin],
  components: {
    AddToFavorite,
    Spinner,
    CategoriesListing,
    VideoListing,
    BlogListing,
    EventListing,
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
      category: null,
      itemsLimit: 8,
      DEFAULT_SORT: 'date_asc',
      showComponent: {
        gc_video: true,
        vy_blog_post: true,
        live_stream: true,
        virtual_meeting: true,
      },
      showLiveStreamViewAll: false,
      showVirtualMeetingViewAll: false,
    };
  },
  watch: {
    id: 'reload',
    '$route.query': 'load',
  },
  async mounted() {
    await this.load();
  },
  methods: {
    async load() {
      this.loading = true;
      client
        .get(`jsonapi/taxonomy_term/gc_category/${this.id}`)
        .then((response) => {
          this.category = response.data.data;
          this.loading = false;
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
          throw error;
        });
    },
    listingIsNotEmpty(component, notEmpty) {
      this.showComponent[component] = notEmpty;
    },
    reload() {
      this.showComponent = {
        gc_video: true,
        vy_blog_post: true,
        live_stream: true,
        virtual_meeting: true,
      };
      this.load();
    },
  },
  computed: {
    ...mapGetters([
      'isCategoriesLoaded',
    ]),
    viewAllContentMode() {
      // Enable viewAllContentMode only when we filter by content.
      return this.selectedComponent !== 'all';
    },
    getAncestors() {
      return this.$store.getters.getAncestors(this.category.attributes.drupal_internal__tid);
    },
  },
};
</script>

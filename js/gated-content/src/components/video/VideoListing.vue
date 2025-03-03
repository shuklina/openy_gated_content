<template>
  <div class="gated-containerV2 my-40-20 px--20-10">
    <div class="listing-header">
      <h2 class="title text-gray" v-if="title !== 'none'">{{ title }}</h2>
      <router-link
        :to="{ name: 'VideoListing', query: { type: categories ? categories[0] : 'all' } }"
        v-if="viewAll && listingIsNotEmpty"
        class="view-all"
      >
        More
      </router-link>
      <slot name="filterButton"></slot>
    </div>
    <div v-if="loading" class="text-center">
      <Spinner></Spinner>
    </div>
    <template v-else-if="listingIsNotEmpty">
      <div v-if="error">Error loading</div>
      <div v-else :class="layoutClass">
        <VideoTeaser
          v-for="video in listing"
          :key="video.id"
          :video="video"
        />
      </div>
    </template>
    <div v-else class="empty-listing">
      {{ emptyBlockMsg }}
    </div>
    <Pagination
      v-if="pagination"
      :links="links"
    ></Pagination>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import client from '@/client';
import VideoTeaser from '@/components/video/VideoTeaser.vue';
import Spinner from '@/components/Spinner.vue';
import Pagination from '@/components/Pagination.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { FavoritesMixin } from '@/mixins/FavoritesMixin';
import { ListingMixin } from '@/mixins/ListingMixin';

export default {
  name: 'VideoListing',
  mixins: [JsonApiCombineMixin, FavoritesMixin, ListingMixin],
  components: {
    VideoTeaser,
    Spinner,
    Pagination,
  },
  props: {
    title: {
      type: String,
      default: 'Videos',
    },
    excludedVideoId: {
      type: String,
      default: '',
    },
    msg: {
      type: String,
      default: 'No videos found.',
    },
    categories: {
      type: Array,
      default: null,
    },
    featured: {
      type: Boolean,
      default: false,
    },
    viewAll: {
      type: Boolean,
      default: false,
    },
    sort: {
      type: Object,
      default() {
        return { path: 'created', direction: 'DESC' };
      },
    },
    limit: {
      type: Number,
      default: 0,
    },
    pagination: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      component: 'gc_video',
      loading: true,
      error: false,
      links: {},
      featuredLocal: false,
      params: [
        'field_gc_video_media',
        'field_gc_video_media.thumbnail',
        'field_gc_video_level',
        'field_gc_video_category',
        'field_gc_video_image',
        'field_gc_video_image.field_media_image',
      ],
    };
  },
  watch: {
    $route: 'load',
    excludedVideoId: 'load',
    sort: 'load',
    isCategoriesLoaded() {
      if (this.categories !== null) {
        this.load();
      }
    },
  },
  computed: {
    ...mapGetters([
      'isCategoriesLoaded',
    ]),
  },
  async mounted() {
    // By default emit that listing not empty to the parent component.
    this.$emit('listing-not-empty', true);
    this.featuredLocal = this.featured;
    await this.load();
  },
  methods: {
    async load() {
      this.loading = true;
      const params = {};
      if (this.params) {
        params.include = this.params.join(',');
      }

      params.sort = {
        sortBy: this.sort,
      };

      params.filter = {};
      if (this.excludedVideoId.length > 0) {
        params.filter.excludeSelf = {
          condition: {
            path: 'id',
            operator: '<>',
            value: this.excludedVideoId,
          },
        };
      }

      if (this.favorites) {
        if (this.isFavoritesTypeEmpty('node', 'gc_video')) {
          this.loading = false;
          return;
        }
        params.filter.includeFavorites = {
          condition: {
            path: 'drupal_internal__nid',
            operator: 'IN',
            value: this.getFavoritesTypeIds('node', 'gc_video'),
          },
        };
      }

      if (this.featuredLocal) {
        params.filter.field_gc_video_featured = 1;
      }
      params.filter.status = 1;
      if (this.pagination) {
        const currentPage = parseInt(this.$route.query.page, 10) || 0;
        params.page = {
          limit: this.config.pager_limit,
          offset: currentPage * this.config.pager_limit,
        };
      } else if (this.limit !== 0) {
        params.page = {
          limit: this.limit,
        };
      }

      if (this.categories !== null) {
        if (!this.isCategoriesLoaded) {
          return;
        }
        const termsIds = [];
        this.categories.forEach((tid) => {
          const subcategories = this.$store.getters.getSubcategories(tid);
          termsIds.push(tid, ...this.$store.getters.getNestedTids(subcategories));
        });
        params.filter.in = {
          condition: {
            path: 'field_gc_video_category.entity.tid',
            operator: 'IN',
            value: termsIds,
          },
        };
      }
      this.loadFromJsonApi(params);
    },
    loadFromJsonApi(params) {
      client
        .get('jsonapi/node/gc_video', { params })
        .then((response) => {
          this.links = response.data.links;
          this.listing = this.combineMultiple(
            response.data.data,
            response.data.included,
            this.params,
          );
          if (this.featuredLocal === true && this.listing.length === 0) {
            // Load one more time without featured filter.
            this.featuredLocal = false;
            this.load();
          }
          if (!this.listingIsNotEmpty) {
            // Emit that listing empty to the parent component.
            this.$emit('listing-not-empty', false);
          }
          this.loading = false;
        })
        .catch((error) => {
          this.error = true;
          this.loading = false;
          console.error(error);
          throw error;
        });
    },
  },
};
</script>

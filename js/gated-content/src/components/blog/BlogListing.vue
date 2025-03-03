<template>
  <div class="gated-containerV2 px--20-10">
    <div class="listing-header">
      <h2 class="title text-gray" v-if="title !== 'none'">{{ title }}</h2>
      <router-link
        :to="{ name: 'BlogsListing', query: { type: categories ? categories[0] : 'all' } }"
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
        <BlogTeaser
          v-for="blog in listing"
          :key="blog.id"
          :blog="blog"
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
import BlogTeaser from '@/components/blog/BlogTeaser.vue';
import Spinner from '@/components/Spinner.vue';
import Pagination from '@/components/Pagination.vue';
import { JsonApiCombineMixin } from '@/mixins/JsonApiCombineMixin';
import { FavoritesMixin } from '@/mixins/FavoritesMixin';
import { ListingMixin } from '@/mixins/ListingMixin';

export default {
  name: 'BlogListing',
  mixins: [JsonApiCombineMixin, FavoritesMixin, ListingMixin],
  components: {
    BlogTeaser,
    Spinner,
    Pagination,
  },
  props: {
    title: {
      type: String,
      default: 'Blog Posts',
    },
    excludedId: {
      type: String,
      default: '',
    },
    msg: {
      type: String,
      default: 'No blog posts found.',
    },
    viewAll: {
      type: Boolean,
      default: false,
    },
    featured: {
      type: Boolean,
      default: false,
    },
    sort: {
      type: Object,
      default() {
        return { path: 'created', direction: 'DESC' };
      },
    },
    pagination: {
      type: Boolean,
      default: false,
    },
    limit: {
      type: Number,
      default: 0,
    },
    categories: {
      type: Array,
      default: null,
    },
  },
  data() {
    return {
      component: 'vy_blog_post',
      loading: true,
      error: false,
      links: {},
      featuredLocal: false,
      params: [
        'field_vy_blog_image',
        'field_vy_blog_image.field_media_image',
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
      if (this.excludedId.length > 0) {
        params.filter.excludeSelf = {
          condition: {
            path: 'id',
            operator: '<>',
            value: this.excludedId,
          },
        };
      }

      if (this.favorites) {
        if (this.isFavoritesTypeEmpty('node', 'vy_blog_post')) {
          this.loading = false;
          return;
        }
        params.filter.includeFavorites = {
          condition: {
            path: 'drupal_internal__nid',
            operator: 'IN',
            value: this.getFavoritesTypeIds('node', 'vy_blog_post'),
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
        .get('jsonapi/node/vy_blog_post', { params })
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

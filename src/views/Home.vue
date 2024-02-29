<template>
  <div class="column is-gapless gap">
    <input ref="searchInput" :class="`input search is-rounded vtb-search ${searchFocusStatus ? 'search-focus' : ''}`"
      v-model="search" type="text" placeholder="查找主播~">
    <div class="columns">
      <div class="column vtb-column"></div>
      <div class="column is-full-mobile is-11-tablet is-10-desktop is-three-fifths-widescreen is-7-fullhd"
        :style="{ height: sumHeight }" ref="container">
        <p v-if="cacheAge">数据缓存于: <span class="tag is-rounded is-info smallMargin">{{ cacheAge }}</span></p>
        <progress class="progress" max="100" v-if="!currentVtbs.length"></progress>
        <card :keyword="search" v-if="mountCard" :vtb="mountCard" hover class="aboveTop" ref="mountCard"></card>
        <transition-group name="flip-list">
          <card :keyword="search" v-for="{ vtb, h } in rankLimit" :vtb="vtb" hover :key="vtb.mid" class="card"
            :style="{ top: h, width: unitWidth }"></card>
        </transition-group>
      </div>
      <div class="column"></div>
    </div>
  </div>
</template>

<script>
import { mapState, mapGetters, mapActions } from 'vuex'
import card from '@/components/card'
import moment from 'moment'

const SHOW = 16

const threshold = Array(2000).fill(0).map((_, i) => 1 / 2000 * i)

export default {
  name: 'home',

  methods: {
    ...mapActions([
      'fetchSecretList',
    ]),
    handleKeyDown(event) {
      // 快捷键事件
      // 搜索拦截（Ctrl+F 或 Cmd+F）
      const isSearchShortcut = (event.ctrlKey || event.metaKey) && event.key === 'f'
      // ESC按钮监听
      const isSearchEsc = (event.key === 'Escape')
      if (isSearchShortcut) {
        // 阻止默认搜索
        event.preventDefault()
        this.searchFocusStatus = true
        this.$refs.searchInput.focus();
      }
      if (isSearchEsc) {
        if (this.searchFocusStatus) {
          this.$refs.searchInput.blur();
          this.searchFocusStatus = false
        }

      }
    },
    handleSearchBlur() {
      if(this.searchFocusStatus){
        this.searchFocusStatus = false
      }
    }
  },

  data() {
    return {
      show: 32,
      search: '',
      intersectionObserver: undefined,
      resizeObserver: undefined,
      unitHeight: 172,
      unitWidth: undefined,
      ratio: 0,
      searchFocusStatus: false,
    }
  },
  components: {
    card,
  },
  watch: {
    mountCard: {
      immediate: true,
      async handler() {
        await this.$nextTick()
        if (this.$refs.mountCard) {
          this.resizeObserver.observe(this.$refs.mountCard.$el)
        }
      },
    },
    secretPage: {
      immediate: true,
      async handler() {
        if (this.secretPage) {
          await this.fetchSecretList()
        }
      }
    }
  },
  computed: {
    ...mapState(['currentVtbs', 'cachedTime']),
    ...mapGetters(['vtbs', 'followerRank', 'liveRank', 'riseRank', 'guardRank', 'secretRank']),
    baseHeight() {
      if (this.cacheAge) {
        return 57 + 86
      }
      return 57
    },
    mountCard() {
      return this.preRank[0]
    },
    sumHeight() {
      return `${this.preRank.length * this.unitHeight}px`
    },
    skip() {
      const total = this.preRank.length
      return Math.max(0, Math.min(total - SHOW, Math.floor(this.ratio * total)))
    },
    cacheAge() {
      if (!this.currentVtbs.length && this.cachedTime) {
        return moment(this.cachedTime).fromNow()
      } else {
        return false
      }
    },
    secretPage() {
      return this.$route.path.includes('secret')
    },
    rank: function () {
      if (this.$route.path.includes('live')) {
        return this.liveRank
      }
      if (this.$route.path.includes('rise')) {
        return this.riseRank
      }
      if (this.$route.path.includes('drop')) {
        return [...this.riseRank].reverse()
      }
      if (this.$route.path.includes('guard')) {
        return this.guardRank
      }
      if (this.$route.path.includes('secret')) {
        return this.secretRank
      }
      return this.followerRank
    },
    preRank() {
      const keys = this.search.toLowerCase().split(' ').filter(Boolean)
      if (keys.length) {
        return this.rank.filter(i => keys.every(key => {
          const user = this.$store.getters.info[i.mid];
          if (user && user.uname && typeof user.uname === 'string') {
            return user.uname.toLowerCase().includes(key);
          }
          return false;
        }));
      }
      return this.rank
    },
    rankLimit: function () {
      return this.preRank
        .slice(this.skip, this.skip + SHOW)
        .map((vtb, i) => ({ vtb, h: `${(i + this.skip) * this.unitHeight + this.baseHeight}px` }))
    },
  },
  created() {
    this.resizeObserver = new ResizeObserver(([{ contentRect: { height, width } }]) => {
      this.unitHeight = height
      this.unitWidth = `${width}px`
    })
    this.intersectionObserver = new IntersectionObserver(([{ intersectionRatio }]) => {
      this.ratio = intersectionRatio
    }, {
      rootMargin: '99999999px 0px -150% 0px',
      root: null,
      threshold,
    })
  },
  mounted() {
    this.intersectionObserver.observe(this.$refs.container)
    window.addEventListener('keydown', this.handleKeyDown)
    this.$refs.searchInput.addEventListener('keydown', this.handleKeyDown)
    this.$refs.searchInput.addEventListener('blur', this.handleSearchBlur)

  },
  destroyed() {
    this.resizeObserver.disconnect()
    this.intersectionObserver.disconnect()
  },
}
</script>

<style scoped>
.vtb-search {
  right: 10px;
  bottom: 10px;
  box-shadow: inset 0 0.0625em 1em rgba(10, 10, 10, 0.05);
  outline: none;
  /* border: none; */
  width: 260px;
  position: fixed;
  z-index: 99;
  backdrop-filter: blur(6px) grayscale(50%);
  background-color: rgba(255, 255, 255, 0.3);
}

.vtb-search:focus {
  box-shadow: inset 0 0.0625em 1em rgba(10, 10, 10, 0.05),
    0 0 3000px 3000px rgba(10, 10, 10, 0.5);
}

.search-focus {
  top: calc(50% - 12px);
  right: calc(50vw - 130px);
  box-shadow: 0 0 3000px 3000px rgba(10, 10, 10, 0.5);

  transition: .5s cubic-bezier(0.39, 0.575, 0.565, 1);
}

.vtb-column {
  padding: 25px;
}

.smallMargin {
  margin-bottom: 6px;
}

.flip-list-move {
  transition: transform 0.5s;
}

.card {
  position: absolute;
}

.aboveTop {
  z-index: -20;
  opacity: 0;
}

.gap {
  padding: 16px;
  padding-left: 32px;
  padding-right: 32px;
}
</style>

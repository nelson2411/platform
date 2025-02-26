<template>
  <div>
    <TButton :icon="icon" :type="type" :label="label" @click="share()" />
    <TPopup v-if="generating" title="Generating poster">
      <div class="p-4">Generating Poster... Please wait...</div>
    </TPopup>
    <TPopup v-if="sharing" title="Share" @close="sharing = false">
      <div class="w-64 space-y-2 py-4">
        <TButton type="nav" @click="copyToClipboard"> Copy Link </TButton>
        <TButton type="nav" download :href="downloadUrl" @click="download">
          Download Poster
        </TButton>
        <TButton
          v-for="(_, platform) in platforms"
          :key="platform"
          type="nav"
          @click="shareTo(platform)"
        >
          {{ platform }}
        </TButton>
        <TButton v-if="nativeShareSupported" type="nav" @click="nativeShare()">
          More
        </TButton>
        <TButton type="nav" @click="refresh()"> Refresh Poster </TButton>
      </div>
    </TPopup>
  </div>
</template>

<script>
import axios from 'axios'
import { openURL } from '~/utils'
import { useCities } from '~/use/cities'
import { useAuth } from '~/use/auth'

export default {
  setup() {
    const { currentCity } = useCities()
    const { uid, account } = useAuth()

    return {
      currentCity,
      uid,
      account,
    }
  },
  props: {
    collection: {
      type: String,
      default: '',
    },
    id: {
      type: String,
      default: '',
    },
    text: {
      type: String,
      default: '',
    },
    place: {
      type: String,
      default: '',
    },
    url: {
      type: String,
      default: '',
    },
    file: {
      type: String,
      default: '',
    },
    fileName: {
      type: String,
      default: '',
    },
    label: {
      type: String,
      default: '',
    },
    icon: {
      type: String,
      default: 'share',
    },
    type: {
      type: String,
      default: 'icon',
    },
  },
  data: () => ({
    sharing: false,
    generating: false,
    downloadUrl: '',
    nativeShareSupported: false,
  }),
  computed: {
    platforms() {
      const text = this.text
      const url = this.url

      return {
        Whatsapp: `whatsapp://send?text=${text} ${url}`,
        Telegram: `https://t.me/share/url?url=${url}&text=${text}`,
        Facebook: `https://www.facebook.com/share.php?display=page&u=${url}&t=${text}`,
        Twitter: `https://twitter.com/intent/tweet?text=${text} ${url}`,
      }
    },
  },
  mounted() {
    this.downloadUrl = this.file
  },
  methods: {
    shareTo(platform) {
      if (!this.platforms[platform]) {
        return
      }

      this.$fire.analytics.logEvent('share', {
        method: platform,
        content_type: this.collection,
        content_id: this.id,
      })

      openURL(this.platforms[platform])
    },
    async refresh() {
      this.sharing = false
      await this.generate()
      await this.share()
    },
    download() {
      this.$fire.analytics.logEvent('share', {
        method: 'Download',
        content_type: this.collection,
        content_id: this.id,
      })
    },
    async copyToClipboard() {
      this.$fire.analytics.logEvent('share', {
        method: 'Link',
        content_type: this.collection,
        content_id: this.id,
      })

      await navigator.clipboard.writeText(this.url)
      this.sharing = false
      this.$toast.success('Link copied to clipboard')
    },
    async generate() {
      if (this.generating) {
        return
      }

      this.$fire.analytics.logEvent('create_poster', {
        collection: this.collection,
      })

      this.generating = true
      this.$nuxt.$loading.start()

      try {
        const result = await axios.get(
          `https://us-central1-wedance-4abe3.cloudfunctions.net/hooks/share${this.$route.path}?timezone=${this.account?.zone}`
        )

        if (!result.data.success) {
          this.$toast.error('Failed to generate a poster')
        }

        this.downloadUrl = result.data.url

        const collection = this.collection === 'profiles' ? 'profiles' : 'posts'

        await this.$fire.firestore
          .collection(collection)
          .doc(this.id)
          .update({ socialCover: this.downloadUrl })
      } catch (e) {
        this.$toast.error(e.message)
      }

      this.generating = false
      this.$nuxt.$loading.finish()
    },
    async share() {
      if (!this.downloadUrl) {
        await this.generate()
      }

      this.$fire.analytics.logEvent('popup_share')
      this.sharing = true

      if (navigator.share && navigator.canShare) {
        this.nativeShareSupported = true
      } else {
        this.nativeShareSupported = false
      }
    },
    async nativeShare() {
      const response = await fetch(this.downloadUrl)
      const blob = await response.blob()
      const file = new File([blob], `${this.fileName}.png`, {
        type: 'image/png',
      })

      const filesArray = [file]

      if (!navigator.canShare({ files: filesArray })) {
        this.nativeShareSupported = false
        return false
      }

      this.$fire.analytics.logEvent('share', {
        method: 'Native',
        content_type: this.collection,
        content_id: this.id,
      })

      try {
        await navigator.share({
          title: this.text,
          url: this.url,
          files: filesArray,
        })
      } catch (e) {}
    },
  },
}
</script>

<template>
  <div>
    <THeader :title="doc.title || '...'" />

    <TSharePreviewPost
      v-if="doc.cover"
      type="Post"
      collection="posts"
      :username="author.username"
      :title="doc.title"
      :description="getExcerpt(doc.description)"
      :photo="doc.cover"
      :styles="doc.styles"
      align="center"
    />

    <TReactions :item="doc" class="p-4 justify-center" />

    <TPost :item="doc" hide-comments show-all />

    <TItemCreator :item="doc" full />

    <TItemComments :reply-to="doc.createdBy" :post-id="doc.id" />
  </div>
</template>

<script>
import { useAuth } from '~/use/auth'
import { useProfiles } from '~/use/profiles'
import { getExcerpt, getMeta, loadDoc, getDateTime } from '~/utils'

export default {
  async asyncData(ctx) {
    return await loadDoc(ctx, 'posts')
  },
  computed: {
    author() {
      return this.getProfile(this.doc.createdBy)
    },
    publishedAt() {
      return getDateTime(this.doc?.createdAt)
    },
  },
  head() {
    return getMeta('posts', this.doc)
  },
  setup() {
    const { uid } = useAuth()
    const { getProfile } = useProfiles()

    return {
      uid,
      getProfile,
      getExcerpt,
    }
  },
}
</script>

<template>
  <div class="bg-dark md:py-4 min-h-screen">
    <div
      class="mx-auto w-full max-w-lg md:rounded md:border md:shadow bg-white"
    >
      <div class="flex justify-between m-4">
        <TInputButtons
          value="events"
          :options="[
            {
              label: 'Post',
              value: 'posts',
              to: `/posts/${item.id || '-'}/edit`,
            },
            {
              label: 'Event',
              value: 'events',
              to: `/events/${item.id || '-'}/edit`,
            },
          ]"
        />
        <button class="cursor-pointer" @click="$router.back()">
          <TIcon name="close" class="cursor-pointer w-4 h-4" />
        </button>
      </div>

      <TForm
        v-model="item"
        :edit-creator="isAdmin()"
        :fields="eventFields"
        show-cancel
        vertical
        submit-label="Save"
        class="bg-white p-4 space-y-4"
        @save="saveItem"
        @cancel="view(item.id)"
      />
    </div>
  </div>
</template>

<script>
import { pickBy } from 'lodash'
import { useAuth } from '~/use/auth'
import { useDoc } from '~/use/doc'
import { useRouter } from '~/use/router'
import { useEvents } from '~/use/events'

export default {
  name: 'EventEdit',
  layout: 'empty',
  middleware: ['auth'],
  props: {
    id: {
      type: String,
      default: '-',
    },
  },
  watch: {
    loading(loading) {
      if (!loading && this.item) {
        if (!this.can('edit', 'posts', this.item)) {
          this.$nuxt.error({ statusCode: 405 })
        }
      }
    },
  },
  mounted() {
    if (this.id === '-') {
      this.item = {
        type: 'event',
        place: this.profile?.place,
        visibility: 'Public',
        form: 'No',
        online: 'No',
        international: 'No',
        claimed: 'No',
        eventType: 'Party',
        duration: 60,
        price: '0 EUR',
        styles: {},
        cover: '',
        organiser: this.profile?.username || '',
        promoter: this.profile?.username || '',
      }
    }
  },
  methods: {
    view(id) {
      if (id && id !== '-') {
        this.$router.push(`/events/${id}`)
      } else {
        this.$router.push(`/events`)
      }
    },
    async copyItem(data) {
      if (!data.name) {
        return
      }

      this.$fire.analytics.logEvent('copy_event')
      const doc = await this.create(data)

      this.$router.push(`/events/${doc.id}`)
    },
    async saveItem(data) {
      data.organisedBy = this.uid
      data.promotedBy = this.uid

      if (data.claimed === 'No') {
        data.organiser = ''
        data.organisedBy = ''
      }

      data = pickBy(data, (v) => v !== undefined)

      if (data.id) {
        this.$fire.analytics.logEvent('update_event')
        await this.update(data.id, data)
        this.view(data.id)
      } else {
        this.$fire.analytics.logEvent('create_event')
        const result = await this.create(data)
        this.view(result.id)
      }
    },
    async removeItem(id) {
      this.$fire.analytics.logEvent('delete_event')
      await this.remove(id)
      this.view()
    },
  },
  setup() {
    const { eventFields } = useEvents()
    const { can, profile, isAdmin, uid } = useAuth()
    const { params } = useRouter()

    const collection = 'posts'

    const { doc: item, load, update, remove, create, loading } = useDoc(
      collection
    )

    if (params.id !== '-') {
      load(params.id)
    }

    return {
      loading,
      item,
      can,
      collection,
      update,
      remove,
      create,
      profile,
      isAdmin,
      uid,
      eventFields,
    }
  },
}
</script>

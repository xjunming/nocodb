<script setup lang="ts">
import { enumColor as colors } from '#imports'

const { lang: currentLang } = useGlobal()

const isRtlLang = $computed(() => ['fa'].includes(currentLang.value))
</script>

<template>
  <a-card :body-style="{ padding: '0px' }" class="w-[300px] shadow-sm !rounded-lg">
    <a-list class="w-full" dense>
      <slot name="before" />

      <a-list-item>
        <nuxt-link
          v-t="['e:docs']"
          class="text-primary !no-underline !text-current"
          target="_blank"
          to="https://docs.nocodb.com/"
        >
          <div class="ml-3 flex items-center text-sm">
            <MdiBookOpenOutline class="text-lg text-accent" />
            <span class="ml-3">{{ $t('labels.documentation') }}</span>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:api-docs']"
          class="text-primary !no-underline !text-current"
          target="_blank"
          to="https://apis.nocodb.com/"
        >
          <div class="ml-3 flex items-center text-sm">
            <MdiJson class="text-lg text-green-500" />
            <!--            todo:  i18n -->
            <span class="ml-3">API {{ $t('labels.documentation') }}</span>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:community:github']"
          class="text-primary !no-underline !text-current"
          to="https://github.com/nocodb/nocodb"
          target="_blank"
        >
          <div class="flex items-center text-sm">
            <mdi-github class="mx-3 text-lg" />
            <div v-if="isRtlLang">
              <!-- us on Github -->
              {{ $t('labels.community.starUs2') }}
              <!-- Star -->
              {{ $t('labels.community.starUs1') }}
              <mdi-star-outline />
            </div>
            <div v-else class="flex items-center">
              <!-- Star -->
              {{ $t('labels.community.starUs1') }}
              {{ ' ' }}
              <!-- us on Github -->
              {{ $t('labels.community.starUs2') }}
            </div>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:community:book-demo']"
          class="!no-underline !text-current"
          to="https://calendly.com/nocodb-meeting"
          target="_blank"
        >
          <div class="flex items-center text-sm">
            <mdi-calendar-month class="mx-3 text-lg" :color="colors.dark[3 % colors.dark.length]" />
            <!-- Book a Free DEMO -->
            <div>
              {{ $t('labels.community.bookDemo') }}
            </div>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:community:discord']"
          class="!no-underline !text-current"
          to="https://discord.gg/5RgZmkW"
          target="_blank"
        >
          <div class="flex items-center text-sm">
            <mdi-discord class="mx-3 text-lg" :color="colors.dark[0 % colors.dark.length]" />
            <!-- Get your questions answered -->
            <div>
              {{ $t('labels.community.getAnswered') }}
            </div>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:community:twitter']"
          class="!no-underline !text-current"
          to="https://twitter.com/NocoDB"
          target="_blank"
        >
          <div class="flex items-center text-sm">
            <mdi-twitter class="mx-3 text-lg" :color="colors.dark[1 % colors.dark.length]" />
            <!-- Follow NocoDB -->
            <div>
              {{ $t('labels.community.followNocodb') }}
            </div>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link v-t="['e:hiring']" class="!no-underline !text-current" target="_blank" to="http://careers.nocodb.com">
          <div class="flex items-center text-sm">
            <!--            todo:  i18n -->
            <div class="ml-3">
              🚀 <span class="ml-3">{{ $t('labels.weAreHiring') }}!!!</span>
            </div>
          </div>
        </nuxt-link>
      </a-list-item>
      <a-list-item>
        <nuxt-link
          v-t="['e:community:reddit']"
          class="!no-underline !text-current"
          target="_blank"
          to="https://www.reddit.com/r/NocoDB/"
        >
          <div class="ml-3 flex items-center text-sm">
            <LogosRedditIcon />
            <span class="ml-4">/r/NocoDB/</span>
          </div>
        </nuxt-link>
      </a-list-item>
    </a-list>
  </a-card>
</template>

<style scoped>
:deep(.ant-list-item) {
  @apply hover:(bg-gray-100 !text-primary);
}
</style>

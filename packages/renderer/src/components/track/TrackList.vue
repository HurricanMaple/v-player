<script setup lang="ts">
import {
  mdiAccountMusic,
  mdiAlbum,
  mdiClockOutline,
  mdiCloseCircleOutline,
  mdiDownload,
  mdiHeart,
  mdiHeartRemove,
  mdiPlaylistMusicOutline,
  mdiPlaylistPlay,
} from '@mdi/js'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'
import { useToast } from 'vue-toastification'
import type { MenuItem } from 'vuetify-ctx-menu/lib/ContextMenuDefine'
import { useContextMenu } from 'vuetify-ctx-menu/lib/main'

import { opPlaylist } from '@/api/music'
import { getSongDownloadUrl } from '@/api/song'
import useDonwload from '@/hooks/useDownload'
import { useCurrentTheme } from '@/hooks/useTheme'
import { usePlayQueueStore } from '@/store/playQueue'
import { useUserStore } from '@/store/user'
import type { Track } from '@/types'
import { specialType } from '@/util/metadata'
const userStore = useUserStore()
const playQueueStore = usePlayQueueStore()
const { themeName } = useCurrentTheme()
const contextMenu = useContextMenu()
const toast = useToast()
const { t } = useI18n()
const router = useRouter()

const props = defineProps<{
  tracks: Track[]
  type: 'album' | 'list' | 'artist' | 'fav'
  ownId?: number | null // 是否是自己的歌单id
  header?: boolean
  offsetIndex?: number
  virtualScrollOptimization?: boolean
  setQueue?: boolean
}>()

const emits = defineEmits<{
  (event: 'removeTrack', payload: number): void
}>()

const eventBus = useEventBus<number>('addToQueue')
const TrackItemHeight = 56
const needScrollNumber = 80
const listHeight = computed(() => {
  const realHeight = props.tracks.length * TrackItemHeight
  return props.tracks.length > needScrollNumber ? needScrollNumber * TrackItemHeight : realHeight
})
const playlists = computed(() => {
  return userStore.createdPlaylists
    .map((i) => {
      return {
        id: i.id,
        name: i.name,
        specialType: i.specialType,
      }
    })
    .filter((playlist) => playlist.specialType !== specialType.fav.type)
})
const className = computed(() => {
  if (props.type !== 'album') {
    return 'list-header'
  } else {
    return 'list-header album-header'
  }
})
const showAlbum = computed(() => {
  return props.type !== 'album'
})
const offsetIndex = computed(() => {
  return props.offsetIndex ?? 1
})
function openMenu(payload: { x: number; y: number; track: Track; liked: boolean }) {
  const { x, y, liked, track } = payload
  const option = {
    theme: themeName.value,
    x,
    y,
    items: genMenu(liked, track),
    offsetFooter: 64,
  }
  contextMenu(option)
}
function genMenu(liked: boolean, track: Track): MenuItem[] {
  const items: MenuItem[] = [
    {
      icon: mdiPlaylistPlay,
      label: t('common.add_to_queue'),
      onClick: (i) => {
        addToQueue(track)
      },
    },
    {
      icon: mdiAccountMusic,
      label: t('common.to_artist'),
      ...(track.ar!.length > 1
        ? {
            children: track.ar?.map((artist) => {
              return {
                label: artist.name,
                onClick: () => {
                  toArtist(artist.id)
                },
              }
            }),
          }
        : {
            onClick: (i) => {
              toArtist(track.ar![0].id)
            },
          }),
    },
    {
      icon: mdiAlbum,
      label: t('common.to_album'),
      onClick: (i) => {
        toAlbum(track.al!.id)
      },
    },
    {
      icon: mdiDownload,
      label: '下载到本地',
      onClick: async (i) => {
        try {
          // todo 获取到的链接直接下载是丢失了歌曲的元数据的, 看有无办法恢复
          const { data } = await getSongDownloadUrl({ id: track.id })
          const artistName = track.ar?.map((i) => i.name)?.join(',')
          const fileName = `${artistName} - ${track.name}.${data.type}`
          useDonwload(data.url, fileName)
        } catch (e) {
          toast.error(t('message.something_wrong'))
        }
      },
    },
    {
      divided: true,
    },
    {
      icon: mdiPlaylistMusicOutline,
      label: t('common.add_to_playlist'),
      children: playlists.value.map((list) => {
        return {
          label: list.name,
          onClick: (i) => {
            trackToPlayList('add', list.id, track.id)
          },
        }
      }),
    },
  ]
  if (liked) {
    items.push({
      icon: mdiHeartRemove,
      label: t('common.remove_from_fav'),
      onClick: (i) => {
        toggleLike(true, track.id)
      },
    })
  } else {
    items.push({
      icon: mdiHeart,
      label: t('common.add_to_fav'),
      onClick: (i) => {
        toggleLike(false, track.id)
      },
    })
  }
  if (props.ownId && props.type !== 'fav') {
    items.push({
      icon: mdiCloseCircleOutline,
      label: t('common.remove_from_playlist'),
      onClick: (i) => {
        // “!”非空断言
        trackToPlayList('del', props.ownId!, track.id)
      },
    })
  }
  return items
}
async function toggleLike(liked: boolean, trackId: number) {
  const success = await userStore.favSong(trackId, !liked)
  if (success) {
    if (liked) {
      toast.success(t('message.remove_fav_success'))
    } else {
      toast.success(t('message.add_fav_success'))
    }
  } else {
    toast.error(t('message.something_wrong'))
  }
}
function addToQueue(track: Track) {
  playQueueStore.addToPlayQueue(track)
}
async function trackToPlayList(op: 'add' | 'del' = 'add', playlistId: number, trackId: number) {
  // add to playlist
  try {
    const { code, message } = await opPlaylist(op, playlistId, trackId)
    if (code === 200) {
      if (op === 'del') {
        emits('removeTrack', trackId)
        toast.success(t('message.remove_list_success'))
      } else {
        toast.success(t('message.add_list_success'))
      }
    } else {
      toast.error(message!)
    }
  } catch (e) {
    toast.error(t('message.something_wrong'))
  }
}
function toArtist(id: number) {
  router.push(`/artist/${id}`)
}
function toAlbum(id: number) {
  router.push(`/album/${id}`)
}
</script>
<template>
  <v-list class="track-list">
    <div v-if="header">
      <div class="px-2 text-caption" :class="[className]">
        <span class="d-flex justify-center">#</span>
        <span>{{ $t('common.title') }}</span>
        <span v-if="showAlbum">{{ $t('main.albums') }}</span>
        <span class="d-flex justify-center align-center"
          ><v-icon size="small"> {{ mdiClockOutline }}</v-icon></span
        >
      </div>
      <v-divider class="mx-4 my-2" />
    </div>
    <RecycleScroller
      v-if="virtualScrollOptimization"
      v-slot="{ item: track, index }"
      class="scroller"
      :style="{
        height: `${listHeight}px`,
      }"
      :items="tracks"
      :item-size="TrackItemHeight"
      key-field="id"
    >
      <track-item
        :track="track"
        :index="index + offsetIndex"
        :album="showAlbum"
        @play="eventBus.emit(track.id, setQueue)"
        @openctxmenu="openMenu"
      />
    </RecycleScroller>
    <template v-else>
      <track-item
        v-for="(track, index) in tracks"
        :key="track.id"
        :track="track"
        :index="index + offsetIndex"
        :album="showAlbum"
        @play="eventBus.emit(track.id, setQueue)"
        @openctxmenu="openMenu"
      />
    </template>
  </v-list>
</template>
<style lang="scss" scoped>
.track-list {
  .list-header {
    display: grid;
    grid-gap: 5px;
    grid-template-columns: [index] 40px [first] 3fr [second] 2fr [last] minmax(140px, 1fr);
    &.album-header {
      grid-template-columns: [index] 40px [first] 4fr [last] minmax(140px, 1fr);
    }
  }
}
</style>

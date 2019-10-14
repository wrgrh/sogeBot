<template>
  <b-container fluid ref="window">
    <b-row>
      <b-col>
        <span class="title text-default mb-2">
          {{ translate('menu.settings') }}
          <small><fa icon="angle-right"/></small>
          {{ translate('menu.translations') }}
        </span>
      </b-col>
    </b-row>

    <panel search @search="search = $event">
      <template v-slot:right>
        <b-pagination
          class="m-0"
          v-model="currentPage"
          :total-rows="rows"
          :per-page="perPage"
          aria-controls="my-table"
        ></b-pagination>
      </template>
    </panel>

    <loading v-if="state.loading === $state.progress || state.settings === $state.progress" />
    <b-table show-empty v-else striped :fields="fields" :items="fItems" small style="cursor: pointer;" :per-page="perPage" :current-page="currentPage">
      <template v-slot:empty="scope">
        <b-alert show class="m-0">
          {{translate('dialog.nothingToShow')}}
        </b-alert>
      </template>
      <template v-slot:cell(current)="data">
        <b-form>
          <b-form-group style="margin:0 !important">
            <b-input-group>
              <b-form-input
                :id="data.item.key"
                v-model="data.item.current"
                @input="updateTranslation(data.item.key, data.item.current)"
                type="text"
              ></b-form-input>
              <b-input-group-append v-if="data.item.current !== data.item.default">
                <b-button variant="primary" @click="data.item.current = data.item.default; revertTranslation(data.item.key)">Revert</b-button>
              </b-input-group-append>
            </b-input-group>
          </b-form-group>
        </b-form>
      </template>
    </b-table>
  </b-container>
</template>

<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator';
import { getSocket } from 'src/panel/helpers/socket';
import { isNil } from 'lodash-es';

@Component({
  components: {
    'loading': () => import('../../components/loading.vue'),
  }
})
export default class translations extends Vue {
  items: { key: string, current: string, default: string }[] = [];
  search: string = '';

  currentPage = 0
  perPage = 10;

  fields = [
    {
      key: 'key',
      label: this.translate('integrations.responsivevoice.settings.key.title'),
      sortable: true,
      thStyle: "width: 375px",
    },
    { key: 'current', label: this.translate('core.permissions.value') },
  ]

  state: {
    loading: number,
  } = {
    loading: this.$state.progress,
  }

  socket = getSocket('/');

  @Watch('search')
  watchSearch() {
    this.currentPage = 0;
  }

  get rows() {
    return this.fItems.length;
  }

  revertTranslation(key) {
    this.socket.emit('responses.revert', { key }, (orig) => {
      console.log('Reverted', key, orig)
    })
  }

  updateTranslation(key, value) {
    this.socket.emit('responses.set', { key, value })
  }

  created() {
    this.state.loading = this.$state.progress;
    this.socket.emit('responses.get', null, (data) => {
      this.items = Object
        .entries(data)
        .map(o => {
          if ((o[1] as any).current.startsWith('{missing')) {
            console.debug(`${o[0]} have missing translation`)
          }
          return {
            key: o[0] as string,
            current: (o[1] as any).current as string,
            default: (o[1] as any).default as string,
          };
        })
        .filter(o => !o.key.startsWith('webpanel') && !o.key.startsWith('ui'))
        .sort((a, b) => {
          var keyA = a.key.toUpperCase(); // ignore upper and lowercase
          var keyB = b.key.toUpperCase(); // ignore upper and lowercase
          if (keyA < keyB) {
            return -1;
          }
          if (keyA > keyB) {
            return 1;
          }

          // names must be equal
          return 0;
      });
      this.state.loading = this.$state.success;
    })
  };

  get fItems() {
    if (this.search.length === 0) return this.items
    return this.items.filter((o) => {
      const isSearchInKey = !isNil(o.key.match(new RegExp(this.search, 'ig')))
      return isSearchInKey
    })
  }
}
</script>
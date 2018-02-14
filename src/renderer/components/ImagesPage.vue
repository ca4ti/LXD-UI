<template>
  <div id="wrapper">
    <div class="container is-fluid">
      <main-header v-bind:current="$route.name" @search-event="handleSearchEvent"></main-header>
      <div id="content-wrapper">
        <div class="columns">
          <div class="column">
            <strong class="title is-5">
              <span class="icon">
                <i class="fa fa-cogs"></i>
              </span>
              Images
            </strong>
            <div class="tabs is-small is-toggle is-pulled-right">
              <ul>
                <li v-bind:class="{ 'is-active': show_images === remote }" v-for="remote in remotes">
                  <a @click="show_images_table(remote)"><span>{{ remote | ucfirst }}</span></a>
                </li>
              </ul>
            </div>
          </div>
        </div>
        <!-- Local Images -->
        <div style="margin-top:-10px" v-show="show_images === 'local'">
          <div class="tabs is-small">
            <ul>
              <li v-bind:class="{ 'is-active': disto === show_distro }" @click="filter_distro(disto)" v-for="disto in distos_list"><a>{{ disto }}</a></li>
            </ul>
          </div>
          <table class="table is-fullwidth is-narrow">
            <thead>
              <tr>
                <th style="width:40%">Description</th>
                <th style="width:12%">Version</th>
                <th style="width:12%">OS</th>
                <th style="width:12%">Release</th>
                <th style="width:12%">Size</th>
                <th style="width:12%">Uploaded</th>
                <th style="width:1%"></th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="image in local_images">
                <td>{{ image.properties.description | ucfirst }}</td>
                <td>{{ image.properties.version ? image.properties.version : '-' }}</td>
                <td>{{ image.properties.os | ucfirst }}</td>
                <td>{{ image.properties.release | ucfirst }}</td>
                <td>{{ formatBytes(image.size) }}</td>
                <td>{{ image.uploaded_at | formatDate }}</td>
                <td><a class="button is-primary is-small">Launch</a></td>
              </tr>
            </tbody>
          </table>
        </div>
        <!-- Public Images -->
        <div style="margin-top:-10px" v-show="show_images !== 'local'">
          <div class="tabs is-small">
            <ul>
              <li v-bind:class="{ 'is-active': disto === show_distro }" @click="filter_distro(disto)" v-for="disto in distos_list"><a>{{ disto }}</a></li>
            </ul>
          </div>
          <table class="table is-fullwidth is-narrow">
            <thead>
              <tr>
                <th style="width:40%">Description</th>
                <th style="width:12%">Version</th>
                <th style="width:12%">OS</th>
                <th style="width:12%">Release</th>
                <th style="width:12%">Size</th>
                <th style="width:12%">Uploaded</th>
                <th style="width:1%"></th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="image in public_images">
                <td>{{ image.properties.description | ucfirst }}</td>
                <td>{{ image.properties.version ? image.properties.version : '-' }}</td>
                <td>{{ image.properties.os | ucfirst }}</td>
                <td>{{ image.properties.release | ucfirst }}</td>
                <td>{{ formatBytes(image.size) }}</td>
                <td>{{ image.uploaded_at | formatDate }}</td>
                <td><a class="button is-primary is-small">Launch</a></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import moment from 'moment'
  import _ from 'lodash'

  import helpers from '../mixins/helpers.js'
  import lxc from '../mixins/lxc.js'
  import MainHeader from './Layout/MainHeader'

  import ElectronStore from 'electron-store'
  const storage = new ElectronStore({
    cwd: 'lxd-ui' // ,
    // encryptionKey: 'obfuscation'
  })

  export default {
    name: 'images-page',
    components: { MainHeader },
    mixins: [lxc, helpers],
    filters: {
      formatDate: function (value) {
        if (value) {
          return moment(String(value)).format('DD/MM/YYYY')
        }
      },
      ucfirst: function (value) {
        if (value) {
          return _.upperFirst(value)
        }
      }
    },
    data () {
      return {
        cache_time: Number(1000 * 86400),
        remotes: [],
        show_images: 'local',
        show_distro: 'Ubuntu',
        search_result: null,
        distos: [],
        images: [],
        btn: {
          refresh_containers: false
        }
      }
    },
    mounted: function () {
      this.$nextTick(() => {
        this.show_images_table('local')
        //
        if (Date.now() - Number(storage.get('remotes_cached', 0)) > this.cache_time) {
          //
          this.lxc_remotes((response) => {
            this.remotes = response
            storage.set('remotes', response)
            storage.set('remotes_cached', Date.now())
          })
        } else {
          this.remotes = storage.get('remotes')
        }
      })
    },
    computed: {
      local_images: function () {
        if (this.show_images !== 'local') {
          // return []
        }
        return this.images.filter((row) => {
          if (_.lowerCase(this.show_distro) !== _.lowerCase(row.properties.os)) {
            return false
          }
          return row
        })
      },
      public_images: function () {
        if (this.show_images !== 'public') {
          // return []
        }
        return this.images.filter((row) => {
          if (_.lowerCase(this.show_distro) !== _.lowerCase(row.properties.os)) {
            return false
          }
          return row
        })
      },
      distos_list: function () {
        return _.uniq(this.distos)
      }
    },
    methods: {
      filter_distro (distro) {
        this.show_distro = distro
      },
      show_images_table (location) {
        this.show_images = location
        this.show_distro = 'Ubuntu'

        this.get_images(location)
      },
      /**
       *
       */
      get_images (remote) {
        //
        if (Date.now() - Number(storage.get('images_cached.' + remote, 0)) > this.cache_time) {
          let architectures = storage.get('info.server.environment.architectures', ['x86_64', 'i686', 'amd64'])
          // for some reason amd64 is not in server environment architectures array :/ so append it if x86_64 is found
          if (_.indexOf(architectures, 'x86_64') > -1) {
            architectures.push('amd64')
          }
          let imagefilter = 'architecture=\'' + architectures.join('|') + '\''
          //
          this.lxc_images(remote + ':', imagefilter, (response) => {
            this.distos = []
            this.images = []
            for (var key in response) {
              this.images.push(response[key])
              this.distos.push(_.upperFirst(response[key].properties.os))
            }
            this.distos = _.uniq(this.distos)
            storage.set('images.' + remote, this.images)
            storage.set('images_distos.' + remote, this.distos)
            storage.set('images_cached.' + remote, Date.now())
          })
        } else {
          this.images = storage.get('images.' + remote)
          this.distos = storage.get('images_distos.' + remote)
        }
      },
      /**
       *
       */
      handleSearchEvent (value) {
        //
        this.lxc_list(value, (response) => {
          this.search_result = response
        })
      },
      open (link) {
        this.$electron.shell.openExternal(link)
      }
    }
  }
</script>

<style>

  #content-wrapper {
    margin-top: -20px;
  }

</style>
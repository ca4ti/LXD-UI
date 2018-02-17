<template>
  <div v-cloak>

    <a class="button is-primary is-small" @click="isActive = true">Launch</a>

    <div class="modal" v-bind:class="{ 'is-active': isActive }">
      <div class="modal-background"></div>
      <div class="modal-card" style="margin-top:-20vh">
        <header class="modal-card-head">
          <p class="modal-card-title">New Container</p>
          <button class="delete" aria-label="close" @click="isActive = false"></button>
        </header>
        <section class="modal-card-body">

          <div class="console" :id="'terminal-' + fingerprint"></div>

          <div v-show="!launching && !launched">
            <div class="field is-horizontal">
              <div class="field-label is-normal">
                <label class="label" for="name">Name</label>
              </div>
              <div class="field-body">
                <div class="field">
                  <p class="control">
                    <input id="name" class="input" type="text" v-model="name" placeholder="Enter name for new container...">
                  </p>
                </div>
              </div>
            </div>
            <div class="field is-horizontal">
              <div class="field-label is-normal">
                <label class="label" for="profile">Profile</label>
              </div>
              <div class="field-body">
                <div class="field">
                  <el-select v-model="profiles" multiple placeholder="Select container profile">
                    <el-option label="default" value="default"></el-option>
                    <el-option label="profile-nameb" value="profile-nameb"></el-option>
                  </el-select>
                </div>
              </div>
            </div>            
            <div class="field is-horizontal">
              <div class="field-label is-normal">
                <label class="label" for="ephemeral">Ephemeral</label>
              </div>
              <div class="field-body">
                <div class="field">
                  <el-switch active-color="#13ce66" v-model="ephemeral"></el-switch>
                </div>
              </div>
            </div>
          </div>
        </section>
        <footer class="modal-card-foot" v-show="!launched">
          <button 
                  @click="launch()" 
                  v-bind:class="{ 'is-loading': launching }" 
                  v-bind:disabled="launching"
                  class="button is-success">Launch</button>
          <button class="button" @click="isActive = false" v-bind:disabled="launching">Cancel</button>
        </footer>
        <footer class="modal-card-foot" v-show="launched">
          <button class="button is-success" @click="isActive = false">Close</button>
        </footer>
      </div>
    </div>
  </div>
</template>

<script>
  // import _ from 'lodash'

  import { Terminal } from 'xterm'
  import * as fit from 'xterm/lib/addons/fit/fit'

  import ElectronStore from 'electron-store'
  const storage = new ElectronStore({
    cwd: 'lxd-ui' // ,
    // encryptionKey: 'obfuscation'
  })

  export default {
    props: ['remote', 'fingerprint'],
    data () {
      return {
        isActive: false,
        name: null,
        profiles: ['default'],
        ephemeral: null,
        launching: false,
        launched: false
      }
    },
    mounted: function () {},
    methods: {
      launch () {
        this.launching = true

        //
        Terminal.applyAddon(fit)
        const xterm = new Terminal({
          useStyle: false,
          screenKeys: false,
          cursorBlink: true
        })

        //
        xterm.open(document.getElementById('terminal-' + this.fingerprint))
        xterm.resize(0, 15)
        xterm.fit()

        // define spawn arguments array
        let spawnProps = []
        spawnProps.push(this.remote + ':' + this.fingerprint)
        spawnProps.push(this.name)
        spawnProps.push('-p ' + this.profiles.join(' -p '))
        spawnProps.push((this.ephemeral ? '-e' : ''))

        //
        const { spawn } = require('child_process')
        const ls = spawn('lxc launch', spawnProps, {
          shell: true
        })

        ls.stdout.on('data', (data) => {
          let line = data.toString().trim()
          if (line !== '') {
            xterm.writeln(line)
          }
        })

        ls.stderr.on('data', (data) => {
          let line = data.toString().trim()
          if (line !== '') {
            xterm.writeln(line)
          }
        })

        ls.on('close', (code) => {
          if (code === 0) {
            this.$notify({
              duration: 2000,
              title: 'Success',
              message: 'Container successfully created.',
              type: 'success'
            })
            xterm.writeln('Container successfully created.')
            this.launching = false
            this.launched = true
          }
        })

        storage.set('images_cached.local', 0)
      },
      close_modal: function () {
        this.$emit('close-modal', true)
      }
    }
  }
</script>

<style scoped>
  .console {
    overflow: hidden;
    width: 100%;
    height: 100%!important;
  }
</style>
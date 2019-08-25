<template>
  <q-page v-bind:class="{ flex: !userSession.isUserSignedIn(), 'flex-center': !userSession.isUserSignedIn() }">
    <q-list
      v-if="!userSession.isUserSignedIn()"
      no-border
    >
      <q-item>
        <q-item-section>
          <div class="row">
            <img
              class="q-pa-lg"
              style="width:150px; height: 150px;"
              alt="Quasar logo"
              src="~assets/quasar-logo-full.svg"
            >
            <q-spinner-hearts
              color="red"
              size="8em"
            />
            <img
              class="q-pa-lg"
              style="width:150px; height: 150px;"
              alt="Blockstack logo"
              src="~assets/blockstack_indigo.svg"
            >
          </div>
        </q-item-section>
      </q-item>
      <q-item>
        <q-item-section>
          <div class="flex flex-center">
            <q-btn
              flat
              @click="login"
              class="q-pa-none q-ma-none"
            >
              <img src="~assets/btn-blockstack-light-l-focus.svg">
            </q-btn>
          </div>
        </q-item-section>
      </q-item>
    </q-list>
    <div
      v-if="profile"
      class="full-width vertical-top"
    >
      <q-card>
        <q-item>
          <q-item-section avatar>
            <q-avatar>
              <img :src="profileImagePath">
            </q-avatar>
          </q-item-section>
          <q-item-section>
            <q-item-label>{{ profile.name() }}</q-item-label>
            <q-item-label caption>{{ username }}</q-item-label>
          </q-item-section>
          <q-card-actions>
            <q-btn
              color="accent"
              no-caps
              @click="logout"
            >Sign out</q-btn>
          </q-card-actions>
        </q-item>
      </q-card>
      <q-input
        class="q-ma-md"
        v-model="searchText"
        placeholder="Search Blockstack names"
        dense
        borderless
        input-class="text-weight-light"
      >
        <template v-slot:prepend>
          <q-icon name="search" />
        </template>
        <template
          v-if="searchText"
          v-slot:append
        >
          <q-icon
            name="clear"
            class="cursor-pointer"
            @click="searchText = ''"
          />
        </template>
      </q-input>
    </div>
    <q-spinner-hearts
      class="full-width"
      :color="inProgress ? 'red' : 'white'"
      size="5em"
    />
    <div
      class="q-pa-md row items-start q-gutter-md"
      v-if="searchResults"
    >
      <q-card
        class="my-card"
        v-for="result in searchResults"
        :key="result.fullyQualifiedName"
      >
        <q-item>
          <q-item-section avatar>
            <q-avatar>
              <img :src="getProfilePic(result)">
            </q-avatar>
          </q-item-section>
          <q-item-section>
            <q-item-label>{{ result.profile.name }}</q-item-label>
            <q-item-label caption>{{ result.username }}</q-item-label>
          </q-item-section>
        </q-item>
      </q-card>
    </div>
    <div
      class="q-pa-md q-gutter-sm"
      v-if="profile"
    >
      <q-editor
        v-model="editor"
        :definitions="{
        upload: {
          tip: 'Upload to Gaia hub',
          icon: 'cloud_upload',
          label: 'Upload',
          handler: promptForFilename
        }
      }"
        :toolbar="[
        ['bold', 'italic', 'strike', 'underline'],
        ['upload']
      ]"
      />
    </div>
    <div
      v-if="files && files.length"
      class="q-pa-md q-gutter-md"
    >
      <q-list
        link
        dense
        bordered
        padding
        class="rounded-borders"
      >
        <q-item-label header>Uploaded files</q-item-label>
        <q-item
          v-for="(file, index) in files"
          :key="file"
          clickable
          v-ripple
        >
          <q-item-section
            class="q-pa-none"
            @click.native="downloadIt(file)"
          >
            {{ `${index + 1}. ${file}` }}
          </q-item-section>
          <q-item-section
            top
            side
          >
            <q-btn
              color="red"
              size="12px"
              flat
              dense
              round
              icon="delete"
              @click="deleteIt(file)"
            />
          </q-item-section>
        </q-item>
      </q-list>
    </div>
    <q-dialog
      v-model="prompt"
      persistent
    >
      <q-card style="min-width: 400px">
        <q-card-section>
          <div class="text-h6">File name</div>
        </q-card-section>
        <q-card-section>
          <q-input
            dense
            v-model="fileName"
            autofocus
            @keyup.enter="prompt = false"
          />
        </q-card-section>
        <q-card-actions
          align="right"
          class="text-primary"
        >
          <q-btn
            flat
            label="Cancel"
            v-close-popup
          />
          <q-btn
            flat
            label="Upload"
            @click="uploadIt"
            v-close-popup
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<style>
</style>

<script>
import { UserSession, Person } from 'blockstack'
export default {
  name: 'Index',
  data () {
    return {
      coreAPIURL: 'https://core.blockstack.org/v1',
      defaultProfilePic: 'assets/blockstack_indigo.svg',
      userSession: new UserSession(),
      searchResults: [],
      searchText: '',
      editor: '',
      prompt: false,
      fileName: 'new_file',
      inProgress: false,
      files: []
    }
  },
  created () {
    if (this.userSession.isUserSignedIn()) {
      if (window.location.href.includes('authResponse')) {
        window.location.replace(window.location.origin)
      }
      this.listFiles()
    } else if (this.userSession.isSignInPending()) {
      this.userSession.handlePendingSignIn()
        .then((userData) => {
          window.location.replace(window.location.origin)
        })
    }
  },
  computed: {
    profile () {
      return this.userSession.isUserSignedIn() ? new Person(this.userSession.loadUserData().profile) : null
    },
    profileImagePath () {
      var imagePath = this.defaultProfilePic
      if (this.profile && this.profile.avatarUrl()) {
        imagePath = this.profile.avatarUrl()
      }
      return imagePath
    },
    username () {
      return this.userSession.isUserSignedIn() ? this.userSession.loadUserData().username : null
    }
  },
  watch: {
    searchText: function (value) {
      if (value) {
        this.searchProfile(value)
      } else {
        this.searchResults = []
      }
    }
  },
  methods: {
    login () {
      const origin = window.location.origin
      this.userSession.redirectToSignIn(
        origin,
        `${origin}/manifest.json`,
        ['store_write'])
    },
    logout () {
      if (this.userSession.isUserSignedIn()) {
        this.userSession.signUserOut()
        this.$router.push('/')
        window.location.replace(window.location.origin)
      }
    },
    searchProfile () {
      this.$axios.get(`${this.coreAPIURL}/search?query=${this.searchText}`)
        .then((response) => {
          if (response.data && response.data.results && response.data.results.length) {
            this.searchResults = response.data.results
          }
        })
        .catch((error) => {
          console.log(error)
        })
    },
    getProfilePic (result) {
      let profilePic = this.defaultProfilePic
      if (result.profile.image && result.profile.image.length) {
        const avatarImage = result.profile.image.find((i) => i.name === 'avatar')
        if (avatarImage) {
          profilePic = avatarImage.contentUrl
        }
      }
      return profilePic
    },
    uploadIt () {
      this.inProgress = true
      this.userSession.putFile(this.fileName, this.editor)
        .then(url => {
          this.inProgress = false
          this.fileName = 'new_file'
          console.log(`URL: ${url}`)
          this.listFiles()
          this.$q.notify({ message: `Uploaded '${this.fileName}'` })
        })
    },
    downloadIt (name) {
      this.inProgress = true
      this.userSession.getFile(name)
        .then(data => {
          this.inProgress = false
          this.editor = data
          this.$q.notify({ message: `Downloaded '${name}'` })
        })
    },
    deleteIt (name, index) {
      this.inProgress = true
      this.userSession.deleteFile(name)
        .then(data => {
          this.inProgress = false
          this.listFiles()
          this.$q.notify({ message: `Deleted '${name}'` })
        })
    },
    listFiles () {
      this.files = []
      this.userSession.listFiles((name) => {
        this.files.push(name)
        return true
      })
    },
    promptForFilename () {
      if (this.editor) {
        this.prompt = true
      }
    }
  }
}
</script>

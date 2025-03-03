<template>
  <div :class="{ theme: true, dark: isDarkTheme }">
    <div
      class="has-text-centered mt2 loading-info xyz-in"
      xyz="fade"
      v-if="user && isDataLoading"
    >
      <span>{{ $t('main.loading_data') }}...</span>
      <spinner class="mt2" />
    </div>
    <router-view v-else />

    <preview-modal
      :active="previewFileIdToShow.length > 0"
      :preview-file-id="previewFileIdToShow"
      @cancel="() => $store.commit('HIDE_PREVIEW_FILE')"
    />
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import PreviewModal from '@/components/modals/PreviewModal'
import Spinner from '@/components/widgets/Spinner'
import crisp from '@/lib/crisp'
import localPreferences from '@/lib/preferences'

export default {
  name: 'app',

  components: {
    PreviewModal,
    Spinner
  },

  computed: {
    ...mapGetters([
      'assetMap',
      'assetTypeMap',
      'currentEpisode',
      'currentProduction',
      'episodeMap',
      'isCurrentUserAdmin',
      'isDataLoading',
      'isPreviewFileDisplayed',
      'isDarkTheme',
      'isLoginLoading',
      'isSavingCommentPreview',
      'isTVShow',
      'previewFileIdToShow',
      'route',
      'personMap',
      'productionMap',
      'sequenceMap',
      'shotMap',
      'taskMap',
      'taskStatusMap',
      'taskTypeMap',
      'user'
    ])
  },

  mounted() {
    if (localStorage.getItem('dark-theme') === 'true' && !this.isDarkTheme) {
      this.$store.commit('TOGGLE_DARK_THEME')
      document.documentElement.style.background = '#36393F'
      document.body.style.background = '#36393F'
    } else {
      document.documentElement.style.background = '#FFF'
      document.body.style.background = '#FFF'
    }
    const supportChat = localPreferences.getBoolPreference('support:show', true)
    this.setSupportChat(supportChat)
    crisp.init(supportChat)
    this.setMainConfig()
  },

  metaInfo: {
    link: [
      {
        rel: 'icon',
        href: '/static/favicon.ico'
      }
    ]
  },

  methods: {
    ...mapActions([
      'getOrganisation',
      'loadTask',
      'loadAsset',
      'loadAssetType',
      'loadComment',
      'loadEpisode',
      'loadOpenProductions',
      'loadPerson',
      'loadPersonTasks',
      'loadProduction',
      'loadSequence',
      'loadShot',
      'loadTaskStatus',
      'loadTaskType',
      'refreshMetadataDescriptor',
      'removeAsset',
      'setMainConfig',
      'setSupportChat'
    ]),

    onAssignation(eventData, assign = true) {
      const personId = eventData.person_id
      const selectedTaskIds = [eventData.task_id]

      // for entity lists
      if (assign) {
        this.$store.commit('ASSIGN_TASKS', { selectedTaskIds, personId })
      } else {
        this.$store.commit('UNASSIGN_TASKS', selectedTaskIds)
      }

      // for todo lists
      if (this.route.path.indexOf(eventData.person_id) > 0) {
        this.loadPersonTasks({
          personId: eventData.person_id,
          forced: true
        })
      }
    }
  },

  watch: {
    isDarkTheme() {
      if (this.isDarkTheme) {
        document.documentElement.style.background = '#36393F'
        document.body.style.background = '#36393F'
      } else {
        document.documentElement.style.background = '#FFF'
        document.body.style.background = '#FFF'
      }
    }
  },

  socket: {
    events: {
      'project:new'(eventData) {
        if (!this.productionMap.get(eventData.project_id)) {
          this.loadProduction(eventData.project_id)
        }
      },

      'project:update'(eventData) {
        if (this.productionMap.get(eventData.project_id)) {
          this.loadProduction(eventData.project_id)
        } else {
          this.loadOpenProductions()
        }
      },

      'project:delete'(eventData) {
        if (this.productionMap.get(eventData.project_id)) {
          this.$store.commit('REMOVE_PRODUCTION', { id: eventData.project_id })
        }
      },

      'sequence:new'(eventData) {
        if (
          !this.sequenceMap.get(eventData.sequence_id) &&
          this.currentProduction &&
          this.currentProduction.id === eventData.project_id
        ) {
          this.loadSequence(eventData.sequence_id)
        }
      },

      'sequence:update'(eventData) {
        if (this.sequenceMap.get(eventData.sequence_id)) {
          this.loadSequence(eventData.sequence_id)
        }
      },

      'sequence:delete'(eventData) {
        if (this.sequenceMap.get(eventData.sequence_id)) {
          this.$store.commit('REMOVE_SEQUENCE', { id: eventData.sequence_id })
        }
      },

      'episode:new'(eventData) {
        if (
          !this.episodeMap.get(eventData.episode_id) &&
          this.currentProduction &&
          this.currentProduction.id === eventData.project_id
        ) {
          this.loadEpisode(eventData.episode_id)
        }
      },

      'episode:update'(eventData) {
        if (this.episodeMap.get(eventData.episode_id)) {
          this.loadEpisode(eventData.episode_id)
        }
      },

      'episode:delete'(eventData) {
        if (this.episodeMap.get(eventData.episode_id)) {
          this.$store.commit('REMOVE_EPISODE', { id: eventData.episode_id })
        }
      },

      'shot:new'(eventData) {
        if (
          !this.shotMap.get(eventData.shot_id) &&
          this.currentProduction &&
          this.currentProduction.id === eventData.project_id &&
          (!this.isTVShow || this.currentEpisode.id === eventData.episode_id)
        ) {
          setTimeout(() => {
            this.loadShot(eventData.shot_id)
          }, 1000)
        }
      },

      'shot:update'(eventData) {
        if (this.shotMap.get(eventData.shot_id)) {
          this.loadShot(eventData.shot_id)
        }
      },

      'shot:delete'(eventData) {
        if (this.shotMap.get(eventData.shot_id)) {
          this.$store.commit('REMOVE_SHOT', { id: eventData.shot_id })
        }
      },

      'asset:new'(eventData) {
        if (
          !this.assetMap.get(eventData.asset_id) &&
          this.currentProduction &&
          this.currentProduction.id === eventData.project_id
        ) {
          setTimeout(() => {
            this.loadAsset(eventData.asset_id)
          }, 1000)
        }
      },

      'asset:update'(eventData) {
        if (this.assetMap.get(eventData.asset_id)) {
          this.loadAsset(eventData.asset_id)
        }
      },

      'asset:delete'(eventData) {
        if (this.assetMap.get(eventData.asset_id)) {
          this.$store.commit('REMOVE_ASSET', { id: eventData.asset_id })
        }
      },

      'task:delete'(eventData) {
        const task = this.taskMap.get(eventData.task_id)
        if (task) {
          this.$store.commit('DELETE_TASK_END', task)
        }
      },

      'task-type:new'(eventData) {
        if (!this.taskTypeMap.get(eventData.task_type_id)) {
          this.loadTaskType(eventData.task_type_id)
        }
      },

      'task-type:update'(eventData) {
        // Do nothing to avoid side effects when reordering task types.
      },

      'task-type:delete'(eventData) {
        if (this.taskTypeMap.get(eventData.task_type_id)) {
          this.$store.commit('DELETE_TASK_TYPE_END', {
            id: eventData.task_type_id
          })
        }
      },

      'task-status:new'(eventData) {
        if (!this.taskStatusMap.get(eventData.task_status_id)) {
          this.loadTaskStatus(eventData.task_status_id)
        }
      },

      'task-status:update'(eventData) {
        if (this.taskStatusMap.get(eventData.task_status_id)) {
          this.loadTaskStatus(eventData.task_status_id)
        }
      },

      'task-status:delete'(eventData) {
        if (this.taskStatusMap.get(eventData.task_status_id)) {
          this.$store.commit('DELETE_TASK_STATUS_END', {
            id: eventData.task_status_id
          })
        }
      },

      'asset-type:new'(eventData) {
        if (!this.assetTypeMap.get(eventData.asset_type_id)) {
          this.loadAssetType(eventData.asset_type_id)
        }
      },

      'asset-type:update'(eventData) {
        if (this.assetTypeMap.get(eventData.asset_type_id)) {
          this.loadAssetType(eventData.asset_type_id)
        }
      },

      'asset-type:delete'(eventData) {
        if (this.assetTypeMap.get(eventData.asset_type_id)) {
          this.$store.commit('DELETE_ASSET_TYPE_END', {
            id: eventData.asset_type_id
          })
        }
      },

      'person:new'(eventData) {
        if (!this.personMap.get(eventData.person_id)) {
          this.loadPerson(eventData.person_id)
        }
      },

      'person:update'(eventData) {
        if (this.personMap.get(eventData.person_id)) {
          this.loadPerson(eventData.person_id)
        }
      },

      'person:delete'(eventData) {
        const person = this.personMap.get(eventData.person_id)
        if (person) {
          this.$store.commit('DELETE_PEOPLE_START', person)
          this.$store.commit('DELETE_PEOPLE_END', person)
        }
      },

      'task:assign'(eventData) {
        this.onAssignation(eventData)
      },

      'task:unassign'(eventData) {
        this.onAssignation(eventData, false)
      },

      'comment:new'(eventData) {
        const commentId = eventData.comment_id
        if (
          !this.isSavingCommentPreview &&
          this.taskMap.get(eventData.task_id)
        ) {
          this.loadComment({ commentId }).catch(console.error)
        }
      },

      'task:update'(eventData) {
        if (this.taskMap.get(eventData.task_id)) {
          this.$nextTick(() => {
            this.loadTask({ taskId: eventData.task_id })
          })
        }
      },

      'task:update-casting-stats'(eventData) {
        const task = this.taskMap.get(eventData.task_id)
        if (task) {
          this.$store.commit('UPDATE_TASK', {
            task,
            nbAssetsReady: eventData.nb_assets_ready
          })
        }
      },

      'episode:casting-update'(eventData) {
        const episode = this.episodeMap.get(eventData.episode_id)
        if (episode) {
          this.$store.commit('UPDATE_EPISODE', {
            id: episode.id,
            nb_entities_out: eventData.nb_entities_out
          })
        }
      },

      'shot:casting-update'(eventData) {
        const shot = this.shotMap.get(eventData.shot_id)
        if (shot) {
          this.$store.commit('UPDATE_SHOT', {
            id: shot.id,
            nb_entities_out: eventData.nb_entities_out
          })
        }
      },

      'metadata-descriptor:new'(eventData) {
        this.refreshMetadataDescriptor(eventData.metadata_descriptor_id)
      },

      'metadata-descriptor:update'(eventData) {
        this.refreshMetadataDescriptor(eventData.metadata_descriptor_id)
      },

      'metadata-descriptor:delete'(eventData) {
        this.$store.commit('DELETE_METADATA_DESCRIPTOR_END', {
          id: eventData.metadata_descriptor_id
        })
      },

      'organisation:update'(eventData) {
        if (this.isCurrentUserAdmin) {
          this.getOrganisation()
        }
      }
    }
  }
}
</script>

<style lang="scss">
:focus {
  outline: none;
}
::-moz-focus-inner {
  border: 0;
}

@font-face {
  font-family: Lato;
  font-style: normal;
  font-weight: 400;
  src: url(./assets/fonts/Lato.woff2) format('woff2');
}

html {
  height: 100%;
  overflow-y: auto;
}

body {
  height: 100%;
  min-height: 100%;
  width: 100%;
  background: #eee;
  overflow: auto;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  border: 0;
}

.dark {
  .hero {
    background-color: $dark-grey;
  }

  .input,
  select,
  textarea,
  .page,
  .loading-info,
  .side-column {
    background: #36393f;
    color: $white-grey;
  }

  textarea.input:focus {
    border-color: $green;
  }

  strong {
    color: white;
  }

  code {
    background: $dark-grey;
    border-color: black;
    color: $white-grey;
  }

  textarea[disabled] {
    background: #36393f;
    color: #bbb;
    border-color: #25282e;
  }

  select,
  textarea,
  .input {
    border-color: #25282e;
  }

  .select::after {
    border-color: #00b242;
  }

  .is-top select {
    background-color: #222427;
  }

  .title,
  .subtitle {
    color: $white-grey;
  }

  label.label {
    color: $white-grey;
  }

  .box .title,
  .box {
    background: #3d4048;
    color: $white-grey;
  }

  .button.is-link {
    background: transparent;
    border-color: transparent;
    color: #dddddd;
  }

  .is-link:hover {
    color: #dddddd;
    background: #5e6169;
  }

  .button {
    background: #4e5159;
    border-color: #25282e;
    color: $white-grey;
  }

  .button.is-danger {
    background: #ff2b56;
  }

  .main-button {
    background: #00b242;
  }

  .add-comment .select select {
    background: #4e5159;
  }

  .hero .box h1.title {
    color: #ddd;
  }

  .splitted-table {
    border-left: 1px solid #36393f;

    tr {
      border-right: 1px solid #25282e;
      border-left: 1px solid #25282e;
    }

    thead tr {
      border-right: 1px solid transparent;
      border-left: 1px solid transparent;
    }

    thead tr a {
      color: #7a7a7a;
    }

    .table-body {
      padding-top: 1em;
      position: relative;
      z-index: 1;
    }

    tbody {
      border-bottom: 1px solid #25282e;
    }

    tbody tr:first-child {
      border-top: 0;
      background: transparent;
    }

    .empty-line {
      border: 0;
    }

    .empty-line td {
      border-color: #36393f;
      background: #36393f;
      border: 0;
    }
  }

  .search-input {
    border-color: #888;
  }

  .erase-search .tag {
    background-color: #999;
    color: #333;
  }

  .erase-search .tag:hover {
    background-color: #ccc;
  }

  .tabs a {
    color: $white-grey;
  }

  .tabs a:hover {
    border-color: $white-grey;
  }

  .tabs li.is-active a:hover {
    border-color: #00c252;
    color: #00c252;
  }

  .tabs li.is-active a:hover {
    border-color: #00c252;
    color: #00c252;
  }

  .search-queries .tag {
    color: #eee;
    background-color: #5e6169;
  }

  .message {
    background: $dark-grey;
    .message-body {
      border-left: 5px solid $dark-grey-lightest;
      color: $white-grey;
    }
  }

  .vdp-datepicker__calendar {
    background-color: #36393f;
    border-color: #25282e;

    .prev,
    .next,
    .day__month_btn,
    header span:hover {
      background: #36393f;
    }

    header .prev::after,
    header .prev::after {
      border-right-color: #eee;
    }

    header .next::after,
    header .next::after {
      border-left-color: #eee;
    }

    header .next.disabled::after,
    header .next.disabled::after {
      border-left-color: #666;
    }

    .cell.year.disabled,
    .cell.month.disabled,
    .cell.day.disabled {
      color: $grey;
    }
  }

  .hero .control .icon {
    color: #555;
  }

  .v-autocomplete-input {
    background: $dark-grey;
    color: white;
  }

  h2 {
    border-bottom: 1px solid $grey;
  }
} // End dark theme

#app .router-link-active {
  color: #00d1b2;
}

.loading-info {
  background: white;
}

.hidden {
  display: none !important;
}

.visuallyhidden {
  border: 0 !important;
  clip: rect(1px, 1px, 1px, 1px) !important;
  -webkit-clip-path: inset(50%) !important;
  clip-path: inset(50%) !important;
  height: 1px !important;
  margin: -1px !important;
  overflow: hidden !important;
  padding: 0 !important;
  position: absolute !important;
  width: 1px !important;
  white-space: nowrap !important;
}

.page {
  padding: 0.5em 2em;
  padding-top: 70px;
  background: white;
  height: 100vh;
}

th.actions {
  min-width: 160px;
}

td.actions {
  min-width: 160px;
}

ul {
  list-style-type: disc;
  margin-left: 1em;
}

h2 {
  border-bottom: 1px solid var(--border);
  color: var(--text);
  font-size: 1.3em;
  font-weight: 500;
  margin-top: 2em;
  margin-bottom: 1em;
  text-transform: uppercase;
}

.hero {
  background-color: #f3f3f3;
}

.avatar {
  border-radius: 50%;
  color: white;
}

.avatar img {
  border-radius: 50%;
  width: 40px;
  height: 40px;
  margin-right: 0px;
}

.mention,
.mention:hover {
  color: $blue;
}

.timecode {
  border: 1px solid $blue;
  border-radius: 5px;
  color: $blue;
  font-size: 0.9em;
  padding: 3px 4px;

  &:hover {
    border-color: $blue-light;
    color: $blue-light;
  }
}

.th-project {
  width: 30px;
  border-radius: 50%;
}

.grey-background {
  background-color: #cfcfcf;
}

td strong {
  font-size: 1.2em;
}

tr .actions p {
  margin-bottom: 0;
}

tr td.actions button,
tr td.actions a {
  opacity: 0;
  color: #999;
}

tr th.actions a {
  color: #999;
}

tr:hover .actions button,
tr:hover .actions a {
  opacity: 1;
}

a {
  color: #999;
}

a:hover {
  color: #999;
}

.info {
  margin-top: 1em;
  font-size: 1.4em;
}

.canceled td:not(.actions),
.canceled th,
.canceled {
  text-decoration: line-through;
  opacity: 0.7;

  div span:first-child {
    text-decoration: line-through;
  }
}

.field {
  margin-bottom: 2em;
}

.pa0 {
  padding: 0em;
}

.pa1 {
  padding: 1em;
}

.pb0 {
  padding-bottom: 0em;
}

.pb1 {
  padding-bottom: 1em;
}

.pt0 {
  padding-top: 0em;
}

.mauto {
  margin: auto;
}

.ml05 {
  margin-left: 0.5em;
}

.ml1 {
  margin-left: 1em;
}

.mr0 {
  margin-right: 0em;
}

.mr05 {
  margin-right: 0.5em;
}

.mr1 {
  margin-right: 1em;
}

.mt05 {
  margin-top: 0.5em;
}

.mt1 {
  margin-top: 1em;
}

.mt15 {
  margin-top: 1.5em;
}

.mt2 {
  margin-top: 2em;
}

.mb0 {
  margin-bottom: 0;
}

.mb05 {
  margin-bottom: 0.5em;
}

.mb1 {
  margin-bottom: 1em;
}

.mb2 {
  margin-bottom: 2em;
}

.filler {
  flex: 1;
}

.z300 {
  z-index: 300000;
}

label.label {
  color: $grey;
  text-transform: uppercase;
  font-size: 0.8em;
  letter-spacing: 1px;
  margin-left: 2px;
}

.subtitle {
  color: $grey;
  font-weight: 500;
  letter-spacing: 0.1em;
  text-transform: uppercase;
}

texarea,
input.input {
  padding: 1em;
  height: 3em;
}

.select select {
  border-radius: 10px;
}

.select select:hover,
.select select:active,
.select select:focus,
input.input:focus {
  border-color: #00b242;
  outline: none;
}

.button,
.button.is-small {
  border-radius: 2em;
}

.datepicker .input.is-small {
  font-size: 1em;
  padding: 0.5em;
  height: 2.3em;
  width: 100px;
  border-radius: 10px;
}

.button:focus {
  box-shadow: none;
}

.button.is-primary {
  border-radius: 2px;
  background: #00b242;
}

.button.is-primary:hover {
  background: #67be4b;
}

.big-button {
  border-radius: 2em;
  font-weight: bold;
  background: #00b242;
  border-color: #00b242;
  color: white;
  font-size: 1.3em;
  max-width: 280px;
  margin: 1em auto;
}

.big-button:hover {
  color: white;
  background: #67be4b;
}

input[type='checkbox'] {
  cursor: pointer;
}

textarea.input {
  border-radius: 10px;
}

textarea.input:focus {
  border-color: $green;
}

.error {
  color: #ff1f4b;
}

.success {
  color: #00b242;
}

.strong {
  font-weight: bold;
}

.pull-right {
  float: right;
}

.footer-info {
  font-style: italic;
}

.container {
  max-width: 400px;
  color: #4a4a4a;
}

.main-button {
  border-radius: 5px;
  min-height: 2.6em;
  color: white;
  border-color: #5e60ba;
  padding: 12px 12px 12px 12px;
  margin: 0.3em 0 0em 0;
  font-size: 1.4em;
  font-weight: 500;
  letter-spacing: 1px;
  background: #00b242;
  color: #fff;
  border: 0;
  transition: all 0.15s ease;
  width: 100%;
  display: block;
  text-align: center;
}

.main-button:hover {
  background: #67be4b;
  color: #fff;
}

.main-button:focus {
  outline: 0;
}

.modal-content {
  max-height: calc(100vh - 140px);
  .box {
    border-radius: 0.5em;
    padding: 2.8em 3em 3em 3em;

    h1.title {
      font-family: Lato;
      font-weight: 400;
      font-size: 3em;
      margin-top: 0;
      padding-top: 0;
      margin-bottom: 1em;
      text-transform: capitalize;
      border: 0;
    }

    .button {
      border-radius: 2em;
    }
  }
}

.hero .box {
  margin-top: 30%;
  padding: 3em 2em 2em 2em;
  border-radius: 2px;
  box-shadow: rgba(0, 0, 0, 0.14902) 0px 1px 1px 0px,
    rgba(0, 0, 0, 0.09804) 0px 1px 2px 0px;
}

.box h1.title {
  color: #6a6a6a;
  font-weight: 500;
  font-size: 1.4em;
  line-height: 1.6em;
}

.hero .box .field {
  margin-bottom: 1em;
}

.hero .box .input {
  height: 2.4em;
}

.hero .box .input:focus {
  border: 1px solid #00b242;
}

.button .icon.is-small:first-child:last-child {
  margin-right: 0em;
}

.actions .button .icon.is-small.icon-only:first-child:last-child {
  margin-right: 0em;
}

.actions .button .icon.is-small:first-child:last-child {
  margin-right: 0em;
}

.button .icon.is-small {
  height: 1.1rem;
  line-height: 1.1rem;
  width: 1.1rem;
}

.button.highlighted {
  background: #00b242;
  color: white;
}

.search-input {
  border: 0;
  box-shadow: none;
  border-radius: 0;
}

.query-list {
  margin-bottom: 2em;
  margin-left: 1em;
  max-width: 95%;
}

.query-list .tag {
  margin-right: 1em;
  margin-bottom: 0.2em;
  border: 1px solid transparent;
}

.query-list .tag .delete {
  transform: rotate(45deg) scale(0.7);
}

.query-list .tag:hover {
  transform: scale(1.1);
}

.fixed-page {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  display: flex;
  flex-direction: column;
  padding-top: 60px;

  min-height: 0;
}

.page-header,
.page-header.level {
  margin-top: 2em;
  margin-bottom: 0;
}

.data-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
  margin-top: 2em;
}

.table td {
  vertical-align: middle;
}

.table td.metadata-descriptor {
  word-wrap: anywhere;
}

.table-header-wrapper {
  overflow: hidden;
}

.table-header {
  display: block;
  width: 100%;
  margin-bottom: 0;
  flex-wrap: wrap;
  position: relative;

  thead th {
    border-width: 0 0 1px;
    font-size: 0.9em;

    &.metadata-descriptor,
    &.validation-cell {
      .descriptor-name,
      .validation-name {
        flex: 1;
      }
    }
  }

  th.actions {
    width: 100%;
  }

  th:hover .header-icon {
    opacity: 100;
  }

  .header-icon {
    width: 15px;
    cursor: pointer;
    opacity: 0;
  }
}

.table-body {
  flex: 1;
  overflow: auto;
  min-height: 1px;
  border-radius: 10px;
}

.table {
  margin-bottom: 0;
}

.table-info {
  margin-top: 1em;
}

.table-body .table tr:hover {
  background: $light-purple;
}

.table tr.type-header {
  border: 1px solid transparent;
  font-size: 1.1em;
}

.table tr.type-header:hover {
  background: transparent;
}

.table tr.type-header td {
  font-weight: bold;
  padding-left: 0.3em;
}

.splitted-table tbody tr {
  border-right: 1px solid #ccc;
  border-left: 1px solid #ccc;
}

.splitted-table thead tr {
  border-right: 1px solid transparent;
  border-left: 1px solid transparent;
}

.splitted-table thead tr a {
  color: #7a7a7a;
}

.splitted-table .table-body {
  padding-top: 1em;
  position: relative;
  z-index: 1;
}

.splitted-table tbody tr:first-child {
  color: #999;
  text-transform: uppercase;
}

.splitted-table tbody:last-child .empty-line:last-child {
  border: 0;
}

.splitted-table tbody {
  border-bottom: 1px solid #ccc;
}

.splitted-table {
  margin-top: 1em;
}

.table-body {
  position: relative;
  z-index: 1;
}

tbody:last-child .empty-line:last-child {
  border: 0;
}

.table-body .table .empty-line {
  background: inherit;
}

.table-header-wrapper {
  position: relative;
}

.table th {
  vertical-align: middle;
}

.datatable-wrapper {
  overflow: auto;
  margin-bottom: 1rem;
}

.datatable {
  position: relative;
  overflow-x: auto;
  white-space: nowrap;
  text-align: left;
  border-collapse: separate;
  min-width: calc(100% - 1px);
  width: auto;

  th {
    position: sticky;
  }

  th.number-cell,
  td.number-cell {
    text-align: right;
    input {
      text-align: right;
    }
  }

  .datatable-row {
    .thumbnail-wrapper,
    .thumbnail-picture,
    .thumbnail-picture.thumbnail-empty {
      margin: 0 0.35rem 0 0;
    }
    &:first-child {
      border-bottom-left-radius: 10px;
      border-bottom-right-radius: 10px;
    }

    .thumbnail-wrapper {
      .thumbnail-picture {
        margin: 0;
      }
    }

    td {
      border-top: 1px solid var(--border);
      border-left: 1px solid var(--border);
    }

    td.end-cell,
    td.actions {
      border-left: 1px solid transparent;
    }

    &:last-child {
      border-bottom-left-radius: 10px;
      border-bottom-right-radius: 10px;
      td:first-child {
        border-bottom-left-radius: 10px;
      }
      td:last-child {
        border-bottom-right-radius: 10px;
      }
    }
  }

  &.no-header {
    .datatable-row {
      &:first-child {
        td:first-child {
          border-top-left-radius: 10px;
        }
        td:last-child {
          border-top-right-radius: 10px;
        }
      }
    }
  }

  &.multi-section {
    .datatable-row {
      &:nth-child(2) {
        th:first-child {
          border-top-left-radius: 10px;
        }
        td:first-child {
          border-top-left-radius: 10px;
        }
        td:last-child {
          border-top-right-radius: 10px;
        }
      }
      &:last-child {
        th:first-child {
          border-bottom-left-radius: 10px;
        }
        td:last-child {
          border-bottom-right-radius: 10px;
        }
      }
    }
  }
}

.datatable-head {
  th {
    background-color: var(--background);
    border-bottom: 1px solid var(--border);
    padding: 0.5rem 0.75rem;
    color: var(--text-alt);
    font-size: 0.8rem;
    letter-spacing: 1px;
    text-transform: uppercase;
    top: 0;
    vertical-align: middle;
    z-index: 2;

    a {
      color: var(--text-alt);
    }

    span.dot {
      margin-right: 7px;
    }

    &:hover .header-icon {
      opacity: 1;
    }
  }

  .validation-cell,
  .hidden-validation-cell {
    padding: 0;
    background-color: var(--background);
  }

  .validation-content {
    padding: 0.5rem 0.75rem;
    height: 38px;
  }
  .hidden-validation-cell {
    .validation-content {
      padding: 0.5rem 0.3rem;
    }
    .datatable-dropdown {
      display: none;
    }
    .header-icon {
      margin: 0 auto;
    }

    &:hover {
      .department-dot {
        display: none;
      }
      .validation-content {
        padding-left: 2px;
      }
    }
  }

  .datatable-row-header {
    z-index: 4;
  }
  .header-icon {
    width: 15px;
    cursor: pointer;
    opacity: 0;
  }
}

.datatable-dropdown {
  flex-grow: 1;
}

.datatable-row {
  user-select: none;
  &,
  & .datatable-row-header {
    background-color: var(--background);
  }

  &:nth-child(even),
  &:nth-child(even) .datatable-row-header {
    background-color: var(--background-alt);
  }

  &:hover,
  &:hover .datatable-row-header {
    background-color: var(--background-hover);
  }

  &:last-child th,
  &:last-child td {
    border-bottom: 1px solid var(--border);
  }

  &.datatable-row--selectable {
    cursor: pointer;

    &:hover,
    &:hover .datatable-row-header {
      background: var(--background-selectable);
    }
  }

  &.datatable-row.selected {
    &,
    & .datatable-row-header,
    &:hover,
    &:hover .datatable-row-header--no-bd,
    &:hover .datatable-row-header {
      background-color: var(--background-selected);
    }
  }

  th,
  td {
    box-sizing: border-box;
    border-top: 1px solid var(--border);
    padding: 0.5rem 0.75rem;
    vertical-align: middle;
    color: var(--text);

    &:first-child {
      border-left: 1px solid var(--border);
    }

    &:last-child {
      border-right: 1px solid var(--border);
    }
  }
  td {
    white-space: normal;
  }
  .hidden-validation-cell {
    min-width: 30px;
    max-width: 30px;
    width: 30px;
    padding: 0.3rem;
  }

  .numeric-cell {
    min-width: 80px;
    max-width: 80px;
    width: 80px;
  }

  .input.stylehidden {
    border-color: transparent;
    background-color: transparent;
    box-shadow: none;
  }
}

.multi-section .datatable-row {
  &:nth-child(odd),
  &:nth-child(odd) .datatable-row-header {
    background-color: var(--background-alt);
  }

  &:nth-child(even),
  &:nth-child(even) .datatable-row-header {
    background-color: var(--background);
  }
}

.datatable-row-header {
  position: sticky;
  left: 0;
  z-index: 1;
  border-right: 1px solid rgba(var(--border-rgb), 0.5);

  &::after {
    content: '';
    display: block;
    position: absolute;
    left: calc(100% + 1px);
    top: 0;
    bottom: 0;
    width: 0.75rem;
    background: linear-gradient(
      90deg,
      rgba(var(--border-rgb), 0.4) 0%,
      rgba(var(--border-rgb), 0.3) 20%,
      rgba(var(--border-rgb), 0.2) 50%,
      rgba(var(--border-rgb), 0) 100%
    );
  }
  .datatable-type-header & {
    border-right: 0;

    &::after {
      display: none;
    }
  }
}

.datatable-row-header--nobd {
  border-right: 0;

  &::after {
    display: none;
  }
}

.datatable-type-header {
  border-bottom: 1px solid var(--background);

  th {
    padding: 1.5rem 0 0.5rem;

    span,
    div {
      display: inline-block;
      text-indent: 0.3rem;
      font-size: 1.1em;
      color: var(--text);
    }
  }
}

.datatable tbody:first-of-type .datatable-type-header th::before {
  display: none;
}

.resizable {
  th {
    .resizable-knob {
      position: absolute;
      right: 0;
      top: 0;
      bottom: 0;
      width: 0.3rem;
      background: transparent;
      cursor: col-resize;
    }
    &:hover .resizable-knob {
      background: var(--border);
    }
  }
}

.ellipsis {
  overflow: hidden;
  text-overflow: ellipsis;
}

.header-icon {
  min-width: 15px;
}

.flexrow {
  display: flex;
  align-items: center;
}

.flexrow-reverse {
  flex-direction: row-reverse;
}

.flexrow-item {
  margin-right: 1rem;

  &.no-margin {
    margin-right: 0;
  }

  &.mr02 {
    margin-right: 0.2em;
  }
}

.flexcolumn {
  display: flex;
  flex-direction: column;
}

.flexcolumn-item {
  width: 100%;
}

.flexrow-item:last-child {
  margin-right: 0;
}

.pointer {
  cursor: pointer;
}

.side {
}

.menu-mask {
  position: fixed;
  background: blue;
  z-index: 100;
  top: 0;
  left: 0;
  opacity: 0;
  overflow: hidden;
  background-color: #000;
  width: 100%;
  height: 100%;
}

.button:focus,
.button:active {
  border-color: #666;
}

.combobox {
  border-radius: 10px;
}

.thumbnail-picture {
  border: 1px solid #ccc;
}

.modal {
}

.modal-content {
  .button {
    margin-left: 0.5em;
  }

  h2 {
    margin-bottom: 0;
    border-bottom: 0;
  }
}

.modal-content label.button {
  margin-left: 0em;
}

.modal-content .box p.text {
  margin-bottom: 1em;
}

.modal-content p.is-danger {
  color: #ff3860;
  font-style: italic;
  margin-bottom: 2em;
}

.selectable {
  user-select: text;
}

.unselectable {
  user-select: none;
}

.columns {
  display: flex;
  flex-direction: row;
  padding: 0;
  margin: 0;
}

.columns:last-child {
  margin: 0;
}

.column {
  overflow-y: auto;
  padding: 0;
}

.main-column {
}

.playlist-column .video-player-box .video-js {
  margin: auto;
}

.tabs li.is-active a {
  border-color: #00b242;
  color: #00b242;
}

.page .columns:last-child {
  margin-bottom: 1em;
}

.side-column {
  background: white;
  margin-top: 60px;
  max-width: 400px;
  width: 400px;
}

.empty-list {
  margin-top: 2em;
  font-size: 1.5em;
}

.entity-thumbnail {
  border-radius: 0.5em;
  box-shadow: 0px 0px 6px var(--box-shadow);
  cursor: pointer;
  transition: transform ease 0.3s;
  max-width: 90px;

  &:hover {
    transform: scale(1.1);
  }
}

.entity-title {
  margin: 0;
  flex: 1;
  font-size: 2em;
  font-weight: 500;
  color: $grey;
}

th.validation-cell {
  &:hover {
    text-decoration: underline $light-grey;
  }
}

.tooltip {
  background: var(--background);
  border-radius: 1em;
  box-shadow: 0 0 2px 2px var(--box-shadow);
  color: var(--text);
  padding: 1em;
  z-index: 2;

  .popover-arrow {
    border-color: var(--background);
  }
}

.block {
  background: var(--background-block);
  border-radius: 1em;
  box-shadow: 0px 0px 6px var(--box-shadow);
  color: var(--text);
  padding: 1.5em;

  &.ready-for {
    color: var(--text);
    border-radius: 1em;
    padding: 1em;
  }
}

.asset .description-cell,
.shot .description-cell,
.description-cell .tooltip {
  h1 {
    font-weight: bold;
    font-size: 1.2em;
    margin-bottom: 1em;
    text-transform: uppercase;
  }
  strong {
    font-size: inherit;
  }
  p,
  ul {
    margin-bottom: 1em;
  }
}

.input.date-input {
  border-radius: 10px;
}


.date-input::placeholder {
  border-radius: 10px;
  color: $light-grey;
}

.project-dates .date-input {
  width: 150px;
}

.dark .button.is-on {
  box-shadow: inset 0 0 10px #111;
  border-color: #25282e;
}

.button.is-toggle {
  transition: box-shadow ease 0.3s;
}

.button.is-toggle:active,
.button.is-toggle:focus {
  border-color: #dbdbdb;
}

.dark .button.is-toggle:active,
.dark .button.is-toggle:focus {
  border-color: #25282e;
}

.break-word {
  overflow-wrap: break-word;
  hyphens: auto;
  word-wrap: anywhere;
}

.button.is-on {
  box-shadow: inset 0 0 4px #999;
}

.break-word {
  word-wrap: break-word;
  overflow-wrap: break-word;
  hyphens: auto;
}

.status-combo {
  padding: 0.3em;
  border-radius: 10px;

  .selected-status-line,
  .status-line {
    padding: 0.4em;
  }
}

.status-combo-wrapper {
  margin: 0;

  .status-combo {
    border-top-left-radius: 2em;
    border-bottom-left-radius: 2em;
  }

  .selected-status-line {
    padding: 0.1em;
    padding-left: 0.2em;
  }
}

.theme .datepicker input {
  width: 150px;

  &.short {
    width: 106px;
  }
}

.theme .datepicker .vdp-datepicker__calendar {
  z-index: 2000;

  .cell.year:not(.blank):not(.disabled):hover,
  .cell.month:not(.blank):not(.disabled):hover,
  .cell.day:not(.blank):not(.disabled):hover {
    background: var(--background-selectable);
    border: 1px solid transparent;
  }

  .cell.year.disabled:hover,
  .cell.month.disabled:hover,
  .cell.day.disabled:hover {
    border: 1px solid transparent;
  }

  .cell.year.selected,
  .cell.month.selected,
  .cell.day.selected {
    background: var(--background-selected);
  }

  .cell.year.selected:not(.blank):not(.disabled):hover,
  .cell.month.selected:not(.blank):not(.disabled):hover,
  .cell.day.selected:not(.blank):not(.disabled):hover {
    border: 1px solid transparent;
    background: var(--background-selected);
  }

  header span:not(.disabled):hover {
    background: var(--background-selectable);
  }
}

.theme .vue-slider-dot-tooltip-inner {
  background: $purple-strong;
  border-color: $purple-strong;
}

.dark .vue-slider-dot-tooltip-inner {
  background: $dark-purple;
  border-color: $dark-purple;
}

.c-mask {
  position: fixed;
  z-index: 204;
  top: 0;
  left: 0;
  overflow: hidden;
  width: 0;
  height: 0;
  background-color: #000;
  opacity: 0;
}

.c-mask.is-active {
  width: 100%;
  height: 100%;
}

.dark .close-button:hover {
  background: $dark-grey-lightest;
}

.close-button {
  cursor: pointer;
  display: inline-block;
  text-align: center;
  padding-top: 3px;
  width: 30px;
  height: 30px;
}

.close-button:hover {
  display: inline-block;
  background: $white-grey;
  border-radius: 50%;
}

.asset-text {
  background: var(--background-tag);
  color: var(--text);
  border-radius: 0.9em;
  cursor: default;
  font-size: 0.8em;
  margin-top: 0.1em;
  margin-bottom: 0.1em;
  padding: 0.2em 0.6em;

  .asset-text-name {
    margin-right: 0;
  }

  .modify-asset {
    background: var(--background-tag-button);
    border-radius: 0.9em;
    cursor: pointer;
    font-size: 0.7em;
    padding: 0.3em 0.5em;
    margin-left: 0.5em;
  }
}

.separator {
  margin: 0.5rem;
  &:before {
    content: '';
    border-left: 1px solid $dark-grey-lightest;
    height: 0.5rem;
  }
}

#model-viewer {
  width: 100%;
  height: 100%;
  background: black;
}

.notification .comment-text {
  p {
    margin-bottom: 1em;
    max-width: 500px;
  }
}

@media screen and (max-width: 1000px) {
  .button .icon.is-small {
    margin-right: 0;
  }
}

@media screen and (max-width: 768px) {
  .level-left + .level-right {
    display: flex;
    justify-content: left;
  }

  .level-item:not(:last-child) {
    margin-bottom: 0;
  }
}

@media (min-width: 500px) {
  .container {
    margin: 0 auto;
  }
}

@media (max-width: 500px) {
  .container {
    flex: 1;
    width: 100%;
    max-width: 100%;
    display: flex;
  }

  .box {
    margin: 0;
    width: 100%;
    min-width: 100%;
    flex: 1;
  }

  .hero {
    display: flex;
    flex-direction: column;
  }
}
</style>

<template>
  <div class="columns fixed-page">
    <div class="column main-column">
      <div class="notifications page" v-scroll="onBodyScroll" ref="body">
        <div class="flexrow">
          <page-title
            class="flexrow-item title"
            :text="$t('notifications.title')"
          />
          <combobox-task-type
            class="flexrow-item selector"
            :label="$t('news.task_type')"
            :task-type-list="taskTypeList"
            v-model="taskTypeId"
          />
          <combobox-status
            class="flexrow-item selector"
            :label="$t('news.task_status')"
            :task-status-list="taskStatusList"
            v-model="taskStatusId"
          />
          <combobox-styled
            class="flexrow-item selector field"
            :label="$t('main.type')"
            :options="typeOptions"
            v-model="typeMode"
          />
        </div>
        <div
          class="empty-list has-text-centered"
          v-if="
            !loading.notifications &&
            (!notifications || notifications.length === 0)
          "
        >
          {{ $t('notifications.no_notifications') }}
        </div>
        <div class="has-text-centered" v-if="loading.notifications">
          <spinner />
        </div>
        <div
          :class="{
            notification: true,
            unread: !notification.read,
            selected: notification.id === currentNotificationId
          }"
          :key="notification.id"
          @click="onNotificationSelected(notification)"
          v-for="notification in notifications"
          v-show="!loading.notifications"
        >
          <div class="flexrow notification-line">
            <div class="flexrow-item">
              <div class="flexrow">
                <at-sign-icon
                  class="icon flexrow-item"
                  v-if="isMention(notification)"
                />
                <message-square-icon
                  class="icon flexrow-item"
                  v-if="isComment(notification)"
                />
                <corner-up-left-icon
                  class="icon flexrow-item"
                  v-if="isReply(notification)"
                />
                <user-icon
                  class="icon flexrow-item"
                  v-if="isAssignation(notification)"
                />

                <task-type-name
                  class="task-type-name"
                  :task-type="buildTaskTypeFromNotification(notification)"
                  :production-id="notification.project_id"
                />
              </div>

              <div class="mt1 has-text-right">
                <validation-tag
                  class="validation-tag mt1"
                  :task="buildTaskFromNotification(notification)"
                  v-if="notification.change"
                />
              </div>

              <div class="date has-text-right">
                {{ formatDate(notification.created_at) }}
              </div>
            </div>

            <div class="flexrow-item comment-content">
              <div>
                <div class="notification-info flexrow">
                  <people-avatar
                    class="flexrow-item"
                    :person="personMap.get(notification.author_id)"
                    :size="30"
                    v-if="personMap.get(notification.author_id)"
                  />

                  <router-link
                    class="person-name flexrow-item"
                    :to="{
                      name: 'person',
                      params: { person_id: notification.author_id }
                    }"
                  >
                    {{ personName(notification) }}
                  </router-link>

                  <span
                    class="explaination flexrow-item"
                    v-if="isComment(notification)"
                  >
                    {{ $t('notifications.commented_on') }}
                  </span>

                  <span
                    class="explaination flexrow-item"
                    v-if="isReply(notification)"
                  >
                    {{ $t('notifications.replied_on') }}
                  </span>

                  <span
                    class="explaination flexrow-item"
                    v-if="isAssignation(notification)"
                  >
                    {{ $t('notifications.assigned_you') }}
                  </span>

                  <span
                    class="explaination flexrow-item"
                    v-if="isMention(notification)"
                  >
                    {{ $t('notifications.mention_you_on') }}
                  </span>

                  <router-link
                    class="flexrow-item"
                    :to="entityPath(notification)"
                  >
                    {{ notification.project_name }} /
                    {{ notification.full_entity_name }}
                  </router-link>
                </div>
              </div>
              <div
                class="comment-text"
                v-html="
                  renderComment(
                    notification.comment_text,
                    notification.mentions,
                    personMap
                  )
                "
                v-if="
                  isComment(notification) ||
                  isMention(notification) ||
                  notification.comment_text
                "
              ></div>
              <div v-if="isReply(notification)">...</div>
              <div v-if="notification.preview_file_id">
                <h3>
                  {{ $t('notifications.new_revision') }}
                </h3>
                <div
                  class="thumbnail-picture-wrapper"
                  v-if="notification.preview_file_id"
                >
                  <entity-thumbnail
                    :entity="{ preview_file_id: notification.preview_file_id }"
                    :height="40"
                  />
                </div>
              </div>
              <div
                class="comment-text"
                v-if="
                  (isComment(notification) || isMention(notification)) &&
                  !notification.comment_text
                "
              >
                {{ $t('comments.empty_text') }}
              </div>

              <div
                class="comment-text reply-text"
                v-html="renderComment(notification.reply_text, [], personMap)"
                v-if="isReply(notification)"
              ></div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div
      class="column side-column is-hidden-mobile hide-small-screen"
      v-if="currentTask"
    >
      <task-info :task="currentTask" :is-loading="loading.currentTask" />
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import {
  AtSignIcon,
  CornerUpLeftIcon,
  MessageSquareIcon,
  UserIcon
} from 'vue-feather-icons'
import { marked } from 'marked'
import moment from 'moment-timezone'
import { pluralizeEntityType } from '@/lib/path'
import { renderComment } from '@/lib/render'

import ComboboxTaskType from '@/components/widgets/ComboboxTaskType'
import ComboboxStatus from '@/components/widgets/ComboboxStatus'
import ComboboxStyled from '@/components/widgets/ComboboxStyled'
import EntityThumbnail from '@/components/widgets/EntityThumbnail'
import PageTitle from '@/components/widgets/PageTitle'
import PeopleAvatar from '@/components/widgets/PeopleAvatar'
import Spinner from '@/components/widgets/Spinner'
import TaskInfo from '@/components/sides/TaskInfo'
import TaskTypeName from '@/components/widgets/TaskTypeName'
import ValidationTag from '@/components/widgets/ValidationTag'

export default {
  name: 'notification-page',
  components: {
    AtSignIcon,
    ComboboxTaskType,
    ComboboxStatus,
    ComboboxStyled,
    CornerUpLeftIcon,
    EntityThumbnail,
    MessageSquareIcon,
    PeopleAvatar,
    PageTitle,
    Spinner,
    TaskInfo,
    TaskTypeName,
    UserIcon,
    ValidationTag
  },

  data() {
    return {
      loading: {
        more: false,
        notifications: false,
        currentTask: true
      },
      errors: {
        notifications: false
      },
      currentTask: null,
      currentNotificationId: null,
      taskTypeId: '',
      taskStatusId: '',
      typeMode: '',
      typeOptions: [
        {
          label: this.$t('notifications.all_types'),
          value: ''
        },
        {
          label: this.$t('notifications.only_comments'),
          value: 'comment'
        },
        {
          label: this.$t('notifications.only_mentions'),
          value: 'mention'
        },
        {
          label: this.$t('notifications.only_assignations'),
          value: 'assignation'
        },
        {
          label: this.$t('notifications.only_replies'),
          value: 'reply'
        }
      ]
    }
  },

  created() {
    this.clearNotifications()
  },

  mounted() {
    this.reloadData()
  },

  computed: {
    ...mapGetters([
      'notifications',
      'personMap',
      'taskStatus',
      'taskTypes',
      'taskTypeMap',
      'taskStatusMap',
      'personMap',
      'user'
    ]),

    taskStatusList() {
      return [
        {
          id: '',
          color: '#999',
          short_name: this.$t('news.all')
        }
      ].concat([...this.taskStatus])
    },

    taskTypeList() {
      return [
        {
          id: '',
          color: '#999',
          name: this.$t('news.all')
        }
      ].concat([...this.taskTypes])
    }
  },

  methods: {
    ...mapActions([
      'clearNotifications',
      'loadMoreNotifications',
      'loadNotification',
      'loadNotifications',
      'loadTask',
      'markAllNotificationsAsRead'
    ]),

    reloadData() {
      this.loading.notifications = true
      this.errors.notifications = false
      this.currentTask = null
      this.loadNotifications({
        task_status_id: this.taskStatusId,
        task_type_id: this.taskTypeId,
        type: this.typeMode
      })
        .then(() => {
          this.loading.notifications = false
        })
        .catch(err => {
          console.error(err)
          this.errors.notifications = true
        })
    },

    onBodyScroll(event, position) {
      const maxHeight =
        this.$refs.body.scrollHeight - this.$refs.body.offsetHeight
      if (maxHeight < position.scrollTop + 100) {
        this.loadFollowingNotifications()
      }
    },

    loadFollowingNotifications() {
      if (!this.loading.more && !this.loading.notifications) {
        this.loading.more = true
        this.loadMoreNotifications({
          task_status_id: this.taskStatusId,
          task_type_id: this.taskTypeId,
          type: this.typeMode
        }).then(() => {
          this.loading.more = false
        })
      }
    },

    entityPath(notification) {
      const taskType = this.taskTypeMap.get(notification.task_type_id)

      const route = {
        name: 'task',
        params: {
          production_id: notification.project_id,
          task_id: notification.task_id,
          type: pluralizeEntityType(taskType.for_entity)
        }
      }

      if (notification.episode_id) {
        route.name = 'episode-task'
        route.params.episode_id = notification.episode_id
      }
      return route
    },

    formatDate(date) {
      const utcDate = moment.tz(date, 'UTC')
      date = moment(utcDate.format()).fromNow()
      return date[0].toUpperCase() + date.slice(1)
    },

    compileMarkdown(input) {
      return marked.parse(input || '')
    },

    personName(notification) {
      const person = this.personMap.get(notification.author_id)
      return person ? person.full_name : ''
    },

    buildTaskFromNotification(notification) {
      return {
        id: notification.task_id,
        task_status_id: notification.task_status_id,
        task_type_id: notification.task_type_id,
        episode_id: notification.episode_id
      }
    },

    buildTaskTypeFromNotification(notification) {
      return {
        ...this.taskTypeMap.get(notification.task_type_id),
        episode_id: notification.episode_id
      }
    },

    onNotificationSelected(notification) {
      this.loading.currentTask = true
      this.loadTask({
        taskId: notification.task_id
      })
        .then(task => {
          this.loading.currentTask = false
          this.currentTask = task
          this.currentNotificationId = notification.id
        })
        .catch(console.error)
    },

    renderComment,
    isAssignation: notification => {
      return notification.notification_type === 'assignation'
    },
    isComment: notification => {
      return (
        !notification.notification_type ||
        notification.notification_type === 'comment'
      )
    },
    isMention: notification => notification.notification_type === 'mention',
    isReply: notification => notification.notification_type === 'reply'
  },

  socket: {
    events: {
      'notification:new'(eventData) {
        if (this.user.id === eventData.person_id) {
          const notificationId = eventData.notification_id
          this.loadNotification(notificationId).catch(console.error)
        }
      },

      'preview-file:add-file'(eventData) {
        const commentId = eventData.comment_id
        const previewId = eventData.preview_file_id
        this.$store.commit('NOTIFICATION_ADD_PREVIEW', {
          previewId,
          commentId
        })
      }
    }
  },

  watch: {
    taskTypeId() {
      this.reloadData()
    },

    taskStatusId() {
      this.reloadData()
    },

    typeMode() {
      this.reloadData()
    }
  },

  beforeDestroy() {
    this.markAllNotificationsAsRead()
  },

  metaInfo() {
    return {
      title: `${this.$t('notifications.title')} - Kitsu`
    }
  }
}
</script>

<style lang="scss" scoped>
.dark {
  .page {
    background: $dark-grey-light;
  }

  .notification {
    background: $dark-grey-lighter;
    color: $white-grey;
    box-shadow: 0px 1px 1px 1px $dark-grey;

    &.selected {
      border-left: 6px solid $dark-purple;
    }
  }

  .icon,
  .comment-text {
    color: $light-grey-light;
  }

  a {
    color: $light-grey-light;
  }
}

a {
  font-weight: bold;
  color: $grey-strong;
}

.flexrow-item:first-child {
  margin-right: 1em;
}

.notification {
  margin-bottom: 0.5em;
  align-items: flex-start;
  background: white;
  box-shadow: 0 0 4px $light-grey;
  cursor: pointer;
  border-left: 6px solid transparent;
  padding-left: 0.7em;
}

.unread {
  border-left: 6px solid $orange;
}

.person-name {
  margin-left: 0.5em;
  margin-right: 0.5em;
}

.comment-text {
  color: $grey-strong;
  margin-top: 1em;
}

.validation-tag {
  padding-top: 1em;
}

.notification-info {
  vertical-align: middle;
  margin-bottom: 1em;
}

.notification-info span,
.notification-info a {
  vertical-align: middle;
  display: inline-flex;
  align-items: center;
  line-height: 2.1em;
}

.date {
  font-size: 0.8em;
  margin-top: 0.5em;
  color: $grey;

  span {
    margin-left: 0;
  }
}

.thumbnail-picture-wrapper {
  margin-left: 0.5em;
}

.icon {
  width: 20px;
  margin-right: 0em;
  font-size: 0.8em;
  color: $grey;
}

.notification-line {
  align-items: start;
}

.comment-content {
  .flexrow-item {
    margin-right: 0.5em;
    margin-left: 0;
  }

  .person-name {
    margin-left: 0.5em;
  }
}

.selected {
  border-left: 6px solid $purple;
}

.columns {
  display: flex;
  flex-direction: row;
}

.column {
  overflow-y: auto;
  padding: 0;
}

.main-column {
  border-right: 3px solid $light-grey;
  flex: 1 1 auto;
  padding-top: 60px;
  background: $white-grey-light;
  overflow-y: hidden;
  height: 100%;
}

.notifications {
  background: $white-grey-light;
  width: 100%;
  padding: 2em;
  overflow-y: auto;
  height: 100%;
}

.side-column {
  width: 450px;
  min-width: 450px;
  max-width: 450px;
}

h3 {
  text-transform: uppercase;
  color: var(--text);
  margin-bottom: 0.5em;
}
</style>

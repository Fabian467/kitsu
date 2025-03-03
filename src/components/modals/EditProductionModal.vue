<template>
  <div
    :class="{
      modal: true,
      'is-active': active
    }"
  >
    <div class="modal-background" @click="$emit('cancel')"></div>

    <div class="modal-content">
      <div class="box">
        <h1 class="title" v-if="productionToEdit && productionToEdit.id">
          {{ $t('productions.edit_title') }} {{ productionToEdit.name }}
        </h1>
        <h1 class="title" v-else>
          {{ $t('productions.new_production') }}
        </h1>

        <form v-on:submit.prevent>
          <text-field
            ref="nameField"
            :label="$t('productions.fields.name')"
            v-model="form.name"
            @enter="runConfirmation"
            v-focus
          />
          <combobox-styled
            class="field"
            :label="$t('productions.fields.status')"
            :options="productionStatusOptions"
            localeKeyPrefix="productions.status."
            @enter="runConfirmation"
            v-model="form.project_status_id"
            v-if="productionToEdit && productionToEdit.id"
          />
          <combobox-styled
            class="field"
            :label="$t('productions.fields.style')"
            :options="productionStyleOptions"
            localeKeyPrefix="productions.style."
            v-model="form.production_style"
          />
          <text-field
            v-if="productionToEdit && productionToEdit.id"
            ref="fpsField"
            :label="$t('productions.fields.fps')"
            v-model="form.fps"
            @enter="runConfirmation"
            v-focus
          />
          <text-field
            v-if="productionToEdit && productionToEdit.id"
            ref="ratioField"
            :label="$t('productions.fields.ratio')"
            v-model="form.ratio"
            @enter="runConfirmation"
            v-focus
          />
          <text-field
            v-if="productionToEdit && productionToEdit.id"
            ref="resolutionField"
            :label="$t('productions.fields.resolution')"
            v-model="form.resolution"
            @enter="runConfirmation"
            v-focus
          />

          <div v-if="productionToEdit && productionToEdit.id">
            <label class="label">{{ $t('productions.picture') }}</label>
            <file-upload
              ref="fileField"
              :label="$t('main.csv.upload_file')"
              accept=".png,.jpg,.jpeg"
              @fileselected="onFileSelected"
            />
          </div>
        </form>

        <modal-footer
          :error-text="$t('productions.edit_error')"
          :is-error="isError"
          :is-loading="isLoading"
          @confirm="runConfirmation"
          @cancel="$emit('cancel')"
        />
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import { modalMixin } from '@/components/modals/base_modal'

import ComboboxStyled from '@/components/widgets/ComboboxStyled'
import ModalFooter from '@/components/modals/ModalFooter'
import FileUpload from '@/components/widgets/FileUpload'
import TextField from '@/components/widgets/TextField'
import {
  PRODUCTION_STYLE_OPTIONS,
  PRODUCTION_TYPE_OPTIONS
} from '@/lib/productions'

export default {
  name: 'edit-production-modal',
  mixins: [modalMixin],
  components: {
    ComboboxStyled,
    FileUpload,
    ModalFooter,
    TextField
  },

  props: {
    active: {
      type: Boolean,
      default: false
    },
    isError: {
      type: Boolean,
      default: false
    },
    isLoading: {
      type: Boolean,
      default: false
    },
    productionToEdit: {
      type: Object,
      default: () => {}
    }
  },

  data() {
    const data = {
      formData: null,
      productionStyleOptions: PRODUCTION_STYLE_OPTIONS,
      productionTypeOptions: PRODUCTION_TYPE_OPTIONS
    }

    if (this.productionToEdit && this.productionToEdit.id) {
      data.form = {
        name: this.productionToEdit.name,
        project_status_id: this.productionToEdit.project_status_id,
        fps: this.productionToEdit.fps,
        ratio: this.productionToEdit.ratio,
        resolution: this.productionToEdit.resolution,
        production_type: this.productionToEdit.production_type || 'short',
        production_style: this.productionToEdit.production_style || '3d'
      }
    } else {
      data.form = {
        name: '',
        project_status_id: this.productionStatus
          ? this.productionStatus[0].id
          : null,
        fps: '',
        ratio: '',
        resolution: '',
        production_type: 'short',
        production_style: '3d'
      }
    }

    return data
  },

  created() {
    this.resetForm()

    this.productionTypeOptions = PRODUCTION_TYPE_OPTIONS
  },

  mounted() {
    if (this.productionStatus.length > 0) {
      this.form.project_status_id = this.productionStatus[0].id
    }
  },

  computed: {
    ...mapGetters([
      'productions',
      'productionStatus',
      'productionStatusOptions'
    ])
  },

  methods: {
    ...mapActions([]),

    runConfirmation() {
      this.$emit('confirm', this.form)
    },

    onFileSelected(formData) {
      this.formData = formData
      this.$emit('fileselected', formData)
    },

    resetForm() {
      if (this.productionToEdit && this.productionToEdit.id) {
        this.form = {
          name: this.productionToEdit.name,
          project_status_id: this.productionToEdit.project_status_id,
          fps: this.productionToEdit.fps,
          ratio: this.productionToEdit.ratio,
          resolution: this.productionToEdit.resolution,
          production_type: this.productionToEdit.production_type || 'short'
        }
        this.form.project_status_id = null
        this.$nextTick(() => {
          this.form.project_status_id = this.productionToEdit.project_status_id
        })
      } else {
        this.form = {
          name: '',
          project_status_id: this.productionStatusOptions[0].value,
          fps: '',
          ratio: '',
          resolution: '',
          production_type: 'short'
        }
      }
    }
  },

  watch: {
    productionToEdit() {
      this.resetForm()
    },

    active() {
      if (this.active) {
        setTimeout(() => {
          this.$refs.nameField.focus()
          this.formData = null
          if (this.$refs.fileField) this.$refs.fileField.reset()
        }, 100)
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.box {
  padding-bottom: 1.5em;
}
</style>

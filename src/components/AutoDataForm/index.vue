<template>
  <DataForm ref="dataForm" v-loading="loading" :fields="totalFields" v-bind="$attrs" v-on="$listeners">
    <FormGroupHeader v-for="(group, i) in groups" :slot="'id:'+group.name" :key="'group-'+group.name" :title="group.title" :line="i != 0" />
  </DataForm>
</template>

<script>
import DataForm from '../DataForm'
import FormGroupHeader from '@/components/FormGroupHeader'
// import { optionUrlMeta } from '@/api/common'
import rules from '@/components/DataForm/rules'
import Select2 from '@/components/Select2'
export default {
  name: 'AutoDataForm',
  components: {
    DataForm,
    FormGroupHeader
  },
  props: {
    url: {
      type: String,
      required: true
    },
    method: {
      type: String,
      default: 'post'
    },
    fields: {
      type: Array,
      default: () => {
        return []
      }
    },
    fieldsMeta: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      meta: {},
      totalFields: [],
      loading: true,
      groups: []
    }
  },
  mounted() {
    this.optionUrlMeta()
  },
  methods: {
    optionUrlMeta() {
      this.$store.dispatch('common/getUrlMeta', { url: this.url }).then(data => {
        this.meta = data.actions[this.method.toUpperCase()] || {}
        this.generateColumns()
      }).catch(err => {
        this.$log.error(err)
      }).finally(() => {
        this.loading = false
      })
    },
    generateFieldByType(type, field, fieldMeta) {
      switch (type) {
        case 'choice':
          type = 'radio-group'
          if (!fieldMeta.read_only) {
            field.options = fieldMeta.choices.map(v => {
              return { label: v.display_name, value: v.value }
            })
          }
          break
        case 'datetime':
          type = 'date-picker'
          field.el = {
            type: 'datetime'
          }
          break
        case 'field':
          type = ''
          field.component = Select2
          if (fieldMeta.required) {
            field.el.clearable = false
          }
          break
        case 'string':
          type = 'input'
          if (!fieldMeta.max_length) {
            field.el.type = 'textarea'
            field.el.rows = 3
          }
          break
        default:
          type = 'input'
          break
      }
      if (type === 'radio-group') {
        if (!fieldMeta.read_only) {
          const options = fieldMeta.choices.map(v => {
            return { label: v.display_name, value: v.value }
          })
          if (options.length > 4) {
            type = 'select'
            field.el.filterable = true
          }
        }
      }
      field.type = type
      return field
    },
    generateFieldByName(name, field) {
      switch (name) {
        case 'email':
          field.el.type = 'email'
          break
        case 'password':
          field.el.type = 'password'
          break
        case 'comment':
          field.el.type = 'textarea'
          break
      }
      return field
    },
    generateFieldByOther(field, fieldMeta) {
      const filedRules = field.rules || []
      if (fieldMeta.required) {
        if (field.type === 'input') {
          filedRules.push(rules.Required)
        } else {
          filedRules.push(rules.RequiredChange)
        }
      }
      field.rules = filedRules
      return field
    },
    generateField(name) {
      let field = { id: name, prop: name, el: {}, attrs: {}}
      // const fieldMeta = this.meta[name] || this.meta['attrs']['children'][name] || {}
      const fieldMeta = this.meta[name] || ((this.meta['attrs']) ? (this.meta['attrs']['children'][name]) : {})
      field.label = fieldMeta.label
      field = this.generateFieldByType(fieldMeta.type, field, fieldMeta)
      field = this.generateFieldByName(name, field)
      field = this.generateFieldByOther(field, fieldMeta)
      field = Object.assign(field, this.fieldsMeta[name] || {})
      _.set(field, 'attrs.error', '')
      return field
    },
    generateFieldGroup(data) {
      const [groupTitle, fields] = data
      this.groups.push({
        id: groupTitle,
        title: groupTitle,
        name: fields[0]
      })
      return this.generateFields(fields)
    },
    generateFieldAttrs(name) {
      const fields = []
      Object.keys(this.meta[name]['children']).forEach((key, i) => {
        const filed = this.generateField(key)
        fields.push(filed)
      })
      return fields
    },
    generateFields(data) {
      let fields = []
      for (let field of data) {
        if (field instanceof Array) {
          const items = this.generateFieldGroup(field)
          fields = [...fields, ...items]
        } else if (field === 'attrs') {
          const items = this.generateFieldAttrs(field)
          fields = [...fields, ...items]
          // 修改title插入ID
          this.groups[this.groups.length - 1].name = items[0].id
        } else if (typeof field === 'string') {
          field = this.generateField(field)
          fields.push(field)
        } else if (field instanceof Object) {
          this.errors[field.prop] = ''
          _.set(field, 'attrs.error', '')
          fields.push(field)
        }
      }
      return fields
    },
    generateColumns() {
      this.totalFields = this.generateFields(this.fields)
      this.$log.debug('Total fields: ', this.totalFields)
    },
    setFieldError(name, error) {
      const field = this.totalFields.find((v) => v.prop === name)
      if (!field) {
        return
      }
      if (typeof error === 'object') {
        const str = error
        error = ''
        Object.keys(str).forEach(key => {
          error += `${parseInt(key) + 1}.${str[key][0]}  `
        })
      }
      // if (field.attrs.error === error) {
      //   error += '.'
      // }
      field.attrs.error = error
    }
  }
}
</script>

<style scoped>

</style>

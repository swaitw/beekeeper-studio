<template>
  <div class="home">
    <section class="subheader">
      <div class="small-wrap">
        <h1>SQL Table Builder</h1>

        <!-- Alert -->
        <tutorial-alert v-if="isHome" />
        <!-- Dialect with Actions -->
      </div>
    </section>
    <section>
      <div class="small-wrap">

        <!-- Schema Builder -->
        <schema-builder v-if="templateSchema.columns"
          :initialColumns="templateSchema.columns"
          :resetOnUpdate="true"
          @columnsChanged="columnsChanged"
        >
          <div class="table-header flex flex-middle">
            <div class="form-group expand flex flex-middle">
              <input type="text" v-model="schema.name" :placeholder="defaultName">
              <span class="input-icon"><i class="material-icons">edit</i></span>
            </div>
            <dialect-picker :confirm="schemaChanges > 0" :confirmMessage="confirmMessage"/>
          </div>
          <template>
          </template>
        </schema-builder>
        <highlighted-code :code="formattedSql" :dialect="highlightDialect">
          <h3>
            Generated SQL for {{dialectTitle}}
          </h3>
          <p class="dialect-warning">{{dialectWarning ? `*${dialectWarning}`: ''}}</p>
        </highlighted-code>
      </div>
    </section>
    <footer>
      <div class="small-wrap flex-col flex-middle">
        <small class="created-by">
          <span>Made by&nbsp;</span>
          <router-link to="https://beeekeeperstudio.io">Beekeeper Studio</router-link>
        </small>
      </div>
    </footer>
  </div>
</template>

<script lang="ts">

import Vue from 'vue';
import _ from 'lodash'
import { mapGetters, mapState } from 'vuex'
import { UserTemplate as users } from '../lib/templates/user'
import { DialectTitles, Schema, SchemaItem } from '@shared/lib/dialects/models'
import SchemaBuilder from '@shared/components/SchemaBuilder.vue'
import Formatter from 'sql-formatter'
import Knex from 'knex'
import { SqlGenerator } from '@shared/lib/sql/SqlGenerator';
import TutorialAlert from '@/components/TutorialAlert.vue';
import DialectPicker from '@/components/DialectPicker.vue'
import { Template } from '@/lib/templates/base';
import templates from '@/lib/templates';
import HighlightedCode from '@/components/HighlightedCode.vue';
interface Data {
  template: Template
  schema: Schema
  sql?: string,
  knex?: Knex,
  generator: SqlGenerator
  schemaChanges: number
  defaultName: string
}


const dialectNotes = {
  sqlite: "Sqlite does not support table or column comments",
}

export default Vue.extend ({
  name: 'Home',
  components: { 
    SchemaBuilder,
    TutorialAlert,
    DialectPicker,
    HighlightedCode
  },
  beforeRouteEnter(to, _from, next) {
    next((component: any) => {
      component.initialize(to.params.id)
      if (!component.template) return "/"
    })
  },
  beforeRouteUpdate(to, _from, next) {
    this.initialize(to.params.id)
    if (!this.template) next('/')
  },
  data(): Data {
    return {
      template: null,
      defaultName: 'untitled_table',
      schema: {
        name: 'untitled_table',
        columns: null
      },
      sql: undefined,
      generator: undefined,
      schemaChanges: 0
    }
  },
  watch: {
    dialect() {
      this.generator.dialect = this.dialect
      this.schemaChanges = 0
      this.schema.columns = this.templateSchema.columns
    },
    schema: {
      deep: true,
      handler() {
        this.generateSql()
      }
    },
  },
  computed: {
    ...mapState(['dialect']),
    ...mapGetters(['knexDialect']),
    isHome() {
      return this.$route.name === 'Home'
    },
    confirmMessage() {
      return `You will lose ${this.schemaChanges} changes, continue?`
    },
    templateSchema() {
      return this.template ? this.template.toSchema(this.dialect) : []
    },
    highlightDialect() {
      switch (this.dialect) {
        case 'postgresql':
        case 'redshift':
          return 'pgsql'
        default:
          return 'sql'
      }
    },
    dialectWarning() {
      return dialectNotes[this.dialect]
    },
    dialectTitle() {
      return DialectTitles[this.dialect]
    },
    formattedSql() {
      // TODO (map dialects)
      if (!this.sql) return null
      return Formatter.format(this.sql, { language: 'sql'})
    }
  },
  methods: {
    generateSql: _.debounce(function() {
      this.sql = this.generator.buildSql(this.schema)
    }, 300),
    initialize(id?: string) {
      this.template = id ? templates.find((t) => t.id === id) : users
      // if (!this.id) this.$router.replace({query: {}})
      if (!this.template) return
      this.schema = _.clone(this.templateSchema)
    },
    columnsChanged(columns: SchemaItem[]) {
      this.schema.columns = columns
      this.schemaChanges += 1
    }
  },
  mounted() {
      this.generator = new SqlGenerator(this.dialect)
  }
})
</script>

<style lang="scss" scoped>
  @import '@/assets/styles/app/_variables';

  section {
    padding: $gutter-w * 2;
  }
  .subheader {
    display: flex;
    flex-direction: column;
    background: rgba($query-editor-bg, 0.5);
    padding: $gutter-w * 2;
  }
  .alert {
    position: relative;
    margin-bottom: $gutter-w * 2;
    line-height: 1.8;
    .close-btn {
      position: absolute;
      top: $gutter-h;
      right: $gutter-h;
      &:hover {
        .material-icons {
          color: $text-dark
        }
      }
      .material-icons {
        color: $text-lighter;
        transition: color 0.2s ease-in-out;
      }
    }
  }

  // Table Header
  .table-header {
    margin: 0 (-$gutter-h * 1.25) ($gutter-w * 2);
    padding-right: $gutter-h * 1.25;
    .form-group {
      position: relative;
      width: 100%;
      padding: 0;
      &:hover {
        .input-icon {
          display: flex;
        }
      }
      input {
        font-size: 1.6rem;
        border: 0;
        height: auto;
        line-height: 1.6;
        font-weight: bold;
        padding-right: 24px;
        &:hover {
          background: rgba($theme-base, 0.035);
        }
        &:focus {
          background: transparent;
          box-shadow: inset 0 0 0 1px $border-color;
        }
      }
      .input-icon {
        position: absolute;
        top: 0;
        right: 0;
        bottom: 0;
        align-items: center;
        display: none;
        .material-icons {
          font-size: 22px;
          width: 24px;
          margin-right: $gutter-h;
          color: $text-lighter;
        }
      }
    }
    select {
      font-size: 1.1rem;
      line-height: 2.1;
      height: auto;
      margin-left: $gutter-w;
    }
  }
  .code-wrap {
    margin-top: $gutter-w * 4;
  }

  // Footer
  footer {
    padding: $gutter-w * 2;
    padding-top: $gutter-w * 4;
    padding-bottom: $gutter-w * 6;
    a {
      color: $theme-primary;
    }
    .created-by {
      display: flex;
      align-items: center;
    }
  }
</style>
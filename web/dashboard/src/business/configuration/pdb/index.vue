<template>
  <layout-content header="Pod Disruption Budget">
    <complex-table :selects.sync="selects" :data="data" @search="search" v-loading="loading" :pagination-config="paginationConfig"
                   :search-config="searchConfig">
      <template #header>
        <el-button type="primary" size="small" @click="onCreate"
                   v-has-permissions="{apiGroup:'policy',scope:'namespace',resource:'poddisruptionbudgets',verb:'create'}">
          YAML
        </el-button>
        <el-button type="primary" size="small" :disabled="selects.length===0" @click="onDelete()"
                   v-has-permissions="{apiGroup:'policy',scope:'namespace',resource:'poddisruptionbudgets',verb:'delete'}">
          {{ $t("commons.button.delete") }}
        </el-button>
      </template>
      <el-table-column type="selection" fix></el-table-column>
      <el-table-column :label="$t('commons.table.name')" prop="metadata.name" show-overflow-tooltip>
        <template v-slot:default="{row}">
          <span class="span-link" @click="openDetail(row)">{{ row.metadata.name }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('business.namespace.namespace')" prop="metadata.namespace">
        <template v-slot:default="{row}">
          {{ row.metadata.namespace }}
        </template>
      </el-table-column>
      <el-table-column :label="$t('commons.table.created_time')" prop="metadata.creationTimestamp" fix>
        <template v-slot:default="{row}">
          {{ row.metadata.creationTimestamp | age }}
        </template>
      </el-table-column>
      <ko-table-operations :buttons="buttons" :label="$t('commons.table.action')"></ko-table-operations>
    </complex-table>
  </layout-content>
</template>

<script>
import LayoutContent from "@/components/layout/LayoutContent"
import ComplexTable from "@/components/complex-table"
import {downloadYaml} from "@/utils/actions"
import KoTableOperations from "@/components/ko-table-operations"
import {checkPermissions} from "@/utils/permission"
import {deletePDB, getPDB, listPDBs} from "@/api/poddisruptionbudgets"

export default {
  name: "PDBs",
  components: {ComplexTable, LayoutContent, KoTableOperations},
  data() {
    return {
      data: [],
      selects: [],
      loading: false,
      cluster: "",
      buttons: [
        {
          label: this.$t("commons.button.edit_yaml"),
          icon: "el-icon-edit",
          click: (row) => {
            this.$router.push({
              name: "PDBEdit",
              params: {namespace: row.metadata.namespace, name: row.metadata.name}
            })
          },
          disabled: () => {
            return !checkPermissions({
              scope: "namespace",
              apiGroup: "policy",
              resource: "poddisruptionbudgets",
              verb: "update"
            })
          }
        },
        {
          label: this.$t("commons.button.download_yaml"),
          icon: "el-icon-download",
          click: (row) => {
            downloadYaml(row.metadata.name + ".yml", getPDB(this.cluster, row.metadata.namespace, row.metadata.name))
          }
        },
        {
          label: this.$t("commons.button.delete"),
          icon: "el-icon-delete",
          click: (row) => {
            this.onDelete(row)
          },
          disabled: () => {
            return !checkPermissions({
              scope: "namespace",
              apiGroup: "policy",
              resource: "poddisruptionbudgets",
              verb: "delete"
            })
          }
        },
      ],
      paginationConfig: {
        currentPage: 1,
        pageSize: 10,
        total: 0,
      },
      searchConfig: {
        keywords: ""
      }
    }
  },
  methods: {
    search(resetPage) {
      this.loading = true
      if (resetPage) {
        this.paginationConfig.currentPage = 1
      }
      listPDBs(this.cluster, true, this.searchConfig.keywords, this.paginationConfig.currentPage, this.paginationConfig.pageSize).then(res => {
        this.data = res.items
        this.paginationConfig.total = res.total
        this.loading = false
      })
    },
    onCreate() {
      this.$router.push({
        name: "PDBCreateYaml",
        query: {
          type:"poddisruptionbudgets"
        }
      })
    },
    onDelete(row) {
      this.$confirm(
          this.$t("commons.confirm_message.delete"),
          this.$t("commons.message_box.prompt"), {
            confirmButtonText: this.$t("commons.button.confirm"),
            cancelButtonText: this.$t("commons.button.cancel"),
            type: "warning",
          }).then(() => {
        this.ps = []
        if (row) {
          this.ps.push(deletePDB(this.cluster, row.metadata.namespace, row.metadata.name))
        } else {
          if (this.selects.length > 0) {
            for (const select of this.selects) {
              this.ps.push(deletePDB(this.cluster, select.metadata.namespace, select.metadata.name))
            }
          }
        }
        if (this.ps.length !== 0) {
          Promise.all(this.ps)
              .then(() => {
                this.search(true)
                this.$message({
                  type: "success",
                  message: this.$t("commons.msg.delete_success"),
                })
              })
              .catch(() => {
                this.search(true)
              })
        }
      })
    },
    openDetail(row) {
      this.$router.push({
        name: "PDBDetail",
        params: {namespace: row.metadata.namespace, name: row.metadata.name}
      })
    }
  },
  created() {
    this.cluster = this.$route.query.cluster
    this.search()
  }
}
</script>

<style scoped>

</style>

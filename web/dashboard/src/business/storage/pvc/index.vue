<template>
  <layout-content header="Persistent Volume Claims">
    <complex-table :data="data" :selects.sync="selects" @search="search" v-loading="loading"
                   :pagination-config="paginationConfig" :search-config="searchConfig">
      <template #header>
        <el-button v-has-permissions="{scope:'namespace',apiGroup:'',resource:'persistentvolumeclaims',verb:'create'}"
                   type="primary" size="small"
                   @click="yamlCreate">
          YAML
        </el-button>
        <el-button type="primary" size="small"
                   v-has-permissions="{scope:'namespace',apiGroup:'',resource:'persistentvolumeclaims',verb:'create'}"
                   @click="onCreate">
          {{ $t("commons.button.create") }}
        </el-button>
        <el-button type="primary" size="small"
                   v-has-permissions="{scope:'namespace',apiGroup:'',resource:'persistentvolumeclaims',verb:'delete'}"
                   :disabled="selects.length===0" @click="onDelete()">
          {{ $t("commons.button.delete") }}
        </el-button>
      </template>
      <el-table-column type="selection" fix></el-table-column>
      <el-table-column :label="$t('commons.table.name')" prop="metadata.name" show-overflow-tooltip>
        <template v-slot:default="{row}">
          <span class="span-link" @click="openDetail(row)">{{ row.metadata.name }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('commons.table.status')" prop="status.phase">
        <template v-slot:default="{row}">
          <el-button v-if="row.status.phase && row.status.phase === 'Bound'" type="success" size="mini" plain round>
            {{ row.status.phase }}
          </el-button>
          <el-button v-if="row.status.phase && row.status.phase === 'Available'" type="success" size="mini" plain round>
            {{ row.status.phase }}
          </el-button>
          <el-button v-if="row.status.phase && row.status.phase === 'Pending'" type="warning" size="mini" plain round>
            {{ row.status.phase }}
          </el-button>
        </template>
      </el-table-column>
      <el-table-column show-overflow-tooltip :label="$t('business.namespace.namespace')" prop="metadata.namespace"/>
      <el-table-column show-overflow-tooltip label="Volume" prop="spec.volumeName">
        <template v-slot:default="{row}">
          <span class="span-link" @click="openPvDetail(row)">{{ row.spec.volumeName }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('business.storage.capacity')" prop="spec.resources.requests.storage"/>
      <el-table-column :label="$t('business.storage.storageClass')" prop="spec.storageClassName"/>
      <el-table-column label="volumeMode" prop="spec.volumeMode"/>
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
import ComplexTable from "@/components/complex-table/index"
import {downloadYaml} from "@/utils/actions"
import KoTableOperations from "@/components/ko-table-operations"
import {deletePvcs, getPvc, listPvcs} from "@/api/pvc"
import {checkPermissions} from "@/utils/permission"

export default {
  name: "PersistentVolumeClaim",
  components: { ComplexTable, LayoutContent, KoTableOperations },
  data () {
    return {
      data: [],
      selects: [],
      cluster: "",
      loading: false,
      buttons: [
        {
          label: this.$t("commons.button.edit"),
          icon: "el-icon-edit",
          click: (row) => {
            this.$router.push({
              name: "PersistentVolumeClaimEdit",
              params: { name: row.metadata.name, namespace: row.metadata.namespace },
              query: { yamlShow: true },
            })
          },
          disabled: () => {
            return !checkPermissions({
              scope: "namespace",
              apiGroup: "",
              resource: "persistentvolumeclaims",
              verb: "update"
            })
          },
        },
        {
          label: this.$t("commons.button.download_yaml"),
          icon: "el-icon-download",
          click: (row) => {
            downloadYaml(row.metadata.name + ".yml",getPvc(this.cluster,row.metadata.namespace,row.metadata.name))
          },
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
              apiGroup: "",
              resource: "persistentvolumeclaims",
              verb: "delete"
            })
          },
        },
      ],
      paginationConfig: {
        currentPage: 1,
        pageSize: 10,
        total: 0,
      },
      searchConfig: {
        keywords: "",
      },
    }
  },
  methods: {
    search () {
      this.loading = true
      const { currentPage, pageSize } = this.paginationConfig
      listPvcs(this.cluster, true, this.searchConfig.keywords, currentPage, pageSize).then((res) => {
        this.data = res.items
        this.loading = false
        this.paginationConfig.total = res.total
      })
    },
    onCreate () {
      this.$router.push({
        name: "PersistentVolumeClaimCreate",
        query: { yamlShow: false },
      })
    },
    yamlCreate () {
      this.$router.push({
        name: "PersistentVolumeClaimCreateYaml",
        query: { type: "persistentvolumeclaims" },
      })
    },
    onDelete (row) {
      this.$confirm(this.$t("commons.confirm_message.delete"), this.$t("commons.message_box.prompt"), {
        confirmButtonText: this.$t("commons.button.confirm"),
        cancelButtonText: this.$t("commons.button.cancel"),
        type: "warning",
      }).then(() => {
        this.ps = []
        if (row) {
          this.ps.push(deletePvcs(this.cluster, row.metadata.namespace, row.metadata.name))
        } else {
          if (this.selects.length > 0) {
            for (const select of this.selects) {
              this.ps.push(deletePvcs(this.cluster, select.metadata.namespace, select.metadata.name))
            }
          }
        }
        if (this.ps.length !== 0) {
          Promise.all(this.ps)
            .then(() => {
              this.search()
              this.$message({
                type: "success",
                message: this.$t("commons.msg.delete_success"),
              })
            })
            .catch(() => {
              this.search()
            })
        }
      })
    },
    openDetail (row) {
      this.$router.push({
        name: "PersistentVolumeClaimDetail",
        params: {
          name: row.metadata.name,
          namespace: row.metadata.namespace,
        },
        query: { yamlShow: false },
      })
    },
    openPvDetail (row) {
      this.$router.push({
        name: "PersistentVolumeDetail",
        params: {
          name: row.spec.volumeName,
        },
        query: { yamlShow: false },
      })
    },
  },
  created () {
    this.cluster = this.$route.query.cluster
    this.search()
  },
}
</script>

<style scoped>
</style>

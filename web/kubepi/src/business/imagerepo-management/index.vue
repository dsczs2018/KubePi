<template>
  <layout-content :header="$t('business.image_repos.list')">
    <complex-table :search-config="searchConfig" :selects.sync="selects" :data="data"
                   :pagination-config="paginationConfig" @search="search">
      <template #header>
        <el-button-group>
          <el-button v-has-permissions="{resource:'imagerepos',verb:'create'}" type="primary" size="small"
                     @click="onCreate">
            {{ $t("commons.button.create") }}
          </el-button>
        </el-button-group>
      </template>
      <el-table-column :label="$t('commons.table.name')" prop="name" min-width="100" fix>
        <template v-slot:default="{row}">
          <el-link @click="onDetail(row.name)">{{ row.name }}</el-link>
        </template>
      </el-table-column>
      <el-table-column :label="$t('business.image_repos.type')" prop="type" min-width="100" fix>
      </el-table-column>
      <el-table-column :label="$t('business.image_repos.endpoint')" prop="endPoint" min-width="100" fix>
      </el-table-column>
      <el-table-column :label="$t('business.image_repos.downloadUrl')" prop="downloadUrl" min-width="100" fix>
      </el-table-column>
      <el-table-column :label="$t('business.image_repos.repo')" prop="repoName" min-width="100" fix>
      </el-table-column>
      <el-table-column :label="$t('commons.table.age')" min-width="80" fix>
        <template v-slot:default="{row}">
          {{ row.createAt | ageFormat }}
        </template>
      </el-table-column>
      <fu-table-operations :buttons="buttons" :label="$t('commons.table.action')"/>
    </complex-table>
  </layout-content>
</template>

<script>
import LayoutContent from "@/components/layout/LayoutContent"
import ComplexTable from "@/components/complex-table"
import {deleteRepo, searchRepos} from "@/api/imagerepos"
import {checkPermissions} from "@/utils/permission"

export default {
  name: "ImageRepo",
  components: { ComplexTable, LayoutContent },
  props: {},
  data () {
    return {
      buttons: [
        {
          label: this.$t("commons.button.edit"),
          icon: "el-icon-edit",
          click: (row) => {
            this.$router.push({
              path: `/imagerepos/${row.name}/edit`,
              query: {mode: "edit" }
            })
          },
          disabled: () => {
            return !checkPermissions({ resource: "imagerepos", verb: "update" })
          }
        },
        {
          label: this.$t("commons.button.delete"),
          icon: "el-icon-delete",
          click: (row) => {
            this.onDelete(row.name)
          },
          disabled: () => {
            return !checkPermissions({ resource: "imagerepos", verb: "delete" })
          }
        },
      ],
      searchConfig: {
        quickPlaceholder: this.$t("commons.search.quickSearch"),
        components: [
          { field: "name", label: this.$t("commons.table.name"), component: "FuComplexInput", defaultOperator: "eq" },
        ]
      },
      selects: [],
      data: [],
      paginationConfig: {
        currentPage: 1,
        pageSize: 10,
        total: 0,
      },
      isSubmitGoing: false
    }
  },
  methods: {
    search (conditions) {
      this.loading = true
      const { currentPage, pageSize } = this.paginationConfig
      searchRepos(currentPage, pageSize, conditions).then(data => {
        this.loading = false
        this.data = data.data.items
        this.paginationConfig.total = data.data.total
      })
    },
    onCreate () {
      this.$router.push({
        path: "/imagerepos/create",
        query: { mode: "create" }
      })
    },
    onDelete (name) {
      if (this.isSubmitGoing) {
        return
      }
      this.isSubmitGoing = true
      this.$confirm(this.$t("commons.confirm_message.delete"), this.$t("commons.message_box.alert"), {
        confirmButtonText: this.$t("commons.button.confirm"),
        cancelButtonText: this.$t("commons.button.cancel"),
        type: "warning"
      }).then(() => {
        deleteRepo(name).then(() => {
          this.$message({
            type: "success",
            message: this.$t("commons.msg.delete_success"),
          })
          this.search()
        }).finally(() => {
          this.isSubmitGoing = false
        })
      })
    },
    onDetail(repo) {
      this.$router.push({
        name: "ImageRepoDetail",
        params: { repo: repo }
      })
    }
  },
  created () {
    this.search()
  }
}
</script>

<style scoped>

</style>

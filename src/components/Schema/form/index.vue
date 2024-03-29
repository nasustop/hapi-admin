<template>
  <div v-loading="loading" class="app-container">
    <el-form
      ref="ruleForm"
      :model="ruleForm"
      :rules="rules"
      :label-position="formSchemaData.labelPosition !== undefined ? formSchemaData.labelPosition : 'right'"
      :label-width="formSchemaData.labelWidth !== undefined ? formSchemaData.labelWidth : '70px'"
      :style="formSchemaData.style !== undefined ? formSchemaData.style : 'width: 80%;margin: auto'"
    >
      <el-form-item
        v-for="(item, index) in formSchemaData.columns"
        v-show="item.show !== undefined ? item.show : true"
        :key="index"
        :label="item.title"
        :prop="index"
        :style="item.style !== undefined ? item.style : 'width:70%;margin: auto auto 22px auto;'"
      >
        <div v-if="item.type === 'avatar'" class="avatar-uploader" @click="showImagecropper(index)">
          <div class="el-upload el-upload--text">
            <img v-if="ruleForm[index]" :src="ruleForm[index]" class="avatar" alt="">
            <i v-else class="el-icon-plus avatar-uploader-icon" />
          </div>
        </div>
        <template v-else-if="item.type === 'image'">
          <schema-dialog-upload-img
            v-if="item.num && item.num > 1"
            :num="item.num"
            :dialog-rel-data="ruleForm[index]"
            @check-change="(val) => dialogUploadImgChange(index, val)"
          />
          <schema-dialog-upload-img
            v-else
            :dialog-rel-img="ruleForm[index]"
            @check-change="(val) => dialogUploadImgChange(index, val)"
          />
        </template>
        <el-tree
          v-else-if="item.type === 'tree'"
          :ref="getTreeRef(index, item)"
          :check-strictly="item.checkStrictly !== undefined ? item.checkStrictly : false"
          :data="item.data !== undefined ? item.data : []"
          :props="item.props !== undefined ? item.props : { children: 'children', label: 'label' }"
          show-checkbox
          :node-key="item.dataKey !== undefined ? item.dataKey : index"
          class="permission-tree"
          @check-change="(data, checked, indeterminate) => treeCheckChange(data, checked, indeterminate, index, item['dataKey'], item.ref !== undefined ? item.ref : (index + 'TreeRef'))"
        />
        <el-switch
          v-else-if="item.type === 'switch'"
          v-model="ruleForm[index]"
          :active-color="item.activeColor !== undefined ? item.activeColor : '#13ce66'"
          :inactive-color="item.inactiveColor !== undefined ? item.inactiveColor : '#ff4949'"
          :active-text="item.activeText !== undefined ? item.activeText : ''"
          :inactive-text="item.inactiveText !== undefined ? item.inactiveText : ''"
        />
        <template v-else-if="item.type === 'span'">
          {{ item.value }}
          <el-input
            v-show="false"
            v-model="ruleForm[index]"
          />
        </template>
        <schema-dialog-table
          v-else-if="item.type === 'dialog'"
          :dialog-schema="item"
          :dialog-rel-data="ruleForm[index]"
          @check-change="(val) => dialogTableChange(index, val)"
        />
        <el-input
          v-else
          v-model="ruleForm[index]"
          :type="item.type"
          :show-password="item.showPassword !== undefined ? item.showPassword : false"
          :placeholder="item.placeholder"
          :clearable="item.clearable"
          :disabled="item.disabled"
          :prefix-icon="item.prefixIcon"
          :suffix-icon="item.suffixIcon"
          :readonly="item.readonly"
          :max="item.max"
          :min="item.min"
        />
      </el-form-item>
    </el-form>
    <div slot="footer" style="text-align: center;">
      <el-button
        v-for="(item, index) in actionsSchemaData"
        :key="index"
        :type="item.type !== undefined ? item.type : ''"
        @click="btnClick(item)"
      >
        {{ item.title }}
      </el-button>
    </div>
    <image-cropper
      v-show="imagecropperShow"
      :key="imagecropperKey"
      :url="uploadUrl"
      :width="300"
      :height="300"
      field="upload"
      lang-type="zh"
      @close="close"
      @crop-upload-success="cropSuccess"
    />
  </div>
</template>

<script>
import { apiCreateUploadsImage } from '@/api/system/uploads'
import ImageCropper from '@/components/ImageCropper'
import request from '@/utils/request'
import SchemaDialogTable from '@/components/Schema/dialog/table'
import SchemaDialogUploadImg from '@/components/Schema/dialog/uploadImg'

export default {
  name: 'SchemaFormIndex',
  components: { SchemaDialogUploadImg, SchemaDialogTable, ImageCropper },
  props: {
    schema: {
      type: Object,
      default() {
        return {}
      }
    }
  },
  data() {
    return {
      loading: false,
      schemaData: {},
      formSchemaData: {},
      actionsSchemaData: {},
      ruleForm: {},
      rules: {},
      treeRef: [],
      imagecropperShow: false,
      imagecropperIndex: '',
      imagecropperKey: 0,
      // dialog
      dialogSchema: {},
      dialogVisible: false
    }
  },
  computed: {
    uploadUrl() {
      return process.env.VUE_APP_BASE_API + apiCreateUploadsImage
    }
  },
  mounted() {
    this.setSchema(this.schema)
  },
  methods: {
    setSchema(data) {
      if (data.form) {
        this.formSchemaData = data.form
      }
      if (data.ruleForm) {
        this.ruleForm = data.ruleForm
      }
      if (data.rules) {
        this.rules = data.rules
        for (const field in this.rules) {
          for (const key in this.rules[field]) {
            if (this.rules[field][key].validator) {
              // eslint-disable-next-line no-eval
              this.rules[field][key].validator = eval(this.rules[field][key].validator)
            }
          }
        }
      }
      if (data.actions) {
        this.actionsSchemaData = data.actions
      }
      if (data.form.uri) {
        const requestData = {
          url: data.form.uri.const,
          method: 'get'
        }
        if (data.form.uri.method) {
          requestData.method = data.form.uri.method.toLowerCase()
        }
        switch (requestData.method) {
          case 'get':
            requestData.params = this.$route.query
            break
          case 'post':
            requestData.data = this.$route.query
            break
        }
        request(requestData).then(res => {
          for (const item in this.ruleForm) {
            if (res.data[item]) {
              this.ruleForm[item] = res.data[item]
            }
          }
          for (const index in this.treeRef) {
            this.$refs[this.treeRef[index]][0].setCheckedKeys(this.ruleForm[index])
          }
        })
      }
    },
    treeCheckChange(data, checked, indeterminate, index, key, refs) {
      const nodes = this.$refs[refs][0].getCheckedNodes()
      const node_ids = []
      for (const node of nodes) {
        node_ids.push(node[key])
      }
      this.ruleForm[index] = node_ids
    },
    dialogTableChange(index, val) {
      this.ruleForm[index] = val
    },
    dialogUploadImgChange(index, val) {
      this.ruleForm[index] = val
    },
    btnClick(item) {
      switch (item.action) {
        case 'rollback':
          this.$router.go(-1)
          break
        case 'reset':
          this.$refs['ruleForm'].resetFields()
          break
        case 'request':
          this.submitForm(item)
          break
      }
    },
    submitForm(item) {
      this.$refs['ruleForm'].validate((valid) => {
        if (!valid) {
          return false
        }
        this.loading = true
        const requestData = {
          url: item.url.const,
          method: 'post'
        }
        if (item.url.method) {
          requestData.method = item.url.method.toLowerCase()
        }
        switch (requestData.method) {
          case 'get':
            if (item.url['hasFilter']) {
              requestData.params = {
                filter: this.$route.query,
                params: this.ruleForm
              }
            } else {
              requestData.params = this.ruleForm
            }
            break
          case 'post':
            if (item.url['hasFilter']) {
              requestData.data = {
                filter: this.$route.query,
                params: this.ruleForm
              }
            } else {
              requestData.data = this.ruleForm
            }
            break
        }
        request(requestData).then(res => {
          if (res['code'] === 0 && item.url['notice'].success) {
            this.$notify({
              title: item.url['notice'].success.title ? item.url['notice'].success.title : 'error',
              message: item.url['notice'].success.message ? item.url['notice'].success.message : res['msg'],
              type: item.url['notice'].success.type ? item.url['notice'].success.type : 'success',
              duration: item.url['notice'].success.duration ? item.url['notice'].success.duration : 2000,
              onClose: () => {
                this.loading = false
                if (item.url['notice'].success.rollback !== false) {
                  this.$router.go(-1)
                }
              }
            })
          } else if (res['code'] !== 0 && item.url['notice'].error) {
            this.$notify({
              title: item.url['notice'].error.title ? item.url['notice'].error.title : 'error',
              message: item.url['notice'].error.message ? item.url['notice'].error.message : res['msg'],
              type: item.url['notice'].error.type ? item.url['notice'].error.type : 'error',
              duration: item.url['notice'].error.duration ? item.url['notice'].error.duration : 2000,
              onClose: () => {
                this.loading = false
                if (item.url['notice'].error.rollback !== false) {
                  this.$router.go(-1)
                }
              }
            })
          } else {
            this.loading = false
          }
        })
      })
    },
    getTreeRef(index, item) {
      const ref = item.ref ? item.ref : (index + 'TreeRef')
      this.treeRef[index] = ref
      return ref
    },
    showImagecropper(item) {
      this.imagecropperShow = true
      this.imagecropperIndex = item
    },
    cropSuccess(resData) {
      this.imagecropperShow = false
      this.imagecropperKey = this.imagecropperKey + 1
      this.ruleForm[this.imagecropperIndex] = resData['full_url']
    },
    close() {
      this.imagecropperShow = false
    }
  }
}
</script>

<style scoped>

</style>

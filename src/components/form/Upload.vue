<template>
  <div>
    <el-upload v-if="multiple"
               action=""
               list-type="picture-card"
               :on-preview="handlePictureCardPreview"
               :on-remove="handleRemove"
               ref="multiUpload"
               :file-list="fileList"
               :http-request="handleUpload"
    >
      <i class="el-icon-plus"></i>
    </el-upload>
    <el-upload ref="singleUpload" v-else
               action=""
               :style="avatarStyle"
               class="logo-uploader"
               :show-file-list="false"
               :http-request="handleUpload"
    >
      <div @mouseover="showBtn=true" @mouseout="showBtn=false">
        <i @click.stop="removeSingle" v-show="dialogImageUrl && showBtn" class="el-icon-close remove-btn"></i>
        <img v-if="dialogImageUrl" :src="dialogImageUrl" :style="avatarStyle">
        <i v-else class="el-icon-plus logo-uploader-icon" :style="avatarStyle"></i>
      </div>
    </el-upload>
    <v-dialog v-model="show" max-width="500">
      <img width="500px" :src="dialogImageUrl" alt="">
    </v-dialog>
  </div>
</template>

<script>
  import {Upload} from 'element-ui';
  import fileUtil from './fileUtil';


  export default {
    name: "vUpload",
    components: {
      elUpload: Upload
    },
    props: {
      url: {
        type: String,
        require: false
      },
      value: {},
      multiple: {
        type: Boolean,
        default: true
      },
      picWidth: {
        type: Number,
        default: 150
      },
      picHeight: {
        type: Number,
        default: 150
      },
      needSignature: {
        type: Boolean,
        default: false
      }
    },
    data() {
      return {
        showBtn: false,
        show: false,
        dialogImageUrl: "",
        avatarStyle: {
          width: this.picWidth + 'px',
          height: this.picHeight + 'px',
          'line-height': this.picHeight + 'px'
        },
        fileList: [],
        resp: {expire: 0}
      }
    },
    mounted() {
      if (!this.value || this.value.length <= 0) {
        return;
      }
      if (this.multiple) {
        this.fileList = this.value.map(f => {
          return {response: f, url: f}
        });
      } else {
        this.dialogImageUrl = this.value;
      }
    },
    watch: {
      value: {
        deep: true,
        handler(val) {
          if ((typeof val) === "string") {
            this.dialogImageUrl = val;
          }
        }
      }
    },
    methods: {
      async handleUpload(params) {
        const file = params.file;
        const fileType = file.type;
        const isImage = fileType.indexOf('image') !== -1;

        const isLt2M = file.size < 2 * 1024 * 1024;

        if (!isLt2M) {
          this.$message.error('??????????????????????????????????????? 2MB!');
          return;
        }

        if (!isImage) {
          this.$message.error('???????????????!');
          return;
        }
        // ??????????????????
        let filename = file.name;
        let uploadUrl = this.url;
        // ??????????????????
        const formData = new FormData();
        // ??????????????????
        if (this.needSignature) {
          // ?????????????????????????????????????????????????????????????????????
          if (this.resp.expire < new Date().getTime()) {
            // ????????????????????????????????????????????????url????????????????????????
            this.resp = await this.$http.loadData(this.url);
          }
          // ?????????????????????????????????
          filename = fileUtil.getFileName(filename, this.resp.dir);
          // ????????????????????????
          uploadUrl = this.resp.host;
          // ??????????????????
          formData.append("key", filename);
          formData.append("policy", this.resp.policy);
          formData.append("OSSAccessKeyId", this.resp.accessId);
          formData.append("success_action_status", '200');
          formData.append("signature", this.resp.signature);
        }
        formData.append("file", file);
        const that = this;
        this.$http.post(uploadUrl, formData, {headers: {'Content-Type': 'multipart/form-data', 'x': 'y'}})
          .then(resp => {
            console.log(resp);
            if (resp.status < '300') {
              // ??????????????????
              const fileUrl = resp.data || uploadUrl + "/" + filename;
              // ?????????????????????????????????
              if (!this.multiple) {
                // ???????????????????????????????????????
                this.dialogImageUrl = fileUrl;
                this.$emit("input", fileUrl)
              } else {
                // ???????????????????????????
                const files = this.$refs.multiUpload.uploadFiles;
                files[files.length - 1].response = fileUrl;
                this.fileList = files;
                this.$emit("input", files.map(f => f.response))
              }
            } else {
              that.$message.error('????????????');
            }
          })
          .catch(function (err) {
            this.$message.error('????????????');
            console.log(err);
          });
      },
      handleRemove(file, fileList) {
        this.fileList = fileList;
        this.$emit("input", fileList.map(f => f.response))
      },
      handlePictureCardPreview(file) {
        this.dialogImageUrl = file.response;
        this.show = true;
      },
      removeSingle() {
        this.dialogImageUrl = "";
        this.$refs.singleUpload.clearFiles();
      }
    }
  }
</script>

<style scoped>
  .logo-uploader {
    border: 1px dashed #d9d9d9;
    border-radius: 6px;
    cursor: pointer;
    position: relative;
    overflow: hidden;
    float: left;
  }

  .logo-uploader:hover {
    border-color: #409EFF;
  }

  .logo-uploader-icon {
    font-size: 28px;
    color: #8c939d;
    text-align: center;
  }

  .remove-btn {
    position: absolute;
    right: 0;
    font-size: 16px;
  }

  .remove-btn:hover {
    color: #c22;
  }
</style>

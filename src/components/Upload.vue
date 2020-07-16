<template>
  <div>
    <el-upload
      class="upload-demo"
      ref="upload"
      :action="action"
      drag
      multiple
      :on-preview="handlePreview"
      :on-remove="handleRemove"
      :file-list="fileList"
      :auto-upload="false"
      :data="authData"
      :before-upload="handleBeforeUpload"
      :http-request="handleUploadFile"
    >
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">
        将文件拖到此处，或
        <em>点击上传</em>
      </div>
      <div class="el-upload__tip" slot="tip">
        只能上传jpg/png文件，且不超过500kb
        <el-row style="margin-top: 15px">
          <el-button
            style="margin-left: 10px;"
            size="small"
            type="success"
            @click="submitUpload"
          >上传到服务器</el-button>
        </el-row>
      </div>
    </el-upload>
  </div>
</template>

<script>
export default {
  data() {
    return {
      fileList: [],
      action: "",
      authData: {}
    };
  },
  methods: {
    submitUpload() {
      this.$refs.upload.submit();
    },
    handleRemove(file, fileList) {
      console.log("rm");
      console.log(file, fileList);
    },
    handlePreview(file) {
      console.log(file);
    },
    async handleBeforeUpload(file) {
      const auth = await this.$axios.get("/api/AliOSS");
      const upload = await this.$axios.post("/api/AliOSS/upload", {
        filename: file.name,
        uploadTime: new Date(),
        status: "uploading"
      });
      console.log(auth);
      console.log(upload);
      this.$set(this.authData, `${file.uid}`, {
        host: auth.data.data.host,
        auth: {
          name: `${file.name}`,
          key: `${auth.data.data.dir}\${filename}`,
          policy: `${auth.data.data.policy}`,
          OSSAccessKeyId: `${auth.data.data.accessid}`,
          success_action_status: 200,
          signature: `${auth.data.data.signature}`
        },
        upload: upload.data.data
      });
    },
    handleUploadFile(param) {
      let config = {
        //添加请求头
        headers: { "Content-Type": "multipart/form-data" },
        //添加上传进度监听事件
        onUploadProgress: e => {
          var completeProgress = (((e.loaded / e.total) * 100) | 0) + "%";
          e.percent = completeProgress;
          this.$refs.upload.handleProgress(e, param.file);
        }
      };
      let formData = new FormData();
      for (let key in this.authData[`${param.file.uid}`].auth) {
        formData.append(key, this.authData[`${param.file.uid}`].auth[key]);
      }
      formData.append("file", param.file);
      this.$axios
        .post(`${this.authData[`${param.file.uid}`].host}`, formData, config)
        .then(res => {
          this.$refs.upload.handleSuccess(res, param.file);
          this.handleUploadSuccess(param.file);
        })
        .catch(error => {
          this.$refs.upload.handleError(error, param.file);
          this.handleUploadError(param.file);
        });
    },
    handleUploadSuccess(file) {
      this.authData[`${file.uid}`].upload.status = "upload finished";
      this.$axios.put("/api/AliOSS/upload", this.authData[`${file.uid}`].upload);
    },
    handleUploadError(file) {
      this.authData[`${file.uid}`].upload.status = "upload error";
      this.$axios.put("/api/AliOSS/upload", this.authData[`${file.uid}`].upload);
    },
    handleUploadProgress() {}
  }
};
</script>

<style scoped>
</style>


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
          >上传</el-button>
          <el-button
            style="margin-left: 10px;"
            size="small"
            type="danger"
            @click="handleUploadStop"
          >暂停</el-button>
          <el-button
            style="margin-left: 10px;"
            size="small"
            type="info"
            @click="handleUploadResume"
          >继续</el-button>
        </el-row>
      </div>
    </el-upload>
  </div>
</template>

<script>
import OSS from 'ali-oss';

export default {
  data() {
    return {
      fileList: [],
      action: "",
      authData: {},
      ossClient: null,
      upload: {}
    };
  },
  methods: {
    async submitUpload() {
      await this.initOssClient()
      this.$refs.upload.submit();
    },
    handleRemove(file, fileList) {
      console.log("rm", file, fileList);
    },
    handlePreview(file) {
      console.log("prev:",file);
    },
    async handleBeforeUpload(file) {
      // 数据库 标记开始上传 
      const upload = await this.$axios.post("/api/AliOSS/upload", {
        filename: file.name,
        uploadTime: new Date(),
        status: "uploading"
      });
      this.$set(this.upload, `${file.uid}`, upload.data.data);
    },
    async handleUploadFile(param) {
      // 单片上传
      // this.singlePartUpload(param.file)

      // 分片上传
      this.multiPartUpload(param.file)
      
      // this.$axios
      //   .post(`${this.authData[`${param.file.uid}`].host}`, formData, config)
      //   .then(res => {
      //     this.$refs.upload.handleSuccess(res, param.file);
      //     this.handleUploadSuccess(param.file);
      //   })
      //   .catch(error => {
      //     this.$refs.upload.handleError(error, param.file);
      //     this.handleUploadError(param.file);
      //   });
    },
    handleUploadSuccess(file) {
      this.upload[`${file.uid}`].status = "upload finished";
      this.$axios.put("/api/AliOSS/upload", this.upload[`${file.uid}`]);
    },
    handleUploadError(file) {
      this.upload[`${file.uid}`].status = "upload error";
      this.$axios.put("/api/AliOSS/upload", this.upload[`${file.uid}`]);
    },
    handleUploadProgress(progress, checkpoint) {
      // this.$refs.upload
      console.log(`${checkpoint.file.name} ---> ${progress * 100}`)
      console.log(checkpoint)
      this.$refs.upload.onProgress(progress * 100, checkpoint.file)
    },
    handleUploadStop() {
      if (this.ossClient) {
        this.ossClient.cancle()
      }
    },
    handleUploadResume() {

    },
    async initOssClient() {
      // 初始化 OSS客户端
      const auth = await this.$axios.get("/api/AliOSS/getAliStsToken");
      this.ossClient = new OSS({
        accessKeyId: auth.data.data.accrssKeyId,
        accessKeySecret: auth.data.data.accessKeySecret,
        bucket: auth.data.data.bucket,
        region: `oss-${auth.data.data.region}`,
        stsToken: auth.data.data.securityToken
      })
    },
    async multiPartUpload(file) {
      console.log(file)
      if (!this.ossClient) {
        await this.initOssClient()
      }
      const partSize = 1024 * 100; // 片大小
      const parallel = 3; // 同时上传
      this.ossClient.multipartUpload(file.name, file, {
        parallel,
        partSize,
        progress: this.handleUploadProgress
      }).then(res => {
        console.log(res)
        this.handleUploadSuccess(file)
      }).catch(res=> {
        console.error(res)
        this.handleUploadError(file)
      })

    },
    async singlePartUpload(file) {
      if (!this.ossClient) {
        await this.initOssClient()
      }
      this.ossClient.put(file.name, file).then(res => {
        console.log(res)
        this.handleUploadSuccess(file)
      }).catch(res => {
        console.error(res)
        this.handleUploadError(file)
      })
    }
  }
};
</script>

<style scoped>
</style>


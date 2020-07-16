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
          <el-button style="margin-left: 10px;" size="small" type="success" @click="submitUpload">上传</el-button>
          <el-button
            style="margin-left: 10px;"
            size="small"
            type="danger"
            @click="handleUploadStop"
          >暂停</el-button>
          <el-button
            style="margin-left: 10px;"
            size="small"
            type="primary"
            @click="handleUploadResume"
          >继续</el-button>
        </el-row>
      </div>
    </el-upload>
  </div>
</template>

<script>
import OSS from "ali-oss";

export default {
  data() {
    return {
      fileList: [],
      action: "",
      authData: {},
      ossClient: null,
      upload: {},
      checkpoints: {},
      multiPartConfig: {
        parallel: 3,
        partSize: 1024 * 512,
        progress: this.handleUploadProgress
      }
    };
  },
  methods: {
    async submitUpload() {
      await this.initOssClient();
      this.$refs.upload.submit();
    },
    handleRemove(file, fileList) {
      console.log("rm", file, fileList);
    },
    handlePreview(file) {
      console.log("prev:", file);
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
      this.multiPartUpload(param.file);

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
    handleUploadError(file, msg) {
      this.upload[`${file.uid}`].status = msg;
      this.$axios.put("/api/AliOSS/upload", this.upload[`${file.uid}`]);
    },
    handleUploadProgress(progress, checkpoint) {
      // this.$refs.upload
      this.$set(this.checkpoints, `${checkpoint.uploadId}`, checkpoint);

      this.$refs.upload.handleProgress(
        {
          percent: progress * 100
        },
        checkpoint.file
      );
    },
    handleUploadStop() {
      if (this.ossClient) {
        this.ossClient.cancel();
      }
    },
    handleUploadResume() {
      this.resumeMultiPartUpload()
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
      });
    },
    async multiPartUpload(file) {
      if (!this.ossClient) {
        await this.initOssClient();
      }
      this.ossClient
        .multipartUpload(file.name, file, {
          parallel: this.multiPartConfig.parallel,
          partSize: this.multiPartConfig.partSize,
          progress: this.multiPartConfig.progress
        })
        .then(res => {
          this.handleUploadSuccess(file, res.name);
        })
        .catch(res => {
          this.handleUploadError(file, res.name);
        });
    },
    async singlePartUpload(file) {
      if (!this.ossClient) {
        await this.initOssClient();
      }
      this.ossClient
        .put(file.name, file)
        .then(res => {
          this.handleUploadSuccess(file, res.name);
        })
        .catch(res => {
          this.handleUploadError(file, res.name);
        });
    },
    async resumeMultiPartUpload() {
      if (!this.ossClient) {
        await this.initOssClient();
      }
      Object.values(this.checkpoints).forEach(checkpoint => {
        const { uploadId, file } = checkpoint;
          this.ossClient.multipartUpload(uploadId, file, {
          parallel: this.multiPartConfig.parallel,
          partSize: this.multiPartConfig.partSize,
          progress: this.multiPartConfig.progress,
          checkpoint: checkpoint
        }).then(res => {
          this.$delete(this.checkpoints, uploadId)
          this.handleUploadSuccess(file, res.name)
        }).catch(res => {
          this.handleUploadError(file, res.name)
        })
      });
    }
  }
};
</script>

<style scoped>
</style>


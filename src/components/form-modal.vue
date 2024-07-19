<template>
  <a-modal
    v-model:open="open"
    :title="title"
    @cancel="handleCancel"
    :footer="null"
    :maskClosable="false"
  >
    <a-form
      ref="formRef"
      :label-col="{ span: 6 }"
      :wrapper-col="{ span: 16 }"
      autocomplete="off"
      :model="formState"
      :rules="rules"
      @finish="onFinish"
      @finishFailed="onFinishFailed"
      style="margin: 10%"
    >
      <a-form-item label="文件别名" name="name">
        <a-input v-model:value="formState.name" />
      </a-form-item>

      <a-form-item v-if="!formState.id" label="文件上传" name="files">
        <a-upload
          v-model:file-list="fileList"
          name="file"
          :headers="headers"
          :customRequest="uploadRequest"
          :before-upload="beforeUploadFj"
          @change="changeFj"
          @remove="removeFj"
          :maxCount="1"
        >
          <a-button> 上传 </a-button>
        </a-upload>
      </a-form-item>

      <div>
        <a-form-item :wrapper-col="{ offset: 10, span: 16 }">
          <a-button type="primary" html-type="submit">提交</a-button>
          <a-button style="margin: 0 10px" @click="handleCancel" type="default"
            >退出</a-button
          >
          <a-button @click="resetForm" type="dashed">重置</a-button>
        </a-form-item>
      </div>
    </a-form>
  </a-modal>
</template>

<script setup>
import { ref, reactive, defineExpose, defineEmits, unref } from "vue";
// import dayjs from "dayjs";
import axios from "axios";
import { message } from "ant-design-vue";
import { Upload } from "ant-design-vue";

const emit = defineEmits(["close"]);
const open = ref(false);

function validateFiles(rule, value, callback) {
  console.log("🚀 ~ validateFiles ~ rule:", rule);
  console.log("🚀 ~ validateFiles ~ value:", value);
  if (fileList.value.length <= 0) {
    callback(new Error("请上传文件!"));
  } else {
    callback();
  }
}
const rules = {
  name: [{ required: true, message: "文件名称!" }],
  files: [{ required: true, validator: validateFiles, trigger: "change" }],
};
const formRef = ref();
const title = ref(undefined);

// const formData = reactive({ name: "" });

// 后端地址
const apiUrl = "http://localhost:3000"; // 替换为你的后端 API 地址

// const actionUrl = `${apiUrl}/uploads`;
const formState = reactive({
  name: "",
  id: undefined,
  files: undefined,
});

const fileState = reactive({
  id: undefined,
  fileName: undefined,
  downloadUrl: undefined,
});

const fileList = ref([]);
// const uploadUrl = "http://localhost:3000/upload-file"; // 后端文件上传接口
const headers = {
  "Content-Type": "multipart/form-data",
};

const onShow = (id = undefined) => {
  open.value = true;
  title.value = `${id ? "编辑" : "新增"}文件`;
  if (id) {
    formState.id = id;
    getInfo(id);
  }
};

function resetToUndefined(obj) {
  Object.keys(obj).forEach((key) => {
    obj[key] = undefined;
  });
}

const handleCancel = () => {
  resetForm();
  open.value = false;
};

const getInfo = (id) => {
  // 根据id 获取表单数据
  const url = `${apiUrl}/getTableInfoById/${id}`;
  axios.get(url).then((res) => {
    let { code, row } = res.data;
    if (code == 200) {
      formState.name = row.name;
      formState.files = JSON.parse(row.files);
    }
  });
};

const onFinish = (values) => {
  console.log("Success:", values);
  //updateTableData
  const url = `${apiUrl}/${formState.id ? "updateTableData" : "postFormData"}`; // get 查看数据库数据

  formState.files = Object.assign({}, fileState);
  if (formState.id) {
    axios.put(url, unref(formState)).then((res) => {
      let { code, msg } = res.data;
      if (code == 200) {
        message.success(msg);
        emit("close");
        resetForm();
        handleCancel();
      }
    });
  } else {
    axios.post(url, formState).then((res) => {
      let { code, msg } = res.data;
      if (code == 200) {
        message.success(msg);
        emit("close");
        resetForm();
        handleCancel();
      }
    });
  }
};
const onFinishFailed = (errorInfo) => {
  console.log("Failed:", errorInfo);
};

const resetForm = () => {
  formRef.value.resetFields();
  resetToUndefined(fileState);

  fileList.value.pop();
};

function formatFileSize(sizeInBytes) {
  const sizeInGB = sizeInBytes / (1024 * 1024 * 1024);
  return `${sizeInGB.toFixed(2)}gb`;
}

const uploadRequest = (event) => {
  const formData = new FormData();
  formData.append("file", event.file);
  formData.append("name", event.file.name); // 添加编码后的文件名到 FormData

  const url = `${apiUrl}/uploads`;
  axios
    .post(url, formData, {
      headers: {
        "Content-Type": "multipart/form-data; charset=utf-8", // 设置请求头为 multipart/form-data，并指定字符集为 UTF-8
      },
    })
    .then((res) => {
      let { code } = res.data;
      if (code == 200) {
        event.onSuccess(res, event.file); //上传成功监听事件
        fileState.id = res.data.data.id;
        fileState.fileName = res.data.data.name;
        fileState.downloadUrl = res.data.data.downloadUrl;
      }
    })
    .catch((err) => {
      event.onError(err, err, event.file);
    });
};

const changeFj = (info) => {
  if (info.file.status !== "uploading") {
    message.success(`${info.file.name} 上传中!`);
  }
  if (info.file.status === "done") {
    fileList.value = info.fileList;
    message.success(`${info.file.name} 上传成功!`);
  } else if (info.file.status === "error") {
    message.success(`${info.file.name} 上传失败!`);
    fileList.value.pop();
  }
};

const removeFj = (file) => {
  console.log("🚀 ~ removeFj ~ file:", file);
  fileList.value.pop();
  resetToUndefined(fileState);
};

const beforeUploadFj = (file) => {
  console.log("🚀 ~ beforeUploadFj ~ file:", file);
  const isAcceptType = [
    "application/msword",
    "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "application/vnd.ms-excel",
    "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "application/vnd.ms-powerpoint",
    "application/pdf",
    "application/vnd.openxmlformats-officedocument.presentationml.presentation",
    "application/x-zip-compressed",
    "image/tiff",
    "application/vnd.android.package-archive",
  ];
  if (!isAcceptType.includes(file.type)) {
    message.error("仅支持doc、xls、ppt、pdf、docx、xlsx、pptx、zip、tiff、apk");
    return false || Upload.LIST_IGNORE;
  }
  const isLt10M = file.size / 1024 / 1024 < 500;
  fileState.fileSize = formatFileSize(file.size);
  if (!isLt10M) {
    message.error("上传文件大小不能超过 500MB!");
  }
  return isLt10M || Upload.LIST_IGNORE;
};

defineExpose({ onShow });
</script>
